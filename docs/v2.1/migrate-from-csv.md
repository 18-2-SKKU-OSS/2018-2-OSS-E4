---
title: Migrate from CSV
summary: Learn how to migrate data from CSV files into a CockroachDB cluster.
toc: true
build_for: [standard, managed]
---

이 페이지에서는 [`IMPORT`][import]를 사용하여 CSV 파일의 데이터를 CockroachDB로 마이그레이션하는 방법을 설명합니다.

아래의 예는 [MySQL 문서](https://dev.mysql.com/doc/employee/en/)에서도 사용된 [직원 데이터 세트](https://github.com/datacharmer/test_db)을 사용한 예시입니다.

아래의 예에서는 [Amazon S3](https://aws.amazon.com/s3/)에서 실제 데이터를 수집합니다. [MySQL 문서](https://dev.mysql.com/doc/employee/en/)에서도 사용된 [직원 데이터 세트](https://github.com/datacharmer/test_db)를 사용하여 CSV 파일 세트로 덤프합니다.

CockroachDB의 [`IMPORT`][import]는 기존 테이블로 데이터 가져오기를 지원하지 않는다는 점에 유의하십시오.

## 1단계. 데이터를 CSV로 내보내기

CSV로 데이터를 내보내는 방법은 데이터베이스 설명서를 참조하십시오.

다음 요구 사항에 따라 테이블당 CSV 파일을 하나씩 내보내십시오.

- 파일은 [유효한 CSV 형식](https://tools.ietf.org/html/rfc4180)이어야하며 구분 기호는 단일 문자여야한다는 경고가 있어야 합니다. 쉼표 (예시: 탭) 이외의 문자를 사용하려면 [delimiter 옵션](import.html#delimiter)을 사용하여 맞춤 구분 기호를 설정하십시오.
- 파일은 UTF-8로 인코딩되어 있어야합니다.
- 다음 문자 중 하나가 필드에 나타나면 필드를 큰 따옴표로 묶어야 합니다.
     - 구분 기호 (`,`: 기본값)
     - 큰 따옴표 (`"`)
     - 개행 문자 (`\n`)
     - 캐리지 리턴 (`\r`)
- 큰 따옴표를 사용하여 필드를 묶는 경우 필드에 나타나는 큰 따옴표는 다른 큰 따옴표로 시작하여 먼저 빼내야합니다. 예시: `"aaa","b""bb","ccc"`
- 열의 유형이 [`BYTES`](bytes.html) 인 경우 유효한 UTF-8 문자열 또는 `\x` 로 시작하는 [16진수 바이트 리터럴](sql-constants.html#hexadecimal-encoded-byte-array-literals)이 될 수 있습니다. 예를 들어 값이 `1`, `2` 바이트가 되어야 하는 필드는 `\x0102`로 씁니다..

## Step 2. 클러스터에서 액세스할 수 있는 파일 호스트

CockroachDB 클러스터의 각 노드는 가져올 파일에 액세스할 수 있어야 합니다. 클러스터가 데이터에 액세스할 수 있는 몇 가지 방법이 있습니다. [`IMPORT`][import]의 전체 저장소 유형은 [파일 URL 가져오기](import.html#import-file-urls)를 참조하십시오.

{{site.data.alerts.callout_success}}
가져오려는 데이터 파일을 호스팅하기 위해 Amazon S3 또는 Google Cloud와 같은 클라우드 스토리지를 사용할 것을 적극 권장합니다.
{{site.data.alerts.end}}

## Step 3. CSV 가져오기

가져올 테이블 데이터의 스키마와 일치하는 [`IMPORT TABLE`][import] 문을 작성하십시오.

예를 들어, `employees.csv` 에서 `employees` 테이블로 데이터를 가져오려면 다음을 실행합니다:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees (
    emp_no INT PRIMARY KEY,
    birth_date DATE NOT NULL,
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    gender STRING NOT NULL,
    hire_date DATE NOT NULL
  ) CSV DATA ('https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/csv/employees.csv.gz');
~~~

~~~
       job_id       |  status   | fraction_completed |  rows  | index_entries | system_records |  bytes   
--------------------+-----------+--------------------+--------+---------------+----------------+----------
 381866942129111041 | succeeded |                  1 | 300024 |             0 |              0 | 13258389
(1 row)
~~~

가져올 각 CSV 파일에 대해 위의 내용을 반복하십시오.

외부 키 관계를 추가하려면 [`ALTER TABLE ... ADD CONSTRAINT`](add-constraint.html)를 실행해야 한다는 점에 유의하십시오.

## 구성 옵션

 [`IMPORT ... CSV`][import]에서 사용할 수 있는 옵션:

+ [열 구분 기호](#column-delimiter)
+ [코멘트 구문](#comment-syntax)
+ [스킵 헤더 행](#skip-header-rows)
+ [Null 문자열](#null-strings)
+ [파일 압축](#file-compression)

### 열 구분 기호

`delimiter` 옵션은 각 열이 끝나는 지점을 표시하는 유니코드 문자를 설정하는 데 사용됩니다  **Default: `,`**.

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
  CSV DATA ('s3://acme-co/employees.csv?AWS_ACCESS_KEY_ID=123&AWS_SECRET_ACCESS_KEY=456')
        WITH delimiter = e'\t';
~~~

### 코멘트 구문

`comment` 옵션은 건너뛸 데이터의 행을 표시하는 유니코드 문자를 결정합니다.

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
  CSV DATA ('s3://acme-co/employees.csv?AWS_ACCESS_KEY_ID=123&AWS_SECRET_ACCESS_KEY=456')
        WITH comment = '#';
~~~

### 스킵 헤더 행

`skip` 옵션은 파일을 가져올 때 건너뛸 헤더 행의 수를 결정합니다.

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
  CSV DATA ('s3://acme-co/employees.csv?AWS_ACCESS_KEY_ID=123&AWS_SECRET_ACCESS_KEY=456')
        WITH skip = '2';
~~~

### Null 문자열

`nullif` 옵션은 `NULL` 로 변환할 문자열을 정의합니다.

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
  CSV DATA ('s3://acme-co/employees.csv?AWS_ACCESS_KEY_ID=123&AWS_SECRET_ACCESS_KEY=456')
        WITH nullif = '';
~~~

### 파일 압축

`compress` 옵션은 가져올 CSV 파일에 사용할 압축 풀기 코덱을 정의합니다.
이 옵션에 포함되는 항목:

+ `gzip`: [gzip](https://en.wikipedia.org/wiki/Gzip) 알고리즘을 사용하여 파일 압축 해제
+ `bzip`: [bzip](https://en.wikipedia.org/wiki/Bzip2) 알고리즘을 사용하여 파일 압축 해제
+ `none`: 파일 압축 해제 사용하지 않음
+ `auto`: **Default**. 파일 확장자를 기준으로 추측 (`.csv` 의 경우 'none', `.gz` 의 경우 'gzip',`.bz` 와 `.bz2` 의 경우 'bzip').

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
  CSV DATA ('s3://acme-co/employees.csv.gz?AWS_ACCESS_KEY_ID=123&AWS_SECRET_ACCESS_KEY=456')
        WITH compress = 'gzip';
~~~

## 더 알아보기

- [`IMPORT`][import]
- [MySQL로부터 마이그레이션][mysql]
- [Postgres로부터 마이그레이션][postgres]
- [SQL 덤프 (내보내기)](sql-dump.html)
- [데이터 백업 및 복원](backup-and-restore.html)
- [Built-in SQL 클라이언트 사용하기](use-the-built-in-sql-client.html)
- [다른 Cockroach 명령어](cockroach-commands.html)

<!-- Reference Links -->

[postgres]: migrate-from-postgres.html
[mysql]: migrate-from-mysql.html
[import]: import.html
