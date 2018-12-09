---
title: Migration Overview
summary: Learn how to migrate data into a CockroachDB cluster.
redirect_from: import-data.html
toc: true
build_for: [standard, managed]
---

CockroachDB는 아래의 데이터베이스에서 데이터 가져오기를 지원합니다:

- MySQL
- Postgres

그리고 다음의 데이터 형식으로 부터 데이터 가져오기를 지원합니다:

- CSV/TSV

이 페이지에는 CockroachDB로의 마이그레이션을 계획할 때 알아야 할 일반적인 고려 사항이 나열되어 있다.

아래 나열된 정보 외에도 마이그레이션할 데이터베이스(또는 데이터 형식)에 적용되는 구체적인 지침 및 고려 사항은 다음 페이지를 참조하십시오:

- [Migrate from Postgres][postgres]
- [Migrate from MySQL][mysql]
- [Migrate from CSV][csv]

## 가져오는 동안의 파일 저장소

마이그레이션 동안, 외부 파일 스토리지와 상호 작용하는 [`IMPORT`][import] 의 모든 기능은 모든 노드가 해당 스토리지에 대해 동일한 뷰를 가지고 있다고 가정합니다. 즉, 파일에서 가져오기 위해서는 모든 노드가 그 파일에 대해 동일한 액세스 권한을 가져야 합니다.

## 스키마 및 애플리케이션 변경

일반적으로 스키마를 변경하고, 앱이 데이터베이스와 상호 작용하는 방식을 변경해야 합니다. 아래의 것들을 확실히 하기 위해 **CockroachDB에 대해 귀하의 신청서를 테스트 할 것을 강력하게 권장** 합니다:

1. 데이터의 상태는 마이그레이션 후 기대하는 부분입니다.
2. 응용 프로그램의 작업 부하에 대해 예상대로 성능이 발휘됩니다. [CockroachDB에서 SQL 성능을 최적화하기 위한 모범 사례](performance-best-practices-overview.html)를 적용할 필요가 있을 것입니다.

## 데이터 유형 크기

특정 크기 이상에서는, [`STRING`](string.html)s, [`DECIMAL`](decimal.html)s, [`ARRAY`](array.html), [`BYTES`](bytes.html), 그리고 [`JSONB`](jsonb.html) 같은 많은 데이터 유형들이 [쓰기를 증폭시키는 것](https://en.wikipedia.org/wiki/Write_amplification)으로 인해 성능 문제가 발생 할 수 있습니다. 권장되는 크기 제한은 각 데이터 형식의 설명서를 참조하십시오.

## 지원되지 않는 데이터 유형

CockroachDB는 `ENUM` 또는 `SET` 데이터 유형을 지원하지 않습니다.

Postgres에서, [`CHECK` constraint](check.html) 을 사용해서 `ENUM` 형식을 모방 할 수 있습니다.  MySQL의 경우 가져오기 중에 이 변환을 자동으로 수행하십시오.

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE orders (
    id UUID PRIMARY KEY,
    -- ...
    status STRING check (
      status='processing' or status='in-transit' or status='delivered'
    ) NOT NULL,
    -- ...
  );
~~~

## 다른 참고 문헌

- [`IMPORT`][import]
- [Migrate from CSV][csv]
- [Migrate from MySQL][mysql]
- [Migrate from Postgres][postgres]
- [Can a Postgres or MySQL application be migrated to CockroachDB?](frequently-asked-questions.html#can-a-postgresql-or-mysql-application-be-migrated-to-cockroachdb)
- [Porting from PostgreSQL](porting-postgres.html)
- [SQL Dump (Export)](sql-dump.html)
- [Back Up and Restore](backup-and-restore.html)
- [Use the Built-in SQL Client](use-the-built-in-sql-client.html)
- [Other Cockroach Commands](cockroach-commands.html)

<!-- Links -->

[postgres]: migrate-from-postgres.html
[mysql]: migrate-from-mysql.html
[csv]: migrate-from-csv.html
[import]: import.html
