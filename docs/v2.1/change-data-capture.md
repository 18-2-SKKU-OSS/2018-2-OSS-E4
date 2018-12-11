---
title: Change Data Capture (CDC)
summary: Change data capture (CDC) provides efficient, distributed, row-level change subscriptions.
toc: true
---

<span class="version-tag">v2.1의 새로운 기능:</span> Change data capture (CDC)는 리포트, 캐싱 또는 전문 인덱스(full-texted index)와 같은 다운스트림 프로세싱(후처리)을 위해 효율적이고 분산된 행-수준 변경 피드를 Apache Kafka에 제공합니다.

{{site.data.alerts.callout_danger}}
**이 기능은 현재 개발 중이며** 목표로 하는 사용 사례에서만 작동합니다. 로드맵에 대한 피드백이 있는 경우 [Github 이슈](file-an-issue.html)를 보내주십시오.
{{site.data.alerts.end}}

{{site.data.alerts.callout_info}}
CDC는 [기업(enterprise) 전용](enterprise-licensing.html)입니다. 향후 출시에서는 핵심 버전이 제공될 예정입니다.
{{site.data.alerts.end}}

## Change data capture(CDC)란?

CockroachDB는 뛰어난 기록 시스템이지만 다른 시스템과도 공존해야 합니다. 예를 들어, 전문 인덱스(full-texted index), 분석 엔진 또는 빅데이터 파이프라인에 데이터를 미러링 해야 하는 경우의 필요성이 있습니다.

