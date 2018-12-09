---
title: Build a PHP App with CockroachDB
summary: Learn how to use CockroachDB from a simple PHP application with a low-level client driver.
toc: true
twitter: false
---

이 튜토리얼에서는 PostgreSQL과 호환되는 드라이버를 사용하여 CockroachDB로 간단한 PHP 어플리케이션을 제작하는 방법을 보여줍니다.

[php-pgsql driver](http://php.net/manual/en/book.pgsql.php)는 **베타-레벨** 지원을 요청할 수 있을 정도로 테스트 되었고, 여기에 사용되었습니다. 만약 문제가 발생할 경우 상세 설명과 함께 [이슈 열기](https://github.com/cockroachdb/cockroach/issues/new)를 하여 저희가 전체를 지원할 수 있도록 도와주시길 부탁드립니다.

## 시작하기 전에

{% include {{page.version.version}}/app/before-you-begin.md %}

## 1단계. php-pgsql driver 설치하기

php-pgsql driver를[공식 설명서](http://php.net/manual/en/book.pgsql.php)에 설명된 대로 설치하시오.

<section class="filter-content" markdown="1" data-scope="secure">

## 2단계. `maxroach` 사용자와 `bank` 데이터베이스 생성하기

{% include {{page.version.version}}/app/create-maxroach-user-and-bank-database.md %}

## 3단계. `maxroach` 사용자에 대한 인증서 생성하기

다음 명령을 실행하여 `maxroach` 사용자에 대한 인증서와 키를 생성하시오. 코드 샘플은 이 사용자로 실행됩니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach cert create-client maxroach --certs-dir=certs --ca-key=my-safe-directory/ca.key
~~~

## 4단계. PHP 코드 실행하기

이제 데이터베이스와 사용자가 있으므로 코드를 실행하여 표를 만들고 행을 삽입한 다음, 코드를 실행하여 원자성 [트랜잭션](transactions.html)으로 값을 읽고 업데이트합니다.

### 기초적인 명령문

첫째로, 다음 코드를 이용하여 `maxroach` 사용자에 연결하고, 몇 가지 기본 SQL 명령문을 실행하고, 표를 생성하고, 행을 삽입하고, 행을 읽고 인쇄합니다.

<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/basic-sample.php" download><code>basic-sample.php</code></a> 파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{% include copy-clipboard.html %}
~~~ php
{% include {{ page.version.version }}/app/basic-sample.php %}
~~~

출력은 다음과 같아야 합니다:

~~~ shell
Account balances:
1: 1000
2: 250
~~~

### 트랜잭션 (재시도 논리 사용)

다음으로, 다음 코드를 이용하여 다시 `maxroach` 사용자로 연결하지만, 이번에는 포함된 모든 명령문이 커밋되거나 중단되는 한 계좌에서 다른 계좌로 자금을 이전하기 위해 원자성 트랜잭션으로 명령문 그룹을 실행할 것입니다.

<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/txn-sample.php" download><code>txn-sample.php</code></a> 파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{{site.data.alerts.callout_info}}
기본 `SERIALIZABLE` 격리 수준을 사용하면, CockroachDB는 읽기/쓰기 경합 시 [클라이언트가 트랜잭션을 다시 시도하기](transactions.html#transaction-retries)를 요구할 수 있습니다. CockroachDB는 트랜잭션 내에서 실행되고 필요에 따라 재시도하는 일반적인 **재시도 함수**를 제공합니다. 여기서 재시도 함수를 복사하여 코드에 붙여 넣을 수 있습니다.
{{site.data.alerts.end}}

{% include copy-clipboard.html %}
~~~ php
{% include {{ page.version.version }}/app/txn-sample.php %}
~~~

출력은 다음과 같아야 합니다:

~~~ shell
Account balances after transfer:
1: 900
2: 350
~~~

한 계좌에서 다른 계좌로 자금이 이전되었는지 확인하려면 [built-in SQL client](use-the-built-in-sql-client.html)을 시작하시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --certs-dir=certs -e 'SELECT id, balance FROM accounts' --database=bank
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

## 3단계. PHP 코드 실행하기

이제 데이터베이스와 사용자가 있으므로 코드를 실행하여 표를 만들고 행을 삽입한 다음, 코드를 실행하여 원자성 [트랜잭션](transactions.html)으로 값을 읽고 업데이트합니다.

### 기초적인 명령문

첫째로, 다음 코드를 이용하여 `maxroach` 사용자에 연결하고, 몇 가지 기본 SQL 명령문을 실행하고, 표를 생성하고, 행을 삽입하고, 행을 읽고 인쇄합니다.

<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/insecure/basic-sample.php" download><code>basic-sample.php</code></a> 파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{% include copy-clipboard.html %}
~~~ php
{% include {{ page.version.version }}/app/insecure/basic-sample.php %}
~~~

출력은 다음과 같아야 합니다:

~~~ shell
Account balances:
1: 1000
2: 250
~~~

### 트랜잭션 (재시도 논리 사용)

다음으로, 다음 코드를 이용하여 다시 `maxroach` 사용자로 연결하지만, 이번에는 포함된 모든 명령문이 커밋되거나 중단되는 한 계좌에서 다른 계좌로 자금을 이전하기 위해 원자성 트랜잭션으로 명령문 그룹을 실행할 것입니다.

<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/insecure/txn-sample.php" download><code>txn-sample.php</code></a> 파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{{site.data.alerts.callout_info}}
기본 `SERIALIZABLE` 격리 수준을 사용하면, CockroachDB는 읽기/쓰기 경합 시 [클라이언트가 트랜잭션을 다시 시도하기](transactions.html#transaction-retries)를 요구할 수 있습니다. CockroachDB는 트랜잭션 내에서 실행되고 필요에 따라 재시도하는 일반적인 **재시도 함수**를 제공합니다. 여기서 재시도 함수를 복사하여 코드에 붙여 넣을 수 있습니다.
{{site.data.alerts.end}}

{% include copy-clipboard.html %}
~~~ php
{% include {{ page.version.version }}/app/insecure/txn-sample.php %}
~~~

출력은 다음과 같아야 합니다:

~~~ shell
Account balances after transfer:
1: 900
2: 350
~~~

한 계좌에서 다른 계좌로 자금이 이전되었는지 확인하려면 [built-in SQL client](use-the-built-in-sql-client.html)을 시작하시오:

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

</section>

## 더 보기

[php-pgsql driver](http://php.net/manual/en/book.pgsql.php) 사용에 대해 자세히 알아보세요.

{% include {{ page.version.version }}/app/see-also-links.md %}
