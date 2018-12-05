---
title: Build a Node.js App with CockroachDB
summary: Learn how to use CockroachDB from a simple Node.js application with the Sequelize ORM.
toc: true
twitter: false
---

<div class="filters filters-big clearfix">
    <a href="build-a-nodejs-app-with-cockroachdb.html"><button class="filter-button">Use <strong>pg</strong></button></a>
    <a href="build-a-nodejs-app-with-cockroachdb-sequelize.html"><button class="filter-button current">Use <strong>Sequelize</strong></button></a>
</div>

이 튜토리얼에서는 PostgreSQL과 호환되는 드라이버나 ORM을 사용하여 CockroachDB로 간단한 Node.js 어플리케이션을 제작하는 방법을 보여줍니다.

[Node.js pg driver](https://www.npmjs.com/package/pg)와 [Sequelize ORM](https://sequelize.readthedocs.io/en/v3/)는 **베타-레벨** 지원을 요청할 수 있을 정도로 테스트 되었고, 여기에 사용되었습니다. 만약 문제가 발생할 경우 상세 설명과 함께 [이슈 열기](https://github.com/cockroachdb/cockroach/issues/new)를 하여 저희가 전체를 지원할 수 있도록 도와주시길 부탁드립니다.

{{site.data.alerts.callout_success}}
Sequelize를 CockroachDB와 함께 보다 현실적으로 사용하려면, [`examples-orms`](https://github.com/cockroachdb/examples-orms)repository의 예시를 참조하세요.
{{site.data.alerts.end}}


## 시작하기 전에

{% include {{page.version.version}}/app/before-you-begin.md %}

## 1단계. Sequelize ORM 설치하기

CockroachDB와 PostgreSQL의 일부 사소한 차이를 설명하는 [CockroachDB Node.js package](https://github.com/cockroachdb/sequelize-cockroachdb)와 함께 Sequelize를 설치하려면, 다음의 명령을 실행시키시오:

{% include copy-clipboard.html %}
~~~ shell
$ npm install sequelize sequelize-cockroachdb
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

다음 코드는 [Sequelize](https://sequelize.readthedocs.io/en/v3/) ORM을 사용하여 Node.js-specific 개체들을 SQL operations에 매핑합니다. 특히, `Account.sync({force: true})`는 계좌 모델을 바탕으로 `accounts`표를 생성하고(또는 표가 이미 존재하는 경우 삭제하고 다시 생성합니다), `Account.bulkCreate([...])`는 표에 행을 삽입하며, `Account.findAll()`는 잔액을 프린트할 수 있도록 표에서 선택을 합니다.

코드를 복사하거나
<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/sequelize-basic-sample.js" download>직접 다운로드 받으시오</a>.

{% include copy-clipboard.html %}
~~~ js
{% include {{ page.version.version }}/app/sequelize-basic-sample.js %}
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ node sequelize-basic-sample.js
~~~

출력은 다음과 같아야 합니다:

~~~ shell
1 1000
2 250
~~~

한 계좌에서 다른 계좌로 자금이 이전되었는지 확인하려면, [built-in SQL client](use-the-built-in-sql-client.html)을 시작하시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --certs-dir=/tmp/certs -e 'SELECT id, balance FROM accounts' --database=bank
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

</section>

</section>

<section class="filter-content" markdown="1" data-scope="insecure">

## 2단계. `maxroach` 사용자와 `bank` 데이터베이스 생성하기

{% include {{page.version.version}}/app/insecure/create-maxroach-user-and-bank-database.md %}

## 3계. Node.js 코드 실행하기

다음 코드는 Sequelize](https://sequelize.readthedocs.io/en/v3/) ORM을 사용하여 Node.js-specific 개체들을 SQL operations에 매핑합니다. 특히,  `Account.sync({force: true})`는 계좌 모델을 바탕으로 `accounts`표를 생성하고(또는 테이블이 이미 존재하는 경우 삭제하고 다시 생성합니다),  `Account.bulkCreate([...])`는 표에 행을 삽입하며, `Account.findAll()` 는 잔액을 프린트할 수 있도록 표에서 선택을 합니다.


코드를 복사하거나
<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/insecure/sequelize-basic-sample.js" download>직접 다운로드 받으시오</a>.

{% include copy-clipboard.html %}
~~~ js
{% include {{ page.version.version }}/app/insecure/sequelize-basic-sample.js %}
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ node sequelize-basic-sample.js
~~~

출력은 다음과 같아야 합니다:

~~~ shell
1 1000
2 250
~~~

표와 행이 성공적으로 생성되었는지 확인하려면, [built-in SQL client](use-the-built-in-sql-client.html)를 다시 사용하시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure -e 'SHOW TABLES' --database=bank
~~~

~~~
+------------+
| table_name |
+------------+
| accounts   |
+------------+
(1 row)
~~~

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure -e 'SELECT id, balance FROM accounts' --database=bank
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

</section>

## 더 보기

[Sequelize ORM](https://sequelize.readthedocs.io/en/v3/) 사용에 대해 자세히 알아보거나, [`examples-orms`](https://github.com/cockroachdb/examples-orms) repository에서 CockroachDB를 이용한 Sequelize의 보다 현실적인 구현을 확인하세요.
{% include {{ page.version.version }}/app/see-also-links.md %}
