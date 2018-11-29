---
title: Serializable Transactions
summary:
toc: true
---

대부분의 데이터베이스와는 다르게, CockroachDB v2.1은 항상 `SERIALIZABLE` 격리를 사용합니다. 이는 SQL 표준에 이해 정의된 네 개의 [트랜잭션 격리 수준] (https://en.wikipedia.org/wiki/Isolation_(database_systems)) 중에서 가장 강력한 격리이고, 나중에 개발된 `SNAPSHOT` 격리 수준보다 강력합니다.
`SERIALIZABLE` 격리는 트랜잭션이 병렬로 실행될 수 있다고 하더라도 동시성없이 한 번에 하나씩 실행했을 때와 같은 최종 결과를 보장합니다. "예외"을 방지함으로써 데이터 정확성을 보장합니다.

 이 튜토리얼에서는 데이터 정확성을 위한 `SERIALIZABLE` 격리의 중요성을 보여주는 가상의 시나리오를 살펴보겠습니다.

1. 시나리오와 스키마를 검토하며 시작합니다.
2. 그런 다음 스큐 쓰기 예외와 그 영향을 관찰하면서 약한 격리 수준인 `READ COMMITTED` 중 하나에서 시나리오를 실행합니다. CockroachDB는 항상 `SERIALIZABLE` 격리를 사용하기 때문에 기본값이 `READ COMMITTED`인 포스트그레스에서 이 부분을 실행하게 됩니다. 

3. `SERIALIZABLE` 격리 시나리오를 실행하여 그것이 정확성을 보장하는 방법을 관찰하면서 끝낼 것입니다. 이 부분에는 CockroachDB를 사용합니다.


{{site.data.alerts.callout_info}}
트랜잭션 격리 및 스큐 쓰기 예외에 대한 자세한 내용은 [실제 트랜잭션 직렬화](https://www.cockroachlabs.com/blog/acid-rain/) 및 [스큐 쓰기 모양](https://www.cockroachlabs.com/blog/what-write-skew-looks-like/) 블로그 게시물을 보십시오.
{{site.data.alerts.end}}

## 개요

### 시나리오

- 병원에는 의사들의 대기 교대를 관리하는 어플리케이션이 있다.
- 병원은 적어도 한 명의 의사가 언제든지 대기중이어야 한다는 규칙을 가지고 있습니다.
- 두 명의 의사가 특정 교대를 요구하고 있으며, 둘은 거의 같은 시간에 교대 휴가를 요청합니다.
- 포스트그레스에서 디폴트 `READ COMMITTED` 격리 수준을 사용하면 스큐 쓰기 예외는 두 의사가 휴가를 성공적으로 예약하고 병원은 특정 교대시간에 의사를 보유하지 못하게 합니다.
- CockroachDB에서 기본 `SERIALIZABLE` 격리 수준을 사용하면, 스큐 쓰기가 방지되고, 의사 한 명은 휴가가 허용되고 다른 한 명은 대기중이어서, 생명을 구할 수 있습니다.


#### 스큐 쓰기

스큐 쓰기가 발생하면 트랜잭션은 무언가를 읽고, 읽은 값을 기반으로 결정을 내리고, 결정을 데이터베이스에 기록합니다.
그러나 기록이 작성 될 때까지, 결정의 전제는 더 이상 사실이 아닙니다. `SERIALIZABLE` 및`REPEATABLE READ` 격리의 일부 구현만이 예외를 방지합니다.

### 스키마

<img src="{{ 'images/v2.1/serializable_schema.png' | relative_url }}" alt="직렬화 가능 트랜잭션 튜토리얼을 위한 스키마" style="max-width:100%" />

## 포스트그레스 시나리오

### 1단계. 포스트그레스 시작하기

1. 아직 설치하지 않았다면 포스트그레스를 로컬에 설치하십시오. Mac에서는 [Homebrew](https://brew.sh/)를 사용할 수 있습니다.


    {% include copy-clipboard.html %}
    ~~~ shell
    $ brew install postgres
    ~~~

2. [포스트그레스 시작하기](https://www.postgresql.org/docs/10/static/server-start.html):

    {% include copy-clipboard.html %}
    ~~~ shell
    $ postgres -D /usr/local/var/postgres &
    ~~~

### 2단계. 스키마 생성하기

1. 포스트그레스에 대한 SQL 연결을 엽니다:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ psql
    ~~~

2. `doctors` 테이블을 생성합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE doctors (
        id INT PRIMARY KEY,
        name TEXT
    );
    ~~~

3. `schedules` 테이블을 생성합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE schedules (
        day DATE,
        doctor_id INT REFERENCES doctors (id),
        on_call BOOL,
        PRIMARY KEY (day, doctor_id)
    );
    ~~~

### 3단계. 데이터 삽입하기

1. `doctors` 테이블에 의사 두명을 추가합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO doctors VALUES
        (1, 'Abe'),
        (2, 'Betty');
    ~~~

2. `schedules`테이블에 일주일 분량의 데이터를 삽입합니다:


    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO schedules VALUES
        ('2018-10-01', 1, true),
        ('2018-10-01', 2, true),
        ('2018-10-02', 1, true),
        ('2018-10-02', 2, true),
        ('2018-10-03', 1, true),
        ('2018-10-03', 2, true),
        ('2018-10-04', 1, true),
        ('2018-10-04', 2, true),
        ('2018-10-05', 1, true),
        ('2018-10-05', 2, true),
        ('2018-10-06', 1, true),
        ('2018-10-06', 2, true),
        ('2018-10-07', 1, true),
        ('2018-10-07', 2, true);
    ~~~

3. 일주일에 적어도 한 명의 의사가 대기중인지 확인합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SELECT day, count(*) AS doctors_on_call FROM schedules
      WHERE on_call = true
      GROUP BY day
      ORDER BY day;
    ~~~

    ~~~
        day     | doctors_on_call
    ------------+-----------------
     2018-10-01 |               2
     2018-10-02 |               2
     2018-10-03 |               2
     2018-10-04 |               2
     2018-10-05 |               2
     2018-10-06 |               2
     2018-10-07 |               2
    (7 rows)
    ~~~

### 4단계. 의사 1이 휴가를 요청

의사 1 Abe가 병원의 일정 관리 어플리케이션을 사용하여 10/5/18 휴가를 요청하기 시작합니다.

1. 응용 프로그램이 트랜잭션을 시작합니다: 

    {% include copy-clipboard.html %}
    ~~~ sql
    > BEGIN;
    ~~~

2. 어플리케이션은 요청한 날짜에 적어도 한 명의 다른 의사가 대기중인지 확인합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SELECT count(*) FROM schedules
      WHERE on_call = true
      AND day = '2018-10-05'
      AND doctor_id != 1;
    ~~~

    ~~~
     count
    -------
         1
    (1 row)
    ~~~

### 5단계. 의사 2가 휴가를 요청

같은 시간에 의사 2, Betty는 병원의 일정 관리 어플리케이션을 사용하여 같은 날 휴가를 요청하기 시작합니다:

1. 새 터미널에서 두 번째 SQL 세션을 시작합니다:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ psql
    ~~~

2. 응용 프로그램이 트랜잭션을 시작합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > BEGIN;
    ~~~

3. 어플리케이션은 요청한 날짜에 적어도 한 명의 다른 의사가 대기중인지 확인합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SELECT count(*) FROM schedules
      WHERE on_call = true
      AND day = '2018-10-05'
      AND doctor_id != 2;
    ~~~

    ~~~
     count
    -------
         1
    (1 row)
    ~~~

### 6단계. 두 의사에게 휴가를 잘못 예약

1. 의사 1의 터미널에서, 이전에 다른 의사가 10/5/18에 대기중인 것이 확인되었으므로, 어플리케이션은 의사 1의 일정을 업데이트하려고 시도합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > UPDATE schedules SET on_call = false
      WHERE day = '2018-10-05'
      AND doctor_id = 1;
    ~~~

2. 의사 2의 터미널에서, 이전에 같은 것이 확인되었으므로, 어플리케이션은 의사 2의 일정을 업데이트하려고 시도합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > UPDATE schedules SET on_call = false
      WHERE day = '2018-10-05'
      AND doctor_id = 2;
    ~~~

3. 의사 1의 터미널에서 이전의 검사 (SELECT 쿼리)가 더 이상 사실이 아니더라도 어플리케이션이 트랜잭션을 커밋합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > COMMIT;
    ~~~

4. 의사 2의 터미널에서 이전의 검사 (SELECT 쿼리)가 더 이상 사실이 아니더라도 어플리케이션이 트랜잭션을 커밋합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > COMMIT;
    ~~~

### 7단계. 데이터 정확성 검사

무슨 일이 일어났을까요? 각 트랜잭션은 트랜잭션이 끝나기 전에 올바르지 않은 값을 읽음으로써 시작되었습니다. 그 사실에도 불구하고, 각각의 트랜잭션은 커밋될 수 있었습니다. 이는 스큐 쓰기로 알려져 있으며 결과적으로 0명의 의사가 10/5/18에 대기 예정입니다.

이를 확인하려면 터미널에서 다음을 실행하십시오: 

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM schedules WHERE day = '2018-10-05';
~~~

~~~
    day     | doctor_id | on_call
------------+-----------+---------
 2018-10-05 |         1 | f
 2018-10-05 |         2 | f
(2 rows)
~~~

다시 한번 말하면, 이 예외는 Postgres의 기본 격리 수준인 `READ COMMITTED` 의 결과이지만, `SERIALIZABLE`과 `REPEATABLE READ`의 일부 구현을 제외하고는 격리 수준에서 발생합니다: 

{% include copy-clipboard.html %}
~~~ sql
> SHOW TRANSACTION_ISOLATION;
~~~

~~~
 transaction_isolation
-----------------------
 read committed
(1 row)
~~~

### 8단계. 포스그레스 중지하기

`\q`를 사용하여 각 SQL 쉘을 종료 한 다음 포스그레스 서버를 중지하십시오:


{% include copy-clipboard.html %}
~~~ shell
$ pkill -9 postgres
~~~

## CockroachDB의 시나리오

CockroachDB에서 시나리오를 반복하면 CockroachDB의 `SERIALIZABLE` 트랜잭션 격리로 인해 예외가 방지된다는 것을 알 수 있습니다.


### 1단계. CockroachDB 시작하기

1. 아직 설치하지 않았다면 [CockroachDB 설치](install-cockroachdb.html)를 로컬에 설치하십시오.


2. 비보안 모드에서 단일 노드 CockroachDB 클러스터를 시작하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --store=serializable-demo \
    --listen-addr=localhost \
    --background
    ~~~

### 2단계. 스키마 생성하기

1. `root` 사용자로서, [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)를 엽니다 :

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure --host=localhost
    ~~~

2. `doctors` 테이블을 생성합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE doctors (
        id INT PRIMARY KEY,
        name TEXT
    );
    ~~~

3. `schedules` 테이블을 생성합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE schedules (
        day DATE,
        doctor_id INT REFERENCES doctors (id),
        on_call BOOL,
        PRIMARY KEY (day, doctor_id)
    );
    ~~~

### 3단계. 데이터 삽입하기

1. 의사 두 명을 `doctors`표에 추가하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO doctors VALUES
        (1, 'Abe'),
        (2, 'Betty');
    ~~~

2. `schedules`테이블에 일주일 분량의 데이터를 삽입합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO schedules VALUES
        ('2018-10-01', 1, true),
        ('2018-10-01', 2, true),
        ('2018-10-02', 1, true),
        ('2018-10-02', 2, true),
        ('2018-10-03', 1, true),
        ('2018-10-03', 2, true),
        ('2018-10-04', 1, true),
        ('2018-10-04', 2, true),
        ('2018-10-05', 1, true),
        ('2018-10-05', 2, true),
        ('2018-10-06', 1, true),
        ('2018-10-06', 2, true),
        ('2018-10-07', 1, true),
        ('2018-10-07', 2, true);
    ~~~

3. `schedules`테이블에 일주일 분량의 데이터를 삽입합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SELECT day, count(*) AS on_call FROM schedules
      WHERE on_call = true
      GROUP BY day
      ORDER BY day;  
    ~~~

    ~~~
                 day            | on_call
    +---------------------------+---------+
      2018-10-01 00:00:00+00:00 |       2
      2018-10-02 00:00:00+00:00 |       2
      2018-10-03 00:00:00+00:00 |       2
      2018-10-04 00:00:00+00:00 |       2
      2018-10-05 00:00:00+00:00 |       2
      2018-10-06 00:00:00+00:00 |       2
      2018-10-07 00:00:00+00:00 |       2
    (7 rows)
    ~~~

### 4단계. 의사 1이 휴가를 요청

의사 1 Abe가 병원의 일정 관리 어플리케이션을 사용하여 10/5/18 휴가를 요청하기 시작합니다.

1. 응용 프로그램이 트랜잭션을 시작합니다: 

    {% include copy-clipboard.html %}
    ~~~ sql
    > BEGIN;
    ~~~

2. 어플리케이션은 요청한 날짜에 적어도 한 명의 다른 의사가 대기중인지 확인합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SELECT count(*) FROM schedules
      WHERE on_call = true
      AND day = '2018-10-05'
      AND doctor_id != 1;
    ~~~

    Enter 키를 두 번 눌러 서버가 결과를 반환하도록 합니다:

    ~~~
      count
    +-------+
          1
    (1 row)    
    ~~~

### 5단계. 의사 2가 휴가를 요청

같은 시간에 의사 2, Betty는 병원의 일정 관리 어플리케이션을 사용하여 같은 날 휴가를 요청하기 시작합니다:

1. 새 터미널에서 두 번째 SQL 세션을 시작합니다:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure --host=localhost
    ~~~

2. 응용 프로그램이 트랜잭션을 시작합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > BEGIN;
    ~~~

3. 어플리케이션은 요청한 날짜에 적어도 한 명의 다른 의사가 대기중인지 확인합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SELECT count(*) FROM schedules
      WHERE on_call = true
      AND day = '2018-10-05'
      AND doctor_id != 2;
    ~~~

    Enter 키를 두 번 눌러 서버가 결과를 반환하도록 합니다:

    ~~~
      count
    +-------+
          1
    (1 row)    
    ~~~

### 6단계. 한 의사에게만 휴가를 예약

1. 의사 1의 터미널에서, 이전에 다른 의사가 10/5/18에 대기중인 것이 확인되었으므로, 어플리케이션은 의사 1의 일정을 업데이트하려고 시도합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > UPDATE schedules SET on_call = false
      WHERE day = '2018-10-05'
      AND doctor_id = 1;
    ~~~

2. 의사 2의 터미널에서, 이전에 같은 것이 확인되었으므로, 어플리케이션은 의사 2의 일정을 업데이트하려고 시도합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > UPDATE schedules SET on_call = false
      WHERE day = '2018-10-05'
      AND doctor_id = 2;
    ~~~

3. 의사 1의 터미널에서 어플리케이션이 트랜잭션을 커밋하려고 시도합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > COMMIT;
    ~~~

    CockroachDB는`SERIALIZABLE` 격리를 사용하기 때문에, 데이터베이스는 병행 트랜잭션 때문에 이전 검사 (`SELECT` 쿼리)가 더 이상 참이 아님을 감지합니다.


따라서 트랜잭션이 커밋되지 않도록 하여 트랜잭션을 다시 시도해야 함을 나타내는 "다시 시도 가능한 오류"를 반환합니다.


    ~~~
    pq: restart transaction: HandledRetryableTxnError: TransactionRetryError: retry txn (RETRY_SERIALIZABLE): "sql txn" id=57dd0454 key=/Table/53/1/17809/1/0 rw=true pri=0.00710012 iso=SERIALIZABLE stat=PENDING epo=0 ts=1539116499.676097000,2 orig=1539115078.961557000,0 max=1539115078.961557000,0 wto=false rop=false seq=4
    ~~~

    {{site.data.alerts.callout_success}}
    For this kind of error, CockroachDB recommends a [client-side transaction retry loop](transactions.html#client-side-transaction-retries) that would transparently observe that the one doctor can't take time off because the other doctor already succeeded in asking for it. You can find generic transaction retry functions for various languages in our [Build an App](build-an-app-with-cockroachdb.html) tutorials.
    {{site.data.alerts.end}}

4. 의사 2의 터미널에서 어플리케이션이이 트랜잭션을 커밋하려고 시도합니다:

    {% include copy-clipboard.html %}
    ~~~ sql
    > COMMIT;
    ~~~

    의사 1의 트랜잭션이 실패했기 때문에 의사 2의 트랜잭션은 데이터의 정확성 문제를 일으키지 않고 커밋할 수 있습니다.


### 7단계. 데이터 정확성 검사

어느 터미널에서든 10/5/18 동안 한 명의 의사가 여전히 대기중인지 확인하십시오:


{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM schedules WHERE day = '2018-10-05';
~~~

~~~
             day            | doctor_id | on_call
+---------------------------+-----------+---------+
  2018-10-05 00:00:00+00:00 |         1 |  true
  2018-10-05 00:00:00+00:00 |         2 |  false
(2 rows)
~~~

다시 한 번, 스큐 쓰기 예외는 `SERIALIZABLE` 격리 수준을 사용하는 CockroachDB에 의해 방지되었습니다.


{% include copy-clipboard.html %}
~~~ sql
> SHOW TRANSACTION_ISOLATION;
~~~

~~~
  transaction_isolation
+-----------------------+
  serializable
(1 row)
~~~

### Step 8. CockroachDB 중지하기

테스트 클러스터가 끝나면 `\q`를 사용하여 각 SQL 쉘을 종료 한 다음 노드를 중지하십시오.


{% include copy-clipboard.html %}
~~~ shell
$ cockroach quit --insecure --host=localhost
~~~

클러스터를 다시 시작하지 않으려는 경우, 노드의 데이터 저장소를 제거하고 싶을 것입니다.


{% include copy-clipboard.html %}
~~~ shell
$ rm -rf serializable-demo
~~~

## 더 알아보기

CockroachDB의 다른 주요 이점 및 특징에 대해 알아보십시오: 

{% include {{ page.version.version }}/misc/explore-benefits-see-also.md %}

또한 일반적으로 CockroachDB에서 트랜잭션이 어떻게 작동하는지에 대해 더 자세히 알고 싶을 수도 있습니다:

- [트랜잭션 개요](transactions.html)
- [실제 트랜잭션은 직렬화 가능](https://www.cockroachlabs.com/blog/acid-rain/)
- [Write Skew의 모양](https://www.cockroachlabs.com/blog/what-write-skew-looks-like/)
