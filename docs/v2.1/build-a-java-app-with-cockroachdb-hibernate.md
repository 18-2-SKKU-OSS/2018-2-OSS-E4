---
title: Build a Java App with CockroachDB
summary: Learn how to use CockroachDB from a simple Java application with the Hibernate ORM.
toc: true
twitter: false
---

<div class="filters filters-big clearfix">
    <a href="build-a-java-app-with-cockroachdb.html"><button class="filter-button">Use <strong>JDBC</strong></button></a>
    <a href="build-a-java-app-with-cockroachdb-hibernate.html"><button class="filter-button current">Use <strong>Hibernate</strong></button></a>
</div>

이 튜토리얼에서는 PostgreSQL과 호환되는 드라이버나 ORM을 사용하여 CockroachDB로 간단한 Java 어플리케이션을 제작하는 방법을 보여줍니다.

[Java JDBC driver](https://jdbc.postgresql.org/)와 [Hibernate ORM](http://hibernate.org/)는 **베타-레벨** 지원을 요청할 수 있을 정도로 테스트 되었고, 여기에 사용되었습니다.만약 문제가 발생할 경우 상세 설명과 함께 [이슈 열기](https://github.com/cockroachdb/cockroach/issues/new)를 하여 저희가 전체를 지원할 수 있도록 도와주시길 부탁드립니다.

{{site.data.alerts.callout_success}}
Hibernate를 CockroachDB와 함께 보다 현실적으로 사용하려면,  [`examples-orms`](https://github.com/cockroachdb/examples-orms) repository의 예시를 참조하세요.
{{site.data.alerts.end}}

## 시작하기 전에

{% include {{page.version.version}}/app/before-you-begin.md %}

{{site.data.alerts.callout_danger}}
이 페이지의 예제에서는 java 버전 9를 사용한다고 가정합니다. Java 10과는 작동하지 않습니다.
{{site.data.alerts.end}}

## 1단계. Gradle 빌드 툴 설치하기

이 튜토리얼은 [Gradle build tool](https://gradle.org/)을 사용하여 Hibernate를 포함하여 어플리케이션의 모든 종속성을 가져옵니다.

Mac에서 Gradle을 설치하려면, 다음의 명령을 실행시키시오:

{% include copy-clipboard.html %}
~~~ shell
$ brew install gradle
~~~

Ubuntu와 같은 Debian-기반 Linux 배포판에서 Gradle을 설치하려면:

{% include copy-clipboard.html %}
~~~ shell
$ apt-get install gradle
~~~

Fedora와 같은 Red Hat-기반 Linux 배포판에서 Gradle을 설치하려면:

{% include copy-clipboard.html %}
~~~ shell
$ dnf install gradle
~~~

Gradle을 설치하는 다른 방법은 [its official documentation](https://gradle.org/install)을 참조하세요.

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

다음의 파일들을 포함하는 Java 프로젝트를 다운로드 및 추출 [hibernate-basic-sample.tgz](https://github.com/cockroachdb/docs/raw/master/_includes/v2.1/app/hibernate-basic-sample/hibernate-basic-sample.tgz):

파일 | 설명
-----|------------
[`Sample.java`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/hibernate-basic-sample/Sample.java) |[Hibernate](http://hibernate.org/orm/)를 사용하여 Java 객체 를 SQL operation에 매핑합니다. 더 많은 정보는 [Sample.java](#sample-java)을 참조하십시오.
[`hibernate.cfg.xml`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/hibernate-basic-sample/hibernate.cfg.xml) | 데이터베이스에 연결하는 방법을 지정하고, 어플리케이션을 실행할 때마다 데이터베이스 스키마를 삭제하고 다시 만들도록 합니다. 더 많은 정보는 [hibernate.cfg.xml](#hibernate-cfg-xml)을 참조하십시오.
[`build.gradle`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/hibernate-basic-sample/build.gradle) | 어플리케이션을 빌드하고 실행하는데 사용됩니다. 더 많은 정보는 [build.gradle](#build-gradle)을 참조하십시오.

`hibernate-basic-sample` 디렉토리에서 어플리케이션을 빌드하고 실행시키시오:

{% include copy-clipboard.html %}
~~~ shell
$ gradle run
~~~

출력의 끝 부분은 다음과 같이 보여아합니다:

~~~
1 1000
2 250
~~~

표와 행이 성공적으로 생성되었는지 확인하려면, [built-in SQL client](use-the-built-in-sql-client.html)을 시작하십시오:

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
|  1 |    1000 |
|  2 |     250 |
+----+---------+
(2 rows)
~~~

### Sample.java

아래의 Java 코드는 [Hibernate ORM](http://hibernate.org/orm/)을 사용하여 Java 객체를 SQL operation에 매핑합니다. 특히, 이 코드는:

- `Account` 클래스를 바탕으로 데이터베이스에 `accounts` 표를 생성합니다.

- `session.save(new Account())`을 사용하여 표에 행을 삽입합니다.

- `CriteriaQuery<Account> query` 객체를 사용하여 잔액을 출력할 수 있도록 표에서 선택할 SQL 쿼리를 정합니다.

{% include copy-clipboard.html %}
~~~ java
{% include {{page.version.version}}/app/hibernate-basic-sample/Sample.java %}
~~~

### hibernate.cfg.xml

Hibernate config (아래의 `hibernate.cfg.xml` 안에)은 데이터베이스에 연결하는 방법을 지정합니다. SSL을 설정하고 보안 인증서의 위치를 지정하는 [연결 URL](connection-parameters.html#connect-using-a-url)을 기록해 두십시오.

{% include copy-clipboard.html %}
~~~ xml
{% include {{page.version.version}}/app/hibernate-basic-sample/hibernate.cfg.xml %}
~~~

### build.gradle

Gradle 빌드 파일은 종속성(이 경우에는 Postgres JDBC driver와 Hibernate)을 명시합니다:

{% include copy-clipboard.html %}
~~~ groovy
{% include {{page.version.version}}/app/hibernate-basic-sample/build.gradle %}
~~~

</section>

<section class="filter-content" markdown="1" data-scope="insecure">

## 2단계. `maxroach` 사용자와 `bank` 데이터베이스 생성하기

{% include {{page.version.version}}/app/insecure/create-maxroach-user-and-bank-database.md %}

## 3단계. Java 코드 

다음의 파일들을 포함하는 Java 프로젝트를 다운로드 및 추출 [hibernate-basic-sample.tgz](https://github.com/cockroachdb/docs/raw/master/_includes/v2.1/app/insecure/hibernate-basic-sample/hibernate-basic-sample.tgz):

파일 | 설명
-----|------------
[`Sample.java`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/insecure/hibernate-basic-sample/Sample.java) | [Hibernate](http://hibernate.org/orm/)를 사용하여 Java 객체 상태를 SQL operation에 매핑합니다. 더 많은 정보는 [Sample.java](#sample-java)를 참조하십시오.
[`hibernate.cfg.xml`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/insecure/hibernate-basic-sample/hibernate.cfg.xml) | 데이터베이스에 연결하는 방법을 지정하고, 어플리케이션을 실행할 때마다 데이터베이스 스키마를 삭제하고 다시 만들도록 합니다. 더 많은 정보는 [hibernate.cfg.xml](#hibernate-cfg-xml)를 참조하십시오.
[`build.gradle`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/insecure/hibernate-basic-sample/build.gradle) | 어플리케이션을 빌드하고 실행하는데 사용됩니다. 더 많은 정보는 [build.gradle](#build-gradle)을 참조하십시오.

`hibernate-basic-sample` 디렉토리에서 어플리케이션을 빌드하고 실행시키시오:

{% include copy-clipboard.html %}
~~~ shell
$ gradle run
~~~

출력의 끝 부분은 다음과 같이 보여아합니다:

~~~
1 1000
2 250
~~~

표와 행이 성공적으로 생성되었는지 확인하려면, [built-in SQL client](use-the-built-in-sql-client.html)을 시작하십시오:

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
|  1 |    1000 |
|  2 |     250 |
+----+---------+
(2 rows)
~~~

### Sample.java

아래의 Java 코드는 [Hibernate ORM](http://hibernate.org/orm/)을 사용하여 Java 객체를 SQL operation에 매핑합니다. 특히, 이 코드는:

- `Account` 클래스를 바탕으로 데이터베이스에 `accounts` 표를 생성합니다.

- `session.save(new Account())`을 사용하여 표에 행을 삽입합니다.

- `CriteriaQuery<Account> query` 객체를 사용하여 잔액을 출력할 수 있도록 표에서 선택할 SQL 쿼리를 정합니다.

{% include copy-clipboard.html %}
~~~ java
{% include {{page.version.version}}/app/insecure/hibernate-basic-sample/Sample.java %}
~~~

### hibernate.cfg.xml

Hibernate config (아래의 `hibernate.cfg.xml` 안에)은 데이터베이스에 연결하는 방법을 지정합니다. SSL을 설정하고 보안 인증서의 위치를 지정하는 [연결 URL](connection-parameters.html#connect-using-a-url)을 기록해 두십시오.

{% include copy-clipboard.html %}
~~~ xml
{% include {{page.version.version}}/app/insecure/hibernate-basic-sample/hibernate.cfg.xml %}
~~~

### build.gradle

Gradle 빌드 파일은 종속성(이 경우에는 Postgres JDBC driver와 Hibernate)을 명시합니다:

{% include copy-clipboard.html %}
~~~ groovy
{% include {{page.version.version}}/app/insecure/hibernate-basic-sample/build.gradle %}
~~~

</section>

## 더 보기

Hibernate ORM](http://hibernate.org/orm/)사용에 대해 자세히 알아보거나, [`examples-orms`](https://github.com/cockroachdb/examples-orms) repository에서 CockroachDB를 이용한 Hibernate의 보다 현실적인 구현을 확인하세요.

{% include {{page.version.version}}/app/see-also-links.md %}
