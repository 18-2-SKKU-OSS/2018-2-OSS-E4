---
title: Build a Node.js App with CockroachDB
summary: Learn how to use CockroachDB from a simple Node.js application with the Node.js pg driver.
toc: true
twitter: false
---

<div class="filters filters-big clearfix">
    <a href="build-a-nodejs-app-with-cockroachdb.html"><button class="filter-button current">Use <strong>pg</strong></button></a>
    <a href="build-a-nodejs-app-with-cockroachdb-sequelize.html"><button class="filter-button">Use <strong>Sequelize</strong></button></a>
</div>

이 튜토리얼에서는 PostgreSQL과 호환되는 드라이버나 ORM을 사용하여 CockroachDB로 간단한 Node.js 어플리케이션을 제작하는 방법을 보여줍니다.

[Node.js pg driver](https://www.npmjs.com/package/pg)와 [Sequelize ORM](https://sequelize.readthedocs.io/en/v3/)는 **베타-레벨** 지원을 요청할 수 있을 정도로 테스트 되었고, 여기에 사용되었습니다. 만약 문제가 발생할 경우 상세 설명과 함께 [이슈 열기](https://github.com/cockroachdb/cockroach/issues/new)를 하여 저희가 전체를 지원할 수 있도록 도와주시길 부탁드립니다.


## 시작하기 전에

{% include {{page.version.version}}/app/before-you-begin.md %}

## Step 1. Node.js 패키지 설치하기

어플리케이션이 CockroachDB와 커뮤니케이션을 할 수 있도록 하려면, [Node.js pg driver](https://www.npmjs.com/package/pg)를 설치하시오:

{% include copy-clipboard.html %}
~~~ shell
$ npm install pg
~~~

이 페이지의 예시 앱은 [`async`](https://www.npmjs.com/package/async) 또한 요구합니다:

{% include copy-clipboard.html %}
~~~ shell
$ npm install async
~~~

<section class="filter-content" markdown="1" data-scope="secure">

## 2단계. `maxroach` 사용자와 `bank` 데이터베이스 생성하기

{% include {{page.version.version}}/app/create-maxroach-user-and-bank-database.md %}

## 3단계. `maxroach` 사용자에 대한 인증서 생성하기

다음 명령을 실행하여 `maxroach` 사용자에 대한 인증서와 키를 생성하시오. 코드 샘플은 이 사용자로 실행됩니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach cert create-client maxroach --certs-dir=certs --ca-key=my-safe-directory/ca.key
~~~

## 4단계. Node.js 코드 실행하기

이제 데이터베이스와 사용자가 있으므로 코드를 실행하여 표를 만들고 행을 삽입한 다음, 코드를 실행하여 원자성 [transaction](transactions.html)으로 값을 읽고 업데이트합니다.

### 기초적인 명령문

첫째로, 다음 코드를 이용하여 `maxroach` 사용자에 연결하고, 몇 가지 기본 SQL 명령문을 실행하고, 표를 생성하고, 행을 삽입하고, 행을 읽고 인쇄합니다.

[`basic-sample.js`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/basic-sample.js)  파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{% include copy-clipboard.html %}
~~~ js
{% include {{page.version.version}}/app/basic-sample.js %}
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ node basic-sample.js
~~~

출력은 다음과 같아야 합니다:

~~~
Initial balances:
{ id: '1', balance: '1000' }
{ id: '2', balance: '250' }
~~~

### 트랜잭션 (재시도 논리 사용)

다음으로, 다음 코드를 이용하여 다시 `maxroach` 사용자로 연결하지만, 이번에는 포함된 모든 명령문이 커밋되거나 중단되는 한 계좌에서 다른 계좌로 자금을 이전하기 위해 원자성 트랜잭션으로 명령문 그룹을 실행할 것입니다.

[`txn-sample.js`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/txn-sample.js) 파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{% include v2.1/client-transaction-retry.md %}

{% include copy-clipboard.html %}
~~~ js
{% include {{page.version.version}}/app/txn-sample.js %}
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ node txn-sample.js
~~~

출력은 다음과 같아야 합니다:

~~~
Balances after transfer:
{ id: '1', balance: '900' }
{ id: '2', balance: '350' }
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

## 3단계. Node.js 코드 실행하기

이제 데이터베이스와 사용자가 있으므로 코드를 실행하여 표를 만들고 행을 삽입한 다음, 코드를 실행하여 원자성 [transaction](transactions.html)으로 값을 읽고 업데이트합니다.

### 기초적인 명령문

첫째로, 다음 코드를 이용하여 `maxroach` 사용자에 연결하고, 몇 가지 기본 SQL 명령문을 실행하고, 표를 생성하고, 행을 삽입하고, 행을 읽고 인쇄합니다.

[`basic-sample.js`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/insecure/basic-sample.js) 파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{% include copy-clipboard.html %}
~~~ js
{% include {{page.version.version}}/app/insecure/basic-sample.js %}
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ node basic-sample.js
~~~

출력은 다음과 같아야 합니다:

~~~
Initial balances:
{ id: '1', balance: '1000' }
{ id: '2', balance: '250' }
~~~

### 트랜잭션 (재시도 논리 사용)

다음으로, 다음 코드를 이용하여 다시 `maxroach` 사용자로 연결하지만, 이번에는 포함된 모든 명령문이 커밋되거나 중단되는 한 계좌에서 다른 계좌로 자금을 이전하기 위해 원자성 트랜잭션으로 명령문 그룹을 실행할 것입니다.

[`txn-sample.js`](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/insecure/txn-sample.js) 파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{% include v2.1/client-transaction-retry.md %}

{% include copy-clipboard.html %}
~~~ js
{% include {{page.version.version}}/app/insecure/txn-sample.js %}
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ node txn-sample.js
~~~

출력은 다음과 같아야 합니다:

~~~
Balances after transfer:
{ id: '1', balance: '900' }
{ id: '2', balance: '350' }
~~~

한 계좌에서 다른 계좌로 자금이 이전되었는지 확인하려면, [built-in SQL client](use-the-built-in-sql-client.html)을 시작하시오:

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

[Node.js pg driver](https://www.npmjs.com/package/pg) 사용에 대해 자세히 알아보세요.

{% include {{page.version.version}}/app/see-also-links.md %}
