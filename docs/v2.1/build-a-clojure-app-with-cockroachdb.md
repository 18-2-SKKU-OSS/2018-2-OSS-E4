---
title: Build a Clojure App with CockroachDB
summary: Learn how to use CockroachDB from a simple Clojure application with a low-level client driver.
toc: true
twitter: false
---

이 튜토리얼에서는 [leiningen](https://leiningen.org/)와 PostgreSQL과 호환되는 드라이버를 사용하여 CockroachDB로 간단한 Clojure 어플리케이션을 제작하는 방법을 보여줍니다.

[Clojure java.jdbc driver](http://clojure-doc.org/articles/ecosystem/java_jdbc/home.html)는 [PostgreSQL JDBC driver](https://jdbc.postgresql.org/)와 함께  **베타-레벨** 지원을 요청할 수 있을 정도로 테스트 되었고, 여기에 사용되었습니다. 만약 문제가 발생할 경우 상세 설명과 함께 [이슈 열기](https://github.com/cockroachdb/cockroach/issues/new)를 하여 저희가 전체를 지원할 수 있도록 도와주시길 부탁드립니다.


## 시작하기 전에

[설치된 CockroachDB](install-cockroachdb.html)가 이미 설치되어있는지 확인하시오.

## 1단계. `leiningen` 설치하기

[공식 설명서](https://leiningen.org/)에 설명된 대로 Clojure `lein` 유틸리티를 설치하십시오.

{% include {{ page.version.version }}/app/common-steps.md %}

## 5단계. 새로운 데이터베이스에 표 생성하기

`maxroach` 사용자로 [built-in SQL client](use-the-built-in-sql-client.html)를 사용하여 새로운 데이터베이스에 `accounts` 표를 생성하시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure \
--database=bank \
--user=maxroach \
-e 'CREATE TABLE accounts (id INT PRIMARY KEY, balance INT)'
~~~

## 6단계. Clojure 코드 실행하기

이제 데이터베이스와 사용자가 있으므로 코드를 실행하여 표를 만들고 행을 삽입한 다음, 코드를 실행하여 원자성 [트랜잭션](transactions.html)으로 값을 읽고 업데이트합니다.

### 기초적인 Clojure/JDBC project 생성하기

1. 새디렉터리 `myapp`을 만드시오.
2. `myapp/project.clj` 파일을 생성하고 다음 코드로 채우거나,  <a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/project.clj" download>직접 다운로드 하시오</a>. 프로젝트의 하위 디렉터리 `src/test`에 파일을 저장하시오.

    {% include copy-clipboard.html %}
    ~~~ clojure
    {% include {{ page.version.version }}/app/project.clj %}
    ~~~

3. `myapp/src/test/util.clj` 파일을 생성하고 <a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/util.clj" download>파일의</a> 코드로 채우시오. 프로젝트의 하위 디렉터리 `src/test`에 파일을 저장하시오.

### 기초적인 명령문


첫째로, 다음 코드를 이용하여 `maxroach` 사용자에 연결하고, 몇 가지 기본 SQL 명령문을 실행하고, 표를 생성하고, 행을 삽입하고, 행을 읽고 인쇄합니다.

`myapp/src/test/test.clj` 파일을 생성하고 아래 코드를 복사하거나, <a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/basic-sample.clj" download>직접 다운로드하시오</a>. 프로젝트의 하위 디렉터리 `src/test`에서 이 파일의 이름을 `test.clj`로 바꾸시오.

{% include copy-clipboard.html %}
~~~ clojure
{% include {{ page.version.version }}/app/basic-sample.clj %}
~~~

실행하시오:

{% include copy-clipboard.html %}
~~~ shell
lein run
~~~

### 트랜잭션 (재시도 논리 사용)

다음으로, 다음 코드를 이용하여 다시 `maxroach` 사용자로 연결하지만, 이번에는 포함된 모든 명령문이 커밋되거나 중단되는 한 계좌에서 다른 계좌로 자금을 이전하기 위해 원자성 트랜잭션으로 명령문 그룹을 실행할 것입니다.

아래의 코드를 `myapp/src/test/test.clj`에 복사하거나 
<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/txn-sample.clj" download>직접 다운로드 받으시오</a>. 파일명`test.clj`은 유지하십시오.
{{site.data.alerts.callout_info}}
CockroachDB는 읽기/쓰기 경합 시 
[클라이언트가 트랜잭션을 다시 시도하기](transactions.html#transaction-retries)를 요구할 수 있습니다. CockroachDB 는 트랜잭션 내에서 실행되고 필요에 따라 재시도하는 일반적인 **재시도 함수**를 제공합니다. 여기서 재시도 함수를 복사하여 코드에 붙여 넣을 수 있습니다.
{{site.data.alerts.end}}

{% include copy-clipboard.html %}
~~~ clojure
{% include {{ page.version.version }}/app/txn-sample.clj %}
~~~

실행하시오:

{% include copy-clipboard.html %}
~~~ shell
lein run
~~~

코드를 실행시긴 후에, 한 계좌에서 다른 계좌로 자금이 이전되었는지 확인하려면 [built-in SQL client](use-the-built-in-sql-client.html)을 사용하시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure -e 'SELECT id, balance FROM accounts' --database=bank
~~~

~~~
+----+---------+
| id | balance |
+----+---------+
|  1 |     900 |
|  2 |     350 |
+----+---------+
(2 rows)
~~~

## 더 보기

[Clojure java.jdbc driver](http://clojure-doc.org/articles/ecosystem/java_jdbc/home.html) 사용법에 대해 자세히 읽어보세요.

{% include {{ page.version.version }}/app/see-also-links.md %}
