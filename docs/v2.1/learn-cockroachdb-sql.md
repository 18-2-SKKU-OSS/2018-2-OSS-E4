---
title: Learn CockroachDB SQL
summary: Learn some of the most essential CockroachDB SQL statements.
toc: true
build_for: [standard, managed]
---

이 페이지는 가장 핵심적인 CockroachDB SQL문을 보여줍니다. 전체 목록 및 관련 세부 정보는 [SQL 문](sql-statements.html)을 참조하십시오.

{% unless site.managed %}
{{site.data.alerts.callout_success}}
대화식 SQL쉘을 사용하여 이 명령문을 시도하십시오. 클러스터가 이미 실행 중이면 [`cockroach sql`](use-the-built-in-sql-client.html) 명령을 사용하십시오. 그렇지 않으면, [`cockroach demo`](cockroach-demo.html) 명령을 사용하여 쉘을 임시 메모리 내 클러스터에서 여십시오.
{{site.data.alerts.end}}
{% endunless %}

{{site.data.alerts.callout_info}}
CockroachDB는 표준 SQL에 확장 기능을 제공하는 것을 목표로 하지만, 일부 표준 SQL 기능은 아직 제공되지 않습니다. 자세한 내용은 [SQL 기능 지원](sql-feature-support.html) 페이지를 참조하십시오.
{{site.data.alerts.end}}

{% if site.managed %}
## 시작하기 전에

클러스터에 이미 [CockroachDB SQL 클라이언트를 연결](managed-connect-to-your-cluster.html#use-the-cockroachdb-sql-client)했는지 확인하십시오.

## 데이터베이스 생성

당신의 관리되는 CockroachDB 클러스터는 테스트를 위한 `defaultdb`와 일부 내부 데이터베이스뿐만 아니라 [확인 이메일](managed-sign-for-a-cluster.html)에 언급된 미리 생성된 데이터베이스와 함께 제공됩니다.

새 데이터베이스를 생성하려면, 초기 "admin"사용자와 연결하고 [`CREATE DATABASE`](create-database.html)와 그 뒤에 따라오는 데이터베이스 이름을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> CREATE DATABASE bank;
~~~

데이터베이스 이름은 [이 식별자 규칙](keywords-and-identifiers.html)뒤에 와야합니다. 데이터베이스가 이미 존재하는 경우 오류를 피하려면, `IF NOT EXISTS`을 포함시킬 수 있습니다:

{% include copy-clipboard.html %}
~~~ sql
> CREATE DATABASE IF NOT EXISTS bank;
~~~

더 이상 데이터베이스가 필요없으면, [`DROP DATABASE`](drop-database.html)와 그 뒤에 따라오는 데이터베이스 이름을 사용하여 데이터베이스와 모든 객체를 제거하십시오:

{% include copy-clipboard.html %}
~~~ sql
> DROP DATABASE bank;
~~~

## 데이터베이스 보기

모든 데이터베이스를 보려면, [`SHOW DATABASES`](show-databases.html) 문을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> SHOW DATABASES;
~~~

~~~
  database_name
+---------------+
  bank
  defaultdb
  postgres
  system
(4 rows)
~~~

## 기본 데이터베이스 설정

[연결 문자열](managed-sign-up-for-a-cluster.html)에 기본 데이터베이스를 직접 설정하는 것이 가장 좋습니다.

{% include copy-clipboard.html %}
~~~ sql
> SET DATABASE = bank;
~~~

기본 데이터베이스에서 작업할 때, 명시적으로 명령문에서 참조할 필요가 없습니다. whi를 보시려면

{% include copy-clipboard.html %}
~~~ sql
> SHOW DATABASE;
~~~

~~~
  database
+----------+
  bank
(1 row)
~~~
{% endif %}

## 테이블 생성

테이블을 생성하려면, 각 열에 대해 [`CREATE TABLE`](create-table.html)과 그 뒤에 따라오는 테이블 이름, 열 이름, [데이터 타입](data-types.html), 그리고 [제약사항](constraints.html)그리고 [데이터 형식](data-types.html)을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL
);
~~~