CDC의 핵심 기능은 [변경 피드(changefeed)](create-changefeed.html)입니다. 변경 피드는 "관찰 대상 행(watched rows)"라고 부르는 표의 화이트리스트를 대상으로 합니다. 관찰 대상 행에 대한 모든 변경 사항은 구성 가능한 형식(JSON 또는 Avro)의 기록으로 구성 가능한 싱크([Kafka](https://kafka.apache.org/))로 방출됩니다.

## 순서 보장

- 대부분의 경우 행의 각 버전은 한 번만 방출됩니다. 그러나, 간혹 발생하는 일부 조건(예: 노드 오류, 네트워크 파티션)으로 인해 반복될 수 있습니다. 이것은 변경 에게 적어도 한번의 전달 보증을 제공합니다.

- 일부 타임스탬프를 사용하여 행을 방출하면 이전에 볼 수 없었던 해당 행의 버전은 더 낮은 타임스탬프를 사용하여 방출되지 않습니다. 즉, 이전 타임스탬프에서는 해당 행에 대해 _new_ 변경 사항을 볼 수 없습니다.

    예를 들어, 다음을 실행한 경우:

    ~~~ sql
    > CREATE TABLE foo (id INT PRIMARY KEY DEFAULT unique_rowid(), name STRING);
    > CREATE CHANGEFEED FOR TABLE foo INTO 'kafka://localhost:9092' WITH UPDATED;
    > INSERT INTO foo VALUES (1, 'Carl');
    > UPDATE foo SET name = 'Petee' WHERE id = 1;
    ~~~

    변경 피드가 다음을 방출할 것으로 예상할 것입니다:

    ~~~ shell
    [1]	{"__crdb__": {"updated": <timestamp 1>}, "id": 1, "name": "Carl"}
    [1]	{"__crdb__": {"updated": <timestamp 2>}, "id": 1, "name": "Petee"}
    ~~~

    또한 변경 피드가 이미 보았던 이전 값과 중복되는 오더를 생성할 수도 있다고 예상할 것입니다:

    ~~~ shell
    [1]	{"__crdb__": {"updated": <timestamp 1>}, "id": 1, "name": "Carl"}
    [1]	{"__crdb__": {"updated": <timestamp 2>}, "id": 1, "name": "Petee"}
    [1]	{"__crdb__": {"updated": <timestamp 1>}, "id": 1, "name": "Carl"}
    ~~~

    그러나 다음과 같은 결과는 **절대로** 볼 수 없습니다(예: 이전에 보지 못했던 순서가 잘못된 행):

    ~~~ shell
    [1]	{"__crdb__": {"updated": <timestamp 2>}, "id": 1, "name": "Petee"}
    [1]	{"__crdb__": {"updated": <timestamp 1>}, "id": 1, "name": "Carl"}
    ~~~

- 동일한 처리에서 행이 두 번 이상 수정되면 마지막 변경사항만 방출됩니다.

- 행은 [기본 키](primary-key.html)에 의해 Kafka 파티션 사이에 분할됩니다.

- `UPDATED` 옵션은 방출된 각 행에 "업데이트된" 타임스탬프를 추가합니다. 또한 `RESOLVED` 옵션을 사용하여 각 Kafka 파티션에 주기적으로 "해결된" 타임스탬프 메시지를 내보낼 수도 있습니다. **해결된 타임스탬프**는 낮은 업데이트 타임스탬프의 (이전에 보이지 않았던) 행이 해당 파티션에서 방출되지 않는다는 것을 보장합니다.

    예를 들어:

    ~~~ shell
    {"__crdb__": {"updated": "1532377312562986715.0000000000"}, "id": 1, "name": "Petee H"}
    {"__crdb__": {"updated": "1532377306108205142.0000000000"}, "id": 2, "name": "Carl"}
    {"__crdb__": {"updated": "1532377358501715562.0000000000"}, "id": 3, "name": "Ernie"}
    {"__crdb__":{"resolved":"1532379887442299001.0000000000"}}
    {"__crdb__":{"resolved":"1532379888444290910.0000000000"}}
    {"__crdb__":{"resolved":"1532379889448662988.0000000000"}}
    ...
    {"__crdb__":{"resolved":"1532379922512859361.0000000000"}}
    {"__crdb__": {"updated": "1532379923319195777.0000000000"}, "id": 4, "name": "Lucky"}
    ~~~

- 중복을 제거하면 각각의 행이 업데이트된 처리과정과 동일한 순서로 방출됩니다. 그러나 이는 두 개의 다른 행, 특히 동일한 테이블의 두 행에 대한 업데이트에는 해당되지 않습니다. 모든 Kafka 파티션에서 해결된 타임스탬프 알림을 사용하여 타임스탬프 클로저 사이의 기록을 버퍼링하여 강력한 순서 지정 및 전역 일관성 보장을 제공할 수 있습니다.

    CockroachDB는 클러스터의 모든 부분에 영향을 줄 수 있는 처리를 지원하기 때문에, 처리 로그를 독립적인 변경 피드로 수평적으로 분할할 수 없습니다.

## 열(column) 다시 채우기로 스키마 변경

열 다시 채우기(예: 기본값이 있는 열 추가, 계산된 열 추가, `NOT NULL` 열 추가, 열 삭제)를 사용한 스키마 변경사항이 관찰 대상 행에 적용되면 다시 채우기 과정 중에 일부 중복을 방출합니다. 완료되면, CockroachDB는 새로운 스키마를 사용하여 모든 관찰 대상 행들을 출력합니다.

예를 들어, [아래의 예제]에서 생성된 변경 피드로 시작하십시오:

~~~ shell
[1]	{"id": 1, "name": "Petee H"}
[2]	{"id": 2, "name": "Carl"}
[3]	{"id": 3, "name": "Ernie"}
~~~

관찰 대상 테이블에 열을 추가합니다:

{% include copy-clipboard.html %}
~~~ sql
> ALTER TABLE office_dogs ADD COLUMN likes_treats BOOL DEFAULT TRUE;
~~~

변경 피드는 새 스키마를 사용하여 레코드를 출력하기 전에 중복 레코드 1, 2 및 3을 방출합니다:

~~~ shell
[1]	{"id": 1, "name": "Petee H"}
[2]	{"id": 2, "name": "Carl"}
[3]	{"id": 3, "name": "Ernie"}
[1]	{"id": 1, "name": "Petee H"}  # Duplicate
[2]	{"id": 2, "name": "Carl"}     # Duplicate
[3]	{"id": 3, "name": "Ernie"}    # Duplicate
[1]	{"id": 1, "likes_treats": true, "name": "Petee H"}
[2]	{"id": 2, "likes_treats": true, "name": "Carl"}
[3]	{"id": 3, "likes_treats": true, "name": "Ernie"}
~~~

## 변경 피드 구성

### 생성

변경 피드를 생성하려면:

{% include copy-clipboard.html %}
~~~ sql
> CREATE CHANGEFEED FOR TABLE name INTO 'kafka://host:port';
~~~

자세한 내용은 [`CREATE CHANGEFEED`](create-changefeed.html)를 참조하십시오.

### 중지

변경 피드를 중지하려면:

{% include copy-clipboard.html %}
~~~ sql
> PAUSE JOB job_id;
~~~

자세한 내용은 [`PAUSE JOB`](pause-job.html)를 참조하십시오.

### 재개

중지된 변경 피드를 재개하려면:

{% include copy-clipboard.html %}
~~~ sql
> RESUME JOB job_id;
~~~

자세한 내용은 [`RESUME JOB`](resume-job.html)를 참조하십시오.

### 취소

변경 피드를 취소하려면:

{% include copy-clipboard.html %}
~~~ sql
> CANCEL JOB job_id;
~~~

자세한 내용은 [`CANCEL JOB`](cancel-job.html)를 참조하십시오.

## 변경 피드 모니터링

변경 피드 진행 상황은 변경 피드가 진행됨에 따라 개선되는 최고 수준의 타임스탬프로 표시됩니다. 이는 타임스탬프 이전 또는 타임스탬프에서의 모든 변경 사항이 방출되었음을 보장하는 것입니다. 변경 피드를 모니터링 할 수 있습니다:

- 관리 UI의 [작업 페이지](admin-ui-jobs-page.html)에서 최고 수준의 타임스탬프 위로 마우스를 올려 [시스템 시간](as-of-system-time.html)을 확인하십시오.
- `crdb_internal.jobs`를 사용:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SELECT * FROM crdb_internal.jobs WHERE job_id = <job_id>;
    ~~~
    ~~~
            job_id       |  job_type  |                              description                               | ... |      high_water_timestamp      | error | coordinator_id
    +--------------------+------------+------------------------------------------------------------------------+ ... +--------------------------------+-------+----------------+
      383870400694353921 | CHANGEFEED | CREATE CHANGEFEED FOR TABLE office_dogs2 INTO 'kafka://localhost:9092' | ... | 1537279405671006870.0000000000 |       |              1
    (1 row)
    ~~~

{{site.data.alerts.callout_info}}
최고 수준의 타임스탬프를 사용하여 [다른 종료된 곳에서 새로운 변경 피드를 시작](create-changefeed.html#start-a-new-changefeed-where-another-ended)할 수 있습니다.
{{site.data.alerts.end}}

## 사용 

### Kafka에 연결된 changefeed 생성

이 예시에서는 Kafka 싱크에 연결된 단일 노드 클러스터에 대한 변경 피드를 설정합니다.

1. 아직 라이센스를 가지고 있지 않다면, [평가판 엔터프라이즈 라이센스를 요청](enterprise-licensing.html)하십시오.

2. 터미널 창에서 `cockroach`를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start --insecure --listen-addr=localhost --background
    ~~~

3. [Confluent Open Source 플랫폼](https://www.confluent.io/download/)(Kafka 포함)을 다운로드하고 압축을 푸십시오.

4. 압축을 푼 `confluent-<version>` 디렉토리로 이동하여 Confluent를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./bin/confluent start
    ~~~

    `zookeeper`와 `kafka`만 필요합니다. Confluent 문제를 해결하려면 [해당 문서](https://docs.confluent.io/current/installation/installing_cp.html#zip-and-tar-archives)를 참조하십시오.

5. Kafka 항목을 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./bin/kafka-topics \
    --create \
    --zookeeper localhost:2181 \
    --replication-factor 1 \
    --partitions 1 \
    --topic office_dogs
    ~~~

    {{site.data.alerts.callout_info}}
    필요한 복제 및 파티션의 개수만큼 Kafka 항목을 생성해야 합니다. [항목을 수동으로 생성](https://kafka.apache.org/documentation/#basic_ops_add_topic)하거나 [Kafka 중개자를 구성](https://kafka.apache.org/documentation/#topicconfigs)하여 기본 파티션 개수 및 복제 요소로 항목을 자동으로 생성할 수 있습니다.
    {{site.data.alerts.end}}

6. `root` 사용자로 [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)를 여십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure
    ~~~

7. 이메일을 통해 받은 단체명과 [엔터프라이즈 라이센스](enterprise-licensing.html) 키를 설정하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    > SET CLUSTER SETTING cluster.organization = '<organization name>';
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    > SET CLUSTER SETTING enterprise.license = '<secret>';
    ~~~

8. `cdc_demo`라는 데이터베이스를 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE cdc_demo;
    ~~~

9. 8.의 데이터베이스를 기본값으로 설정하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SET DATABASE = cdc_demo;
    ~~~

10. 테이블을 생성하고 데이터를 추가하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE office_dogs (
         id INT PRIMARY KEY,
         name STRING);
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO office_dogs VALUES
       (1, 'Petee'),
       (2, 'Carl');
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > UPDATE office_dogs SET name = 'Petee H' WHERE id = 1;
    ~~~

11. 변경 피드를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE CHANGEFEED FOR TABLE office_dogs INTO 'kafka://localhost:9092';
    ~~~
    ~~~

            job_id       
    +--------------------+
      360645287206223873
    (1 row)
    ~~~

    이것은 백그라운드에서 변경 피드를 시작하고 `job_id`를 반환합니다. 변경 피드는 Kafka에 쓰여집니다.

12. 새 터미널에서, 압축을 푼 `confluent-<version>` 디렉토리로 이동하여 Kafka 항목을 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./bin/kafka-console-consumer \
    --bootstrap-server=localhost:9092 \
    --property print.key=true \
    --from-beginning \
    --topic=office_dogs
    ~~~

    ~~~ shell
    [1]	{"id": 1, "name": "Petee H"}
    [2]	{"id": 2, "name": "Carl"}
    ~~~

    초기 스캔은 변경 피드가 시작된 시점부터 테이블의 상태를 표시합니다(따라서 `"Petee"`의 초기값은 생략됨).

13. SQL 클라이언트로 돌아가서 더 많은 데이터를 삽입하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO office_dogs VALUES (3, 'Ernie');
    ~~~

14. Kafka 항목을 관찰하고 있는 터미널로 돌아가면 다음과 같은 결과가 나타납니다:

    ~~~ shell
    [3]	{"id": 3, "name": "Ernie"}
    ~~~

15. 완료되면 SQL 쉘을 종료하십시오(`\q`).

16. `cockroach`를 멈추려면 다음을 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach quit --insecure
    ~~~

17. Kafka를 멈추려면 압축을 푼 `confluent-<version>` 디렉토리로 이동하여 Confluence를 중단하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./bin/confluent stop
    ~~~

### Avro로 변경 피드 생성

이 예시에서는 Kafka 싱크에 연결되어 [Avro](https://avro.apache.org/docs/1.8.2/spec.html) 레코드를 방출하는 단일 노드 클러스터에 대한 변경 피드를 설정합니다.

1. 아직 라이센스를 가지고 있지 않다면, [평가판 엔터프라이즈 라이센스를 요청](enterprise-licensing.html)하십시오.

2. 터미널 창에서 `cockroach`를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start --insecure --listen-addr=localhost --background
    ~~~

3. [Confluent Open Source 플랫폼](https://www.confluent.io/download/)(Kafka 포함)을 다운로드하고 압축을 푸십시오.

4. 압축을 푼 `confluent-<version>` 디렉토리로 이동하여 Confluent를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./bin/confluent start
    ~~~

    `zookeeper`와 `kafka`, `schema-registry`만 필요합니다. Confluent 문제를 해결하려면 [해당 문서](https://docs.confluent.io/current/installation/installing_cp.html#zip-and-tar-archives)를 참조하십시오.

5. Kafka 항목을 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./bin/kafka-topics \
    --create \
    --zookeeper localhost:8081 \
    --replication-factor 1 \
    --partitions 1 \
    --topic office_dogs
    ~~~

    {{site.data.alerts.callout_info}}
    필요한 복제 및 파티션의 개수만큼 Kafka 항목을 생성해야 합니다. [항목을 수동으로 생성](https://kafka.apache.org/documentation/#basic_ops_add_topic)하거나 [Kafka 중개자를 구성](https://kafka.apache.org/documentation/#topicconfigs)하여 기본 파티션 개수 및 복제 요소로 항목을 자동으로 생성할 수 있습니다.
    {{site.data.alerts.end}}

6. `root` 사용자로 [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)를 여십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure
    ~~~

7. 이메일을 통해 받은 단체명과 [엔터프라이즈 라이센스](enterprise-licensing.html) 키를 설정하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    > SET CLUSTER SETTING cluster.organization = '<organization name>';
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    > SET CLUSTER SETTING enterprise.license = '<secret>';
    ~~~

8. `cdc_demo`라는 데이터베이스를 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE cdc_demo;
    ~~~

9. 8.의 데이터베이스를 기본값으로 설정하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SET DATABASE = cdc_demo;
    ~~~

10. 테이블을 생성하고 데이터를 추가하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE office_dogs (
         id INT PRIMARY KEY,
         name STRING);
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO office_dogs VALUES
       (1, 'Petee'),
       (2, 'Carl');
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > UPDATE office_dogs SET name = 'Petee H' WHERE id = 1;
    ~~~

11. 변경 피드를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE CHANGEFEED FOR TABLE office_dogs INTO 'kafka://localhost:9092' WITH format = experimental_avro, confluent_schema_registry = 'http://localhost:8081';
    ~~~

    ~~~
            job_id       
    +--------------------+
      360645287206223873
    (1 row)
    ~~~

    이것은 백그라운드에서 변경 피드를 시작하고 `job_id`를 반환합니다. 변경 피드는 Kafka에 쓰여집니다.

12. 새 터미널에서, 압축을 푼 `confluent-<version>` 디렉토리로 이동하여 Kafka 항목을 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./bin/kafka-avro-console-consumer \
    --bootstrap-server=localhost:9092 \
    --property print.key=true \
    --from-beginning \
    --topic=office_dogs
    ~~~

    ~~~ shell
    {"id":1}    {"id":1,"name":{"string":"Petee H"}}
    {"id":2}    {"id":2,"name":{"string":"Carl"}}
    ~~~

    초기 스캔은 변경 피드가 시작된 시점부터 테이블의 상태를 표시합니다(따라서 `"Petee"`의 초기값은 생략됨).

13. SQL 클라이언트로 돌아가서 더 많은 데이터를 삽입하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO office_dogs VALUES (3, 'Ernie');
    ~~~

14. Kafka 항목을 관찰하고 있는 터미널로 돌아가면 다음과 같은 결과가 나타납니다:

    ~~~ shell
    {"id":3}    {"id":3,"name":{"string":"Ernie"}}
    ~~~

15. 완료되면 SQL 쉘을 종료하십시오(`\q`).

16. `cockroach`를 멈추려면 다음을 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach quit --insecure
    ~~~

17. Kafka를 멈추려면 압축을 푼 `confluent-<version>` 디렉토리로 이동하여 Confluence를 중단하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./bin/confluent stop
    ~~~

## 알려진 제한 사항

{% include v2.1/known-limitations/cdc.md %}

## 더 보

- [`CREATE CHANGEFEED`](create-changefeed.html)
- [`PAUSE JOB`](pause-job.html)
- [`CANCEL JOB`](cancel-job.html)
- [더 많은 SQL 명령문](sql-statements.html)
