---
title: Build a Python App with CockroachDB
summary: Learn how to use CockroachDB from a simple Python application with the SQLAlchemy ORM.
toc: true
twitter: false
---

<div class="filters filters-big clearfix">
    <a href="build-a-python-app-with-cockroachdb.html"><button style="width: 28%" class="filter-button">Use <strong>psycopg2</strong></button></a>
    <a href="build-a-python-app-with-cockroachdb-sqlalchemy.html"><button style="width: 28%" class="filter-button current">Use <strong>SQLAlchemy</strong></button></a>
</div>

이 튜토리얼에서는 PostgreSQL과 호환되는 드라이버나 ORM을 사용하여 CockroachDB로 간단한 Python 어플리케이션을 제작하는 방법을 보여줍니다.

[Python psycopg2 driver](http://initd.org/psycopg/docs/)와 [SQLAlchemy ORM](https://docs.sqlalchemy.org/en/latest/)는 **베타-레벨** 지원을 요청할 수 있을 정도로 테스트 되었고, 여기에 사용되었습니다. 만약 문제가 발생할 경우 상세 설명과 함께 [이슈 열기](https://github.com/cockroachdb/cockroach/issues/new)를 하여 저희가 전체를 지원할 수 있도록 도와주시길 부탁드립니다.

{{site.data.alerts.callout_success}}
SQLAlchemy을 CockroachDB와 함께 보다 현실적으로 사용하려면, [`examples-orms`](https://github.com/cockroachdb/examples-orms) repository의 예시를 참조하세요.
{{site.data.alerts.end}}

## 시작하기 전에

{% include {{page.version.version}}/app/before-you-begin.md %}

{{site.data.alerts.callout_danger}}

**CockroachDB 2.0 을 2.1로 업그레이드?** 2.0 클러스터에서 SQLAlchemy를 사용한 경우 CockroachDB 2.1로 업그레이드하기 전에 [어댑터를 최신 버전으로 업그레이드](https://github.com/cockroachdb/cockroachdb-python)해야 합니다.

{{site.data.alerts.end}}

## 1단계. SQLAlchemy ORM 설치하기

CockroachDB와 PostgreSQL 간의 사소한 차이점을 설명하는 [CockroachDB Python 패키지](https://github.com/cockroachdb/cockroachdb-python)와 SQLAlchemy을 설치하려면, 다음의 명령을 실행시키시오:

{% include copy-clipboard.html %}
~~~ shell
$ pip install sqlalchemy cockroachdb
~~~

SQLAlchemy를 설치하는 다른 방법에 대해서는 [공식 설명서](http://docs.sqlalchemy.org/en/latest/intro.html#installation-guide)를 참조하십시오.

<section class="filter-content" markdown="1" data-scope="secure">

## 2단계. `maxroach` 사용자와 `bank` 데이터베이스 생성하기

{% include {{page.version.version}}/app/create-maxroach-user-and-bank-database.md %}

## 3단계. `maxroach` 사용자에 대한 인증서 생성하기

다음 명령을 실행하여 `maxroach` 사용자에 대한 인증서와 키를 생성하시오. 코드 샘플은 이 사용자로 실행됩니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach cert create-client maxroach --certs-dir=certs --ca-key=my-safe-directory/ca.key
~~~

## 3단계. Python 코드 실행하기

다음 코드는 [SQLAlchemy ORM](https://docs.sqlalchemy.org/en/latest/)을 사용하여 Python-specific 개체들을 SQL operation에 매핑합니다. 특히 `Base.metadata.create_all(engine)`는 계좌 모델을 바탕으로 `accounts` 표를 생성하고, `session.add_all([Account(),...
])`는 표에 행을 삽입하며, `session.query(Account)`는 잔액을 프린트할 수 있도록 표에서 선택을 합니다.

{{site.data.alerts.callout_info}}
앞에서 설치된 <a href="https://github.com/cockroachdb/cockroachdb-python">CockroachDB Python 패키지</a>는 엔진 URL의 <code>cockroachdb://</code> 접두사에 의해 실행됩니다. <code>postgres://</code>를 사용하여 클러스터에 연결하면 작동하지 않습니다.
{{site.data.alerts.end}}

코드를 복사하거나
<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/sqlalchemy-basic-sample.py" download>직접 다운로드 받으시오</a>

{% include copy-clipboard.html %}
~~~ python
{% include {{page.version.version}}/app/sqlalchemy-basic-sample.py %}
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ python sqlalchemy-basic-sample.py
~~~

출력은 다음과 같아야 합니다:

~~~ shell
1 1000
2 250
~~~

표와 행들이 제대로 생성되었는지 확인하려면, [built-in SQL client](use-the-built-in-sql-client.html)을 시작하시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --certs-dir=certs --database=bank
~~~

다음 명령문을 실행하십시오:

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

</section>

<section class="filter-content" markdown="1" data-scope="insecure">

## 2단계. `maxroach` 사용자와 `bank` 데이터베이스 생성하기

{% include {{page.version.version}}/app/insecure/create-maxroach-user-and-bank-database.md %}

## 3단계. Python 코드 실행하기

다음 코드는 [SQLAlchemy ORM](https://docs.sqlalchemy.org/en/latest/)을 사용하여 Python-specific 개체들을 SQL operation에 매핑합니다. 특히 `Base.metadata.create_all(engine)`는 계좌 모델을 바탕으로 `accounts` 표를 생성하고, `session.add_all([Account(),...
])`는 표에 행을 삽입하며, `session.query(Account)`는 잔액을 프린트할 수 있도록 표에서 선택을 합니다.

{{site.data.alerts.callout_info}}
앞에서 설치된 <a href="https://github.com/cockroachdb/cockroachdb-python">CockroachDB Python 패키지</a>는 엔진 URL의 <code>cockroachdb://</code> 접두사에 의해 실행됩니다. <code>postgres://</code>를 사용하여 클러스터에 연결하면 작동하지 않습니다.
{{site.data.alerts.end}}

코드를 복사하거나
<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/v2.1/app/insecure/sqlalchemy-basic-sample.py" download>직접 다운로드 받으시오</a>

{% include copy-clipboard.html %}
~~~ python
{% include {{page.version.version}}/app/insecure/sqlalchemy-basic-sample.py %}
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ python sqlalchemy-basic-sample.py
~~~

출력은 다음과 같아야 합니다:

~~~ shell
1 1000
2 250
~~~

표와 행들이 제대로 생성되었는지 확인하려면, [built-in SQL client](use-the-built-in-sql-client.html)을 시작하시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure --database=bank
~~~

다음 명령문을 실행하십시오:

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

</section>

## 다음은?

[SQLAlchemy ORM](https://docs.sqlalchemy.org/en/latest/) 사용에 대해 자세히 알아보거나, [`examples-orms`](https://github.com/cockroachdb/examples-orms) repository에서 CockroachDB를 이용한 SQLAlchemy의 보다 현실적인 구현을 확인하세요.

{% include {{page.version.version}}/app/see-also-links.md %}