테이블과 열 이름은 [이 규칙](keywords-and-identifiers.html#identifiers)뒤에 와야합니다. 또한 [기본키](primary-key.html)를 명시적으로 정의하지 않으면, CockroachDB는 숨겨진 `rowid` 열을 기본키로 자동 추가합니다.

테이블이 이미 존재할 경우, 에러를 피하기 위해 `IF NOT EXISTS`를 포함시킬 수 있습니다:

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE IF NOT EXISTS accounts (
    id INT PRIMARY KEY,
    balance DECIMAL
);
~~~

테이블의 모든 열을 표시하려면, [`SHOW COLUMNS FROM`](show-columns.html)과 뒤에 따라오는 테이블 이름을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> SHOW COLUMNS FROM accounts;
~~~

~~~
  column_name | data_type | is_nullable | column_default | generation_expression |   indices   | is_hidden
+-------------+-----------+-------------+----------------+-----------------------+-------------+-----------+
  id          | INT       |    false    | NULL           |                       | {"primary"} |   false
  balance     | DECIMAL   |    true     | NULL           |                       | {}          |   false
(2 rows)
~~~

더 이상 테이블이 필요없으면, [`DROP TABLE`](drop-table.html)와 그 뒤에 따라오는 테이블 이름을 사용하여 테이블과 모든 데이터를 제거합니다:

{% include copy-clipboard.html %}
~~~ sql
> DROP TABLE accounts;
~~~

## 테이블 보기

활성 데이터베이스의 모든 테이블을 보려면, [`SHOW TABLES`](show-tables.html) 문을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> SHOW TABLES;
~~~

~~~
  table_name
+------------+
  accounts
(1 row)
~~~

## 테이블에 행 삽입

테이블에 행을 삽입하려면, [`INSERT INTO`](insert.html)와 그 뒤에 따라오는 테이블 이름, 열이 나타나는 순서대로 나열된 열 값을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO accounts VALUES (1, 10000.50);
~~~

다른 순서로 열 값을 전달하려면, 열 이름을 명시적으로 나열하고 해당 순서대로 열 값을 제공하십시오:

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO accounts (balance, id) VALUES
    (25000.00, 2);
~~~

테이블에 여러 행을 삽입하려면, 한 행에 대한 열 값을 포함하는 괄호로 묶은 쉼표로 구분된 목록을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO accounts VALUES
    (3, 8100.73),
    (4, 9400.10);
~~~

[기본값](default-value.html)은 특정 열을 명령문에서 제외하거나, 명시적으로 기본값을 요청할 때 사용됩니다. 예를 들어, 다음 명령문 모두 `balance` 가 디폴트 값으로 채워진 행을 생성합니다 (이 경우 `NULL`):

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO accounts (id) VALUES
    (5);
~~~

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO accounts (id, balance) VALUES
    (6, DEFAULT);
~~~

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM accounts WHERE id in (5, 6);
~~~

~~~
  id | balance
+----+---------+
   5 | NULL
   6 | NULL
(2 rows)
~~~

## 인덱스 생성

[인덱스](indexes.html)는 테이블의 모든 행을 살펴 보지 않고도 데이터를 찾을 수 있도록 도와줍니다. 그것들은 테이블의 [기본키](primary-key.html)와 [`UNIQUE` 제약 조건](unique.html)이 있는 열을 위해 자동으로 생성됩니다.

고유하지 않은 열에 대한 인덱스를 작성하려면 [`CREATE INDEX`](create-index.html)와 그 뒤에 따라오는 선택할 인덱스 이름, 테이블과 인덱스할 열을 식별하는 `ON`절을 사용하십시오. 각 열에 대해 오름차순 정렬(`ASC`) 또는 내림차순 (`DESC`) 정렬 여부를 선택할 수 있습니다.

{% include copy-clipboard.html %}
~~~ sql
> CREATE INDEX balance_idx ON accounts (balance DESC);
~~~

테이블 생성 중에도 또한 인덱스를 생성할 수 있습니다; `INDEX` 키워드 다음에 선택할 인덱스 이름과 인덱스할 컬럼을 포함하십시오:

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL,
    INDEX balance_idx (balance)
);
~~~

## 테이블의 인덱스 보기

테이블에 인덱스를 표시하려면, [`SHOW INDEX FROM`](show-index.html)와 그 뒤에 따라오는 테이블 이름을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> SHOW INDEX FROM accounts;
~~~

~~~
  table_name | index_name  | non_unique | seq_in_index | column_name | direction | storing | implicit
