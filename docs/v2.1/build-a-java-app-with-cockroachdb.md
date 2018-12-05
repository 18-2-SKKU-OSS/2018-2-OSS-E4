---
title: Build a Java App with CockroachDB
summary: Learn how to use CockroachDB from a simple Java application with the JDBC driver.
toc: true
twitter: false
---

<div class="filters filters-big clearfix">
    <a href="build-a-java-app-with-cockroachdb.html"><button class="filter-button current">Use <strong>JDBC</strong></button></a>
    <a href="build-a-java-app-with-cockroachdb-hibernate.html"><button class="filter-button">Use <strong>Hibernate</strong></button></a>
</div>

이 튜토리얼에서는 PostgreSQL과 호환되는 드라이버나 ORM을 사용하여 CockroachDB로 간단한 Java 어플리케이션을 제작하는 방법을 보여줍니다.

[Java JDBC driver](https://jdbc.postgresql.org/)와 [Hibernate ORM](http://hibernate.org/)는 **베타-레벨** 지원을 요청할 수 있을 정도로 테스트 되었고, 여기에 사용되었습니다.만약 문제가 발생할 경우 상세 설명과 함께 [이슈 열기](https://github.com/cockroachdb/cockroach/issues/new)를 하여 저희가 전체를 지원할 수 있도록 도와주시길 부탁드립니다.

## 시작하기 전에

{% include {{page.version.version}}/app/before-you-begin.md %}

{{site.data.alerts.callout_danger}}
이 페이지의 예제에서는 java 버전 9를 사용한다고 가정합니다. Java 10과는 작동하지 않습니다.
{{site.data.alerts.end}}

## 1단계. Java JDBC driver 설치하기

[공식 문서](https://jdbc.postgresql.org/documentation/head/setup.html)에 설명된 대로 Java JDBC 드라이버를 다운로드하고 설치하시오.

<section class="filter-content" markdown="1" data-scope="secure">

## 2단계. `maxroach` 사용자와 `bank` 데이터베이스 생성하기

{% include {{page.version.version}}/app/create-maxroach-user-and-bank-database.md %}

## 3단계. `maxroach` 사용자에 대한 인증서 생성하기

다음 명령을 실행하여 `maxroach` 사용자에 대한 인증서와 키를 생성하시오. 코드 샘플은 이 사용자로 실행됩니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach cert create-client maxroach --certs-dir=certs --ca-key=my-safe-directory/ca.key
~~~

## 4단계. Java에 사용할 키 파일 변환하기

CockroachDB에서 사용자 `maxroach` 용으로 생성한 개인 키는 [PEM encoded](https://tools.ietf.org/html/rfc1421)입니다. 키를 Java 어플리케이션에서 읽으려면 Java의 표준 키 인코딩 형식인 [PKCS#8 format](https://tools.ietf.org/html/rfc5208)으로 변환해야 합니다.

키를 PKCS#8 형식으로 변환하려면, 이 예제의 인증서를 저장한 디렉토리의 `maxroach` 사용자의 키 파일에서 다음의 OpenSSL 명령을 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ openssl pkcs8 -topk8 -inform PEM -outform DER -in client.maxroach.key -out client.maxroach.pk8 -nocrypt
~~~

## 5단계. Java 코드 실행하기

데이터베이스를 생성하고 암호화 키를 설정했으므로, 이 섹션에서는 다음을 수행합니다:

- [표를 생성하고 행을 삽입하기](#basic1)
- [한 묶음의 명령문을 트랜잭션으로 실행하기](#txn1)

<a name="basic1"></a>

### 기초적인 예시

첫째로, 다음 코드를 이용하여 `maxroach` 사용자에 연결하고, 몇 가지 기본 SQL 명령문: 표를 생성하고, 행을 삽입하고, 행을 읽고 인쇄하기 들을 실행합니다.

실행하려면:

1. [`BasicSample.java`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/BasicSample.java)파일을 다운로드하거나 직접 파일을 만들고 아래의 코드를 복사하시오.
2. [the PostgreSQL JDBC driver](https://jdbc.postgresql.org/download.html)를 다운로드하시오.
3. 컴파일하고 코드를 실행시키시오 (PostgreSQL JDBC driver를 classpath에 추가하시오):

    {% include copy-clipboard.html %}
    ~~~ shell
    $ javac -classpath .:/path/to/postgresql.jar BasicSample.java
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ java -classpath .:/path/to/postgresql.jar BasicSample
    ~~~

    출력은 다음과 같아야 합니다:

    ~~~
    Initial balances:
        account 1: 1000
        account 2: 250
    ~~~

[`BasicSample.java`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/BasicSample.java)의 내용:

{% include copy-clipboard.html %}
~~~ java
{% include {{page.version.version}}/app/BasicSample.java %}
~~~

<a name="txn1"></a>

### 트랜잭션 예시 (재시도 논리 사용)

다음으로, 다음 코드를 이용하여 한 계좌에서 다른 계좌로 자금을 이전하기 위해 한 묶음의 명령문들을 [트랜잭션](transactions.html)으로 실행시키시오. 

실행하려면:

1. <a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/TxnSample.java" download><code>TxnSample.java</code></a>를 다운로드하거나 직접 파일을 만들고 아래의 코드를 복사하시오. `getErrorCode()` 대신에 [`SQLException.getSQLState()`](https://docs.oracle.com/javase/tutorial/jdbc/basics/sqlexception.html)의 사용을 주목하십시오.
2. 컴파일하고 코드를 실행시키시오 (PostgreSQL JDBC driver를 classpath에 다시 추가하시오):

    {% include copy-clipboard.html %}
    ~~~ shell
    $ javac -classpath .:/path/to/postgresql.jar TxnSample.java
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ java -classpath .:/path/to/postgresql.jar TxnSample
    ~~~

    출력은 다음과 같아야 합니다:

    ~~~
    account 1: 900
    account 2: 350
    ~~~

{% include v2.1/client-transaction-retry.md %}

{% include copy-clipboard.html %}
~~~ java
{% include {{page.version.version}}/app/TxnSample.java %}
~~~

한 계좌에서 다른 계좌로 자금이 이전되었는지 확인하려면, [built-in SQL client](use-the-built-in-sql-client.html)을 시작하시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --certs-dir=certs --database=bank
~~~

계좌의 잔액을 확인하려면 다음의 명령문을 실행시키시오:

{% include copy-clipboard.html %}
~~~ sql
> SELECT id, balance FROM accounts;
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

</section>

<section class="filter-content" markdown="1" data-scope="insecure">

## 2단계. `maxroach` 사용자와 `bank` 데이터베이스 생성하기

{% include {{page.version.version}}/app/insecure/create-maxroach-user-and-bank-database.md %}

## 3. Java 코드 실행하기

데이터베이스를 생성했으므로, 이 섹션에서는 다음을 수행합니다:

- [표를 생성하고 행을 삽입하기](#basic2)
- [한 묶음의 명령문을 트랜잭션으로 실행하기](#txn2)

<a name="basic2"></a>

### 

첫째로, 다음 코드를 이용하여 `maxroach` 사용자에 연결하고, 몇 가지 기본 SQL 명령문: 표를 생성하고, 행을 삽입하고, 행을 읽고 인쇄하기 들을 실행합니다.

실행하려면:

1. [`BasicSample.java`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/insecure/BasicSample.java), 파일을 다운로드하거나 직접 파일을 만들고 아래의 코드를 복사하시오.
2. [the PostgreSQL JDBC driver](https://jdbc.postgresql.org/download.html)를 다운로드하시오.
3. 컴파일하고 코드를 실행시키시오 (PostgreSQL JDBC driver를 classpath에 추가하시오):

    {% include copy-clipboard.html %}
    ~~~ shell
    $ javac -classpath .:/path/to/postgresql.jar BasicSample.java
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ java -classpath .:/path/to/postgresql.jar BasicSample
    ~~~

[`BasicSample.java`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/insecure/BasicSample.java)의 내용:

{% include copy-clipboard.html %}
~~~ java
{% include {{page.version.version}}/app/insecure/BasicSample.java %}
~~~

<a name="txn2"></a>

### 트랜잭션 예시 (재시도 논리 사용)

다음으로, 다음 코드를 이용하여 한 계좌에서 다른 계좌로 자금을 이전하기 위해 한 묶음의 명령문들을 [트랜잭션](transactions.html)으로 실행시키시오. 

실행하려면:

1. <a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/insecure/TxnSample.java" download><code>TxnSample.java</code></a>를 다운로드하거나 직접 파일을 만들고 아래의 코드를 복사하시오. `getErrorCode()`  use of [`SQLException.getSQLState()`](https://docs.oracle.com/javase/tutorial/jdbc/basics/sqlexception.html)의 사용을 주목하십시오.
2. 컴파일하고 코드를 실행시키시오 (PostgreSQL JDBC driver를 classpath에 다시 추가하시오):

    {% include copy-clipboard.html %}
    ~~~ shell
    $ javac -classpath .:/path/to/postgresql.jar TxnSample.java
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ java -classpath .:/path/to/postgresql.jar TxnSample
    ~~~

{% include v2.1/client-transaction-retry.md %}

{% include copy-clipboard.html %}
~~~ java
{% include {{page.version.version}}/app/insecure/TxnSample.java %}
~~~

한 계좌에서 다른 계좌로 자금이 이전되었는지 확인하려면, [built-in SQL client](use-the-built-in-sql-client.html)를 시작하시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure --database=bank
~~~

계좌의 잔액을 확인하려면 다음의 명령문을 실행시키시오:

{% include copy-clipboard.html %}
~~~ sql
> SELECT id, balance FROM accounts;
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

</section>

## 더 보기

[Java JDBC driver](https://jdbc.postgresql.org/) 사용에 대해 자세히 알아보세요..

{% include {{page.version.version}}/app/see-also-links.md %}
