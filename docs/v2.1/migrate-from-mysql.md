---
title: Migrate from MySQL
summary: Learn how to migrate data from MySQL into a CockroachDB cluster.
toc: true
build_for: [standard, managed]
---

{% include {{page.version.version}}/misc/beta-warning.md %}

이 페이지에서는 [`IMPORT`](import.html) 의 [`mysqldump`][mysqldump] 파일 읽기 지원을 사용하여 MySQL에서 CockroachDB로 데이터를 마이그레이션하는 방법을 설명합니다.

아래의 예는 [MySQL 문서](https://dev.mysql.com/doc/employee/en/)에서도 사용된 [직원 데이터 세트](https://github.com/datacharmer/test_db)을 사용한 예시입니다.

## 고려 사항

[마이그레이션 개요](migration-overview.html)에 나열된 일반적인 고려 사항 외에 마이그레이션을 준비할 때 고려해야 할 MySQL 관련 정보는 다음과 같습니다.

### 문자열 대/소문자 감도

.MySQL 문자열은 기본적으로 대/소문자를 구분하지 않지만, CockroachDB의 문자열은 대/소문자를 구분합니다. 이는 CockroachDB에서 원하는 결과를 얻으려면 MySQL 덤프 파일을 편집해야 할 수도 있다는 것을 의미합니다. 예를 들어, CockroachDB와 함께 작업하기 위해 변경해야 할 MySQL에서 문자열 비교를 수행해야할 수도 있습니다.

MySQL 문자열의 대/소문자 감도에 대한 자세한 내용은 MySQL 문서에서 [문자열 검색의 대/소문자 감도](https://dev.mysql.com/doc/refman/8.0/en/case-sensitivity.html)를 참조하십시오. CockroachDB 문자열에 대한 자세한 내용은 [`STRING`](string.html)을 참조하십시오.

## 1단계. MySQL 데이터베이스 덤프

MySQL에서 CockroachDB로 가져올 데이터를 덤프(시스템에 저장된 정보를 복사)하는 방법:

- [전체 데이터베이스 덤프](#dump-the-entire-database)
- [한 번에 하나의 테이블 덤프](#dump-one-table-at-a-time)

### 전체 데이터베이스 덤프

대부분의 사용자는 [전체 데이터베이스 덤프 가져오기](#import-a-full-database-dump)에서와 같이 전체 MySQL 데이터베이스를 한 번에 가져오려고 할 것입니다. 전체 데이터베이스를 덤프하려면 아래 나온 [`mysqldump`][mysqldump] 명령을 실행하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ mysqldump -uroot employees > /tmp/employees-full.sql
~~~

데이터베이스 덤프에서 테이블 하나만 가져오려면 아래의 [전체 데이터베이스 덤프에서 하나의 테이블 가져오기](#import-a-table-from-a-full-database-dump) b를 참조하십시오.

### 한 번에 하나의 테이블 덤프

이름이 `employees`인 MySQL 데이터베이스에서 `employees` 테이블을 덤프하려면 아래 표시된  [`mysqldump`][mysqldump] 명령을 실행하십시오. 아래 [테이블 덤프에서 테이블 가져오기](#import-a-table-from-a-table-dump) 의 방법을 사용하여 이 테이블을 가져올 수 있습니다.

{% include copy-clipboard.html %}
~~~ shell
$ mysqldump -uroot employees employees > employees.sql
~~~

## 2단계. 클러스터에서 액세스할 수 있는 파일 호스트

CockroachDB 클러스터의 각 노드는 가져올 파일에 액세스할 수 있어야 합니다. 클러스터가 데이터에 액세스할 수 있는 몇 가지 방법이 있습니다. [`IMPORT`][import]의 전체 저장소 유형은 [파일 URL 가져오기](import.html#import-file-urls)를 참조하십시오.

{{site.data.alerts.callout_success}}
가져오려는 데이터 파일을 호스팅하기 위해 Amazon S3 또는 Google Cloud와 같은 클라우드 스토리지를 사용할 것을 적극 권장합니다.
{{site.data.alerts.end}}

## 3단계. MySQL 덤프 파일 가져오기

전체 데이터베이스를 가져올지 아니면 한 테이블만 가져올지에 따라 [`IMPORT`][import]문의 여러 가지 종류 중에서 선택하여 사용할 수 있습니다.

- [Import a full database dump](#import-a-full-database-dump)
- [Import a table from a full database dump](#import-a-table-from-a-full-database-dump)
- [Import a table from a table dump](#import-a-table-from-a-table-dump)

본 섹션의 모든 [`IMPORT`][import]문은 [Amazon S3](https://aws.amazon.com/s3/)]에서 실제 데이터를 가져와  [`SHOW JOBS`](show-jobs.html) 로 모니터링할 수 있는 백그라운드 가져오기 작업을 시작합니다.

### 전체 데이터베이스 덤프 가져오기

이 예시에서는 [전체 데이터베이스를 덤프](#dump-the-entire-database)했다고 가정합니다.

아래의 [`IMPORT`][import] 문은 전체 데이터베이스 덤프에서 데이터와 [DDL](https://en.wikipedia.org/wiki/Data_definition_language) 문장을 읽어옵니다.(`CREATE TABLE` 와 [foreign key constraints](foreign-key.html) 포함)

{% include copy-clipboard.html %}
~~~ sql
> CREATE DATABASE IF NOT EXISTS employees;
> USE employees;
> IMPORT MYSQLDUMP 'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/mysqldump/employees-full.sql.gz';
~~~

~~~
       job_id       |  status   | fraction_completed |  rows   | index_entries | system_records |   bytes
--------------------+-----------+--------------------+---------+---------------+----------------+-----------
 382716507639906305 | succeeded |                  1 | 3919015 |        331636 |              0 | 110104816
(1 row)
~~~

### 전체 데이터베이스 덤프에서 하나의 테이블 가져오기

이 예시에서는 [전체 데이터베이스를 덤프](#dump-the-entire-database)했다고 가정합니다.

[`IMPORT`][import]는 전체 데이터베이스 덤프에서 한 개 테이블의 데이터를 가져올 수 있으며, 데이터를 읽고 덤프 파일에서 임의의 `CREATE TABLE` 문을 적용합니다.

{% include copy-clipboard.html %}
~~~ sql
> CREATE DATABASE IF NOT EXISTS employees;
> USE employees;
> IMPORT MYSQLDUMP 'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/mysqldump/employees.sql.gz';
~~~

~~~
       job_id       |  status   | fraction_completed |  rows  | index_entries | system_records |  bytes
--------------------+-----------+--------------------+--------+---------------+----------------+----------
 383839294913871873 | succeeded |                  1 | 300024 |             0 |              0 | 11534293
(1 row)
~~~

### 테이블 덤프에서 테이블 가져오기

아래의 예시에서는 [하나의 테이블만 덤프](#dump-one-table-at-a-time)했다고 가정합니다.

테이블 덤프를 가져오는 가장 간단한 방법은 아래와 같이 [`IMPORT TABLE`][import]를 실행하는 것입니다. 이는 테이블 데이터를 읽어오고 `CREATE TABLE`문을 덤프 파일로부터 읽어옵니다.

{% include copy-clipboard.html %}
~~~ sql
> CREATE DATABASE IF NOT EXISTS employees;
> USE employees;
> IMPORT TABLE employees FROM MYSQLDUMP 'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/mysqldump/employees.sql.gz';
~~~

~~~
       job_id       |  status   | fraction_completed |  rows  | index_entries | system_records |  bytes   
--------------------+-----------+--------------------+--------+---------------+----------------+----------
 383855569817436161 | succeeded |                  1 | 300024 |             0 |              0 | 11534293
(1 row)
~~~

테이블의 열을 지정해야 하는 경우 아래 문처럼 [`IMPORT TABLE`][import] 문을 사용하면 데이터는 가져오지만, 덤프 파일의 `CREATE TABLE`문을 무시하고 지정한 열에 의존합니다.
{% include copy-clipboard.html %}
~~~ sql
> CREATE DATABASE IF NOT EXISTS employees;
> USE employees;
> IMPORT TABLE employees (
    emp_no INT PRIMARY KEY,
    birth_date DATE NOT NULL,
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    gender STRING NOT NULL,
    hire_date DATE NOT NULL
  )
  MYSQLDUMP DATA ('https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/mysqldump/employees.sql.gz');
~~~

## 구성 옵션

이 옵션은 `IMPORT ... MYSQLDUMP` 에서 사용할 수 있습니다.

+ [외부 키 스킵하기](#skip-foreign-keys)

### 외부 키 스킵하기

기본적으로 [`IMPORT ... MYSQLDUMP`][import] 는 외부 키를 지원합니다.  **Default: false**.  `skip_foreign_keys` 옵션을 추가하여 덤프 파일의 DDL에 있는 외부 키 제약 조건을 무시하고 데이터 가져오기 속도를 높이십시오. 이 옵션을 사용하면 다른 테이블의 종속성으로 인해 실패할 수 있는 개별 테이블을 가져올 수도 있습니다. 

사용 예시:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT MYSQLDUMP 's3://your-external-storage/employees.sql?AWS_ACCESS_KEY_ID=123&AWS_SECRET_ACCESS_KEY=456' WITH skip_foreign_keys;
~~~

[외부 키 제약](foreign-key.html)은 데이터를 가져온 후 [`ALTER TABLE ... ADD CONSTRAINT`](add-constraint.html) 명령을 사용해 추가할 수 있습니다.

## 더 알아보기

- [`IMPORT`](import.html)
- [CSV로부터 마이그레이션][csv]
- [Postgres로부터 마이그레이션][postgres]
- [Postgres 또는 MySQL 응용 프로그램을 CockroachDB로 마이그레이션할 수 있는가?](frequently-asked-questions.html#can-a-postgresql-or-mysql-application-be-migrated-to-cockroachdb)
- [SQL 덤프(내보내기)](sql-dump.html)
- [데이터 백업](back-up-data.html)
- [데이터 복원](restore-data.html)
- [Built-in SQL 클라이언트 사용하기](use-the-built-in-sql-client.html)
- [다른 Cockroach 명령어](cockroach-commands.html)

<!-- Reference Links -->

[postgres]: migrate-from-postgres.html
[csv]: migrate-from-csv.html
[import]: import.html
[mysqldump]: https://dev.mysql.com/doc/refman/8.0/en/mysqldump-sql-format.html