+------------+-------------+------------+--------------+-------------+-----------+---------+----------+
  accounts   | primary     |   false    |            1 | id          | ASC       |  false  |  false
  accounts   | balance_idx |    true    |            1 | balance     | DESC      |  false  |  false
  accounts   | balance_idx |    true    |            2 | id          | ASC       |  false  |   true
(3 rows)
~~~

## 테이블 쿼리하기

테이블을 쿼리하려면, [`SELECT`](select-clause.html)와 그 뒤에 따라오는 쉼표로 구분된 반환할 열 목록, 데이터를 검색할 테이블을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> SELECT balance FROM accounts;
~~~

~~~
  balance
+----------+
  10000.50
  25000.00
   8100.73
   9400.10
  NULL
  NULL
(6 rows)
~~~

모든 열을 검색하려면, `*` 와일드 카드를 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM accounts;
~~~

~~~
  id | balance
+----+----------+
   1 | 10000.50
   2 | 25000.00
   3 |  8100.73
   4 |  9400.10
   5 | NULL
   6 | NULL
(6 rows)
~~~

결과를 필터링하려면, 필터링 할 열과 값을 식별하는 `WHERE` 절을 추가하십시오:

{% include copy-clipboard.html %}
~~~ sql
> SELECT id, balance FROM accounts WHERE balance > 9000;
~~~

~~~
  id | balance
+----+----------+
   2 | 25000.00
   1 | 10000.50
   4 |  9400.10
(3 rows)
~~~

결과를 정렬하려면, 정렬할 열을 식별하는 `ORDER BY`절을 추가하십시오. 각 열에 대해, 오름차순 정렬 (`ASC`) 또는 내림차순 (`DESC`) 정렬 여부를 선택할 수 있습니다.

{% include copy-clipboard.html %}
~~~ sql
> SELECT id, balance FROM accounts ORDER BY balance DESC;
~~~

~~~
  id | balance
+----+----------+
   2 | 25000.00
   1 | 10000.50
   4 |  9400.10
   3 |  8100.73
   5 | NULL
   6 | NULL
(6 rows)
~~~

## 테이블의 행 업데이트

테이블의 행을 업데이트하려면, [`UPDATE`](update.html)과 그 뒤에 따라오는 테이블 이름, 업데이트할 열과 새로운 값을 식별하는 `SET` 절, 업데이트할 열을 식별하는 `WHERE`절을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> UPDATE accounts SET balance = balance - 5.50 WHERE balance < 10000;
~~~

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM accounts;
~~~

~~~
  id | balance
+----+----------+
   1 | 10000.50
   2 | 25000.00
   3 |  8095.23
   4 |  9394.60
   5 | NULL
   6 | NULL
(6 rows)
~~~

테이블에 기본 키가 있으면, `WHERE` 절에서 이를 사용하여 특정 행을 안정적으로 업데이트 할 수 있습니다. 그렇지 않으면 `WHERE` 절과 일치하는 각 행이 갱신됩니다. `WHERE` 절이 없으면 테이블의 모든 행이 갱신됩니다.

## 테이블의 행 삭제

테이블에서 행을 삭제하려면, [`DELETE FROM`](delete.html)과 그 뒤에 따라오는 테이블 이름을 사용하고 삭제할 행을 식별하는 `WHERE` 절을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> DELETE FROM accounts WHERE id in (5, 6);
~~~

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM accounts;
~~~

~~~
  id | balance
+----+----------+
   1 | 10000.50
   2 | 25000.00
   3 |  8095.23
   4 |  9394.60
(4 rows)
~~~

`UPDATE` 문과 마찬가지로 테이블에 기본 키가 있으면, `WHERE`절에 있는 기본키를 사용하여 특정 행을 확실하게 삭제할 수 있습니다. 그렇지 않으면`WHERE` 절과 일치하는 각 행이 삭제됩니다. `WHERE` 절이 없으면 테이블의 모든 행이 삭제됩니다.

{% unless site.managed %}
## 다음은?

- 모든 [SQL 문](sql-statements.html) 탐색
- [빌트인 SQL 클라이언트 사용](use-the-built-in-sql-client.html)하여 쉘 또는 명령줄에서 직접 명령문을 실행하기
- 선호하는 언어에 대하여 [클라이언트 드라이버 설치](install-client-drivers.html) 와 [앱 빌드](build-an-app-with-cockroachdb.html)
- 자동 복제, 재조정 및 결함 허용과 같은 [핵심 CockroachDB 기능 탐색](demo-data-replication.html) 
{% endunless %}
