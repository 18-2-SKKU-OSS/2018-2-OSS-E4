---
title: Migrate from Postgres
summary: Learn how to migrate data from Postgres into a CockroachDB cluster.
toc: true
build_for: [standard, managed]
---

{% include {{page.version.version}}/misc/beta-warning.md %}

이 페이지는 [`IMPORT`][import]의 [`pg_dump`][pgdump] 파일 읽기 지원을 사용하여 Postgres에서 CockroachDB로 데이터를 마이그레이션하는 방법을 설명합니다.

아래의 예는 [Amazon S3](https://aws.amazon.com/s3/)의 실제 데이터를 가져옵니다.  그것들은 [MySQL docs](https://dev.mysql.com/doc/employee/en/) 에서도 사용되는 [임플로이 데이터 세트](https://github.com/datacharmer/test_db) 을 사용합니다.  [pgloader][pgloader] 를 사용하여 데이터를 Postgres로 가져온 후 아래에서 설명하는대로 여기에서 사용하도록 수정했습니다.

## 단계 1. Postgres 데이터베이스 덤프

Postgres의 데이터를 CockroachDB로 가져 오기 위한 몇 가지 방법이 있습니다:

- [Dump the entire database](#dump-the-entire-database)
- [Dump one table at a time](#dump-one-table-at-a-time)

덤프 파일에 함수 또는 유형 정의가 있으면 임포트를 실패합니다.  아래와 같이 [`pg_dump`][pgdump] 를 호출하는 것 외에도 함수와 데이터 유형을 제거하기 위해 덤프 파일을 편집해야 할 수도 있습니다ㅏ.

또한 CockroachDB 의 [`IMPORT`][import] 는 Postgres의 비공개 [schemas][pgschema] 에서 자동으로 데이터를 가져 오는 것을 지원하지 않습니다.  이 문제를 해결하기 위해 덤프 파일을 편집하여 `CREATE TABLE` 문에서 테이블과 스키마 이름을 변경할 수 있습니다.

### 전체 데이터베이스 덤프하기

대부분의 사용자는 아래 [전체 데이터베이스 덤프 가져 오기](#import-a-full-database-dump) 에 표시된 것처럼 전체 Postgres 데이터베이스를 한꺼번에 가져 오기를 원할 것입니다.

전체 데이터베이스를 덤프하려면, [`pg_dump`][pgdump]  명령을 실행하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ pg_dump employees > /tmp/employees-full.sql
~~~

이 데이터 세트의 경우 Postgres 덤프 파일에는 아래 예제에서 사용 된 파일에 대해 이미 수행 된 편집이 필요합니다:

- `CREATE TABLE` 문의 `employees.gender` 컬럼의 타입은 `employees.employees_gender` 에서 [`STRING`](string.html) 로 변경되어야합니다. 왜냐하면 Postgres는 [`CREATE TYPE`](https://www.postgresql.org/docs/10/static/sql-createtype.html) 문을 사용하기 때문에 CockroachDB에서 지원하지 않습니다.

- `CREATE TYPE employee ...` 명령은 삭제되어야 합니다.

데이터베이스 덤프에서 하나의 테이블 만 가져 오려면 아래의 [전체 데이터베이스 덤프에서 테이블 가져 오기](#import-a-table-from-a-full-database-dump) 를 참조하십시오.

### 한 번에 하나의 테이블 덤프

`employees` 라는 이름의 Postgres 데이터베이스에서 `employees` 테이블을 덤프하려면, 아래의 [`pg_dump`][pgdump] 명령을 실행하십시오.  아래의 [테이블 덤프에서 테이블 가져 오기](#import-a-table-from-a-table-dump) 의 지침에 따라이 테이블을 가져올 수 있습니다.

{% include copy-clipboard.html %}
~~~ shell
$ pg_dump -t employees  employees > /tmp/employees.sql
~~~

이 데이터 세트의 경우, Postgres 덤프 파일은 아래 예제에서 사용 된 파일에서 이미 수행 된 다음 편집을 필요로 합니다.

- The type of the `employees.gender` column in the `CREATE TABLE` statement had to be changed from `employees.employees_gender` to [`STRING`](string.html) since Postgres represented the employee's gender using a [`CREATE TYPE`](https://www.postgresql.org/docs/10/static/sql-createtype.html) statement that is not supported by CockroachDB.

## 단계 2. 클러스터가 액세스 할 수있는 파일 호스트

CockroachDB 클러스터의 각 노드는 가져 오는 파일에 액세스 할 수 있어야합니다.  클러스터가 데이터에 액세스하는 데는 여러 가지 방법이 있습니다;  [`IMPORT`][import] 에서 가져올 수있는 저장 유형의 전체 목록은 [Import File URLs](import.html#import-file-urls) 을 참조하십시오..

{{site.data.alerts.callout_success}}
가져 오려는 데이터 파일을 호스팅하려면 Amazon S3 또는 Google Cloud와 같은 클라우드 저장소를 사용하는 것이 좋습니다.
{{site.data.alerts.end}}

## 단계 3. Postgres 덤프 파일 가져 오기

전체 데이터베이스 또는 단일 테이블을 가져올 것인지에 따라 [`IMPORT`][import] 문의 여러 변형 중에서 선택할 수 있습니다:

- [전체 데이터베이스 덤프 가져 오기](#import-a-full-database-dump)
- [전체 데이터베이스 덤프에서 테이블 가져 오기](#import-a-table-from-a-full-database-dump)
- [테이블 덤프에서 테이블 가져 오기](#import-a-table-from-a-table-dump)

이 섹션의 [`IMPORT`][import] 구문은 [Amazon S3](https://aws.amazon.com/s3/) 에서 실제 데이터를 가져 와서 [`SHOW JOBS`](show-jobs.html) 와 함께 모니터 할 수있는 백그라운드 가져 오기 작업을 시작합니다ㅏ.

### 전체 데이터베이스 덤프 가져 오기

이 예제에서는 [데이터베이스 전체를 덤프](#dump-the-entire-database) 했다고 가정합니다.

아래의 [`IMPORT`][import] 문은 전체 데이터베이스 덤프 파일에서 데이터 및 [DDL](https://en.wikipedia.org/wiki/Data_definition_language) 문 (기존 외래 키 관계 포함)을 읽습니다.

{% include copy-clipboard.html %}
~~~ sql
> IMPORT PGDUMP 'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/pg_dump/employees-full.sql.gz';
~~~

~~~
       job_id       |  status   | fraction_completed |  rows  | index_entries | system_records |  bytes
--------------------+-----------+--------------------+--------+---------------+----------------+----------
 381845110403104769 | succeeded |                  1 | 300024 |             0 |              0 | 11534293
(1 row)
~~~

### 전체 데이터베이스 덤프에서 테이블 가져 오기

이 예제는 [데이터베이스 전체를 덤프](#dump-the-entire-database) 라고 가정합니다.

[`IMPORT`][import] 는 전체 데이터베이스 덤프에서 하나의 테이블의 데이터를 가져올 수 있습니다. 그것은 데이터를 읽고 파일에서 모든 `CREATE TABLE` 문을 적용합니다.

{% include copy-clipboard.html %}
~~~ sql
> CREATE DATABASE IF NOT EXISTS employees;
> USE employees;
> IMPORT TABLE employees FROM PGDUMP 'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/pg_dump/employees-full.sql.gz';
~~~

~~~
       job_id       |  status   | fraction_completed |  rows  | index_entries | system_records |  bytes
--------------------+-----------+--------------------+--------+---------------+----------------+----------
 383839294913871873 | succeeded |                  1 | 300024 |             0 |              0 | 11534293
(1 row)
~~~

### 테이블 덤프에서 테이블 가져 오기

아래의 예는 [한테이블 덤프](#dump-one-table-at-a-time) 를 가정합니다.

테이블 덤프를 가져 오는 가장 간단한 방법은 아래와 같이 [`IMPORT TABLE`][import] 를 실행하는 것입니다.  그것은 파일로부터 테이블 데이터와 모든 `CREATE TABLE` 문을 읽습니다.

{% include copy-clipboard.html %}
~~~ sql
> CREATE DATABASE IF NOT EXISTS employees;
> USE employees;
> IMPORT PGDUMP 'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/pg_dump/employees.sql.gz';
~~~

~~~
       job_id       |  status   | fraction_completed |  rows  | index_entries | system_records |  bytes   
--------------------+-----------+--------------------+--------+---------------+----------------+----------
 383855569817436161 | succeeded |                  1 | 300024 |             0 |              0 | 11534293
(1 row)
~~~

어떤 이유로 테이블의 열을 지정해야하는 경우, 데이터를 가져 오지만 파일의 `CREATE TABLE` 문은 무시하고 지정한 열을 사용하는 [`IMPORT TABLE`][import] 문을 사용할 수 있습니다. 

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees (
    emp_no INT PRIMARY KEY,
    birth_date DATE NOT NULL,
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    gender STRING NOT NULL,
    hire_date DATE NOT NULL
  )
  PGDUMP DATA ('https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/pg_dump/employees.sql.gz');
~~~

## 구성 옵션

다음 옵션들은 `IMPORT ... PGDUMP`에서 사용 가능합니다 :

+ [Max row size](#max-row-size)
+ [Skip foreign keys](#skip-foreign-keys)

### 최대 행 크기

`max_row_size` 옵션은 라인 크기에 대한 제한을 덮어 쓰는 데 사용됩니다.  **Default: 0.5MB**.  이 설정은 Postgres 덤프 파일에 매우 긴 행이있는 경우 (예 : `COPY` 문의 경우) 조정해야 할 수 있습니다.

사용 예시:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees (
    emp_no INT PRIMARY KEY,
    birth_date DATE NOT NULL,
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    gender STRING NOT NULL,
    hire_date DATE NOT NULL
  )
  PGDUMP DATA ('s3://your-external-storage/employees.sql?AWS_ACCESS_KEY_ID=123&AWS_SECRET_ACCESS_KEY=456') WITH max_row_size = '5MB';
~~~

### 외국어 키 건너 뛰기

기본적으로, [`IMPORT ... PGDUMP`][import] 는 외래 키를 지원합니다.  **Default: false**.  덤프 파일의 DDL에서 외래 키 제약 조건을 무시하여 데이터 가져 오기 속도를 높이려면 `skip_foreign_keys` 옵션을 추가하십시오.  또한 다른 테이블에 대한 종속성으로 인해 실패할 개별 테이블을 가져올 수도 있습니다.

사용 예씨:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees (
    emp_no INTEGER NOT NULL,
    birth_date DATE NOT NULL,
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    gender STRING NOT NULL,
    hire_date DATE NOT NULL
  ) PGDUMP DATA ('s3://your-external-storage/employees.sql?AWS_ACCESS_KEY_ID=123&AWS_SECRET_ACCESS_KEY=456') WITH skip_foreign_keys;
~~~

[외래 키 제약 조건](foreign-key.html) 는 데이터를 가져온 후에 [`ALTER TABLE ... ADD CONSTRAINT`](add-constraint.html) 명령을 사용하여 추가 할 수 있습니다.

## 또 다른 참고문헌

- [`IMPORT`][import]
- [Migrate from CSV][csv]
- [Migrate from MySQL][mysql]
- [Can a Postgres or MySQL application be migrated to CockroachDB?](frequently-asked-questions.html#can-a-postgresql-or-mysql-application-be-migrated-to-cockroachdb)
- [SQL Dump (Export)](sql-dump.html)
- [Back up Data](back-up-data.html)
- [Restore Data](restore-data.html)
- [Use the Built-in SQL Client](use-the-built-in-sql-client.html)
- [Other Cockroach Commands](cockroach-commands.html)

<!-- Reference Links -->

[csv]: migrate-from-csv.html
[mysql]: migrate-from-mysql.html
[import]: import.html
[pgdump]: https://www.postgresql.org/docs/current/static/app-pgdump.html
[postgres]: migrate-from-postgres.html
[pgschema]: https://www.postgresql.org/docs/current/static/ddl-schemas.html
[pgloader]: https://pgloader.io/

<!-- Notes

이 지침은 다음 버전으로 준비되었습니다:

- Postgres 10.5

- CockroachDB CCL v2.2.0-alpha.00000000-757-gb33c49ff73
  (x86_64-apple-darwin16.7.0, built 2018/09/12 19:30:43, go1.10.3)
  (built from master on Wednesday, September 12, 2018)

- /usr/local/bin/mysql Ver 14.14 Distrib 5.7.22, for osx10.12 (x86_64)
  using EditLine wrapper

-->
