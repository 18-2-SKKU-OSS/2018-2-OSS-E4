---
title: JSON Support
summary: Use a local cluster to explore how CockroachDB can store and query unstructured JSONB data.
toc: true
---

이 페이지에서는 [역 인덱스](inverted-indexes.html)가 당신의 쿼리를 최적화하는 방법 뿐만 아니라 CockroachDB가 타사 API의 구조화되지 않은 [`JSONB`](jsonb.html) 데이터를 저장하고 쿼리하는 방법에 대한 간단한 설명을 보여줍니다. 


## 1단계. 필수 구성 요소 설치

<div class="filters filters-big clearfix">	<div class="filters filters-big clearfix">
    <button class="filter-button" data-scope="go">Go</button>	   
    <button class="filter-button" data-scope="python">Python</button>
</div>
    
- 최신 버전의 [CockroachDB](install-cockroachdb.html)를 설치하십시오. 
- [Go](https://golang.org/dl/)의 최신 버전을 설치하십시오 : `brew install go`
- [PostgreSQL 드라이버](https://github.com/lib/pq)를 설치하십시오 : `go get github.com / lib / pq`
</div>

- 최신 버전의 [CockroachDB](install-cockroachdb.html)를 설치하십시오.
- [Python psycopg2 드라이버](http://initd.org/psycopg/docs/install.html)를 설치하십시오 : `pip install psycopg2`
- [Python 리퀘스트 라이브러리](http://docs.python-requests.org/en/master/) 설치하십시오 : `pip install requests`
</div>

## 2단계. 단일 노드 클러스터 시작

이 튜토리얼의 목적을 위해, 비보안 모드에서 실행되는 CockroachDB 단 하나의 노드가 필요합니다:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=json-test \
--listen-addr=localhost:26257 \
--http-addr=localhost:8080
~~~

## 3단계. 사용자 생성

새 터미널에서 `root` 사용자로 [`cockroach user`](create-and-manage-users.html) 명령을 사용하여 새로운 사용자인 `maxroach`를 생성합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach user set maxroach --insecure --host=localhost:26257
~~~

## 4단계. 데이터베이스 생성 및 권한 부여

`root`사용자로서, [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)를 엽니다:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure --host=localhost:26257
~~~

다음으로,`jsonb_test`라는 데이터베이스를 생성하십시오:

{% include copy-clipboard.html %}
~~~ sql
> CREATE DATABASE jsonb_test;
~~~

데이터베이스를 기본값으로 설정하십시오:

{% include copy-clipboard.html %}
~~~ sql
> SET DATABASE = jsonb_test;
~~~

그런 다음 `maxroach`사용자에게 [권한 부여](grant.html)하십시오:

{% include copy-clipboard.html %}
~~~ sql
> GRANT ALL ON DATABASE jsonb_test TO maxroach;
~~~

## 5단계. 테이블 생성

여전히 SQL 쉘에서 `programming`이라는 테이블을 생성합니다:

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE programming (
    id UUID DEFAULT uuid_v4()::UUID PRIMARY KEY,
    posts JSONB
  );
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW CREATE programming;
~~~
~~~
+--------------+-------------------------------------------------+
|    Table     |                   CreateTable                   |
+--------------+-------------------------------------------------+
| programming  | CREATE TABLE programming (                      |
|              |     id UUID NOT NULL DEFAULT uuid_v4()::UUID,   |
|              |     posts JSON NULL,                            |
|              |     CONSTRAINT "primary" PRIMARY KEY (id ASC),  |
|              |     FAMILY "primary" (id, posts)                |
|              | )                                               |
+--------------+-------------------------------------------------+
~~~

## 6단계. 코드 실행

이제 데이터베이스, 사용자 및 테이블이 있으므로 테이블에 행을 삽입하는 코드를 실행해 봅시다.

<div class="filters filters-big clearfix">
    <button class="filter-button" data-scope="go">Go</button>
    <button class="filter-button" data-scope="python">Python</button>
</div>

이 코드는 [/r/programming](https://www.reddit.com/r/programming/)에서 게시물에 대해 [Reddit API](https://www.reddit.com/dev/api/)을 쿼리합니다.
Reddit API는 페이지당 25개의 결과만 반환합니다; 그러나 각 페이지는 다음 페이지를 얻는 방법을 알려주는 `"after"`문자열을 반환합니다. 이제 프로그램은 다음과 같은 작업을 반복합니다.

1. API에 요청합니다.
2. 테이블에 결과를 삽입하고 `"after"`문자열을 가져옵니다.
3. 새로운 `"after"`문자열을 다음 요청의 기초로 사용합니다.

<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/json/json-sample.go" download><code>json-sample.go</code></a> 파일을 다운로드하거나, 직접 파일을 만들고 코드를 복사하십시오:

{% include copy-clipboard.html %}
~~~ go
{% include {{ page.version.version }}/json/json-sample.go %}
~~~

새 터미널 창에서 샘플 코드 파일을 탐색하여 실행하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ go run json-sample.go
~~~
</section>

이 코드는 [/r/programming](https://www.reddit.com/r/programming/)에서 게시물에 대해 [Reddit API](https://www.reddit.com/dev/api/)을 쿼리합니다. Reddit API는 페이지당 25개의 결과만 반환합니다; 그러나 각 페이지는 다음 페이지를 얻는 방법을 알려주는 `"after"`문자열을 반환합니다. 이제 프로그램은 다음과 같은 작업을 반복합니다.

1. API에 요청합니다.
2. `"after"`문자열을 가져옵니다.
3. 결과를 테이블에 삽입합니다.
4. 새로운 `"after"` 문자열을 다음 요청의 기초로 사용합니다.

<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/json/json-sample.go" download><code>json-sample.go</code></a> 파일을 다운로드하거나, 직접 파일을 만들고 코드를 복사하십시오:

{% include copy-clipboard.html %}
~~~ python
{% include {{ page.version.version }}/json/json-sample.py %}
~~~

새 터미널 창에서 샘플 코드 파일을 탐색하여 실행하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ python json-sample.py
~~~
</section>

프로그램을 완료하는 데 시간이 다소 걸리지만, 곧바로 데이터 쿼리를 시작할 수 있습니다.

## 7단계. 데이터 쿼리

SQL쉘이 실행 중인 터미널로 돌아가서 데이터 행이 테이블에 삽입되고 있는지 확인합니다:

{% include copy-clipboard.html %}
~~~ sql
> SELECT count(*) FROM programming;
~~~
~~~
+-------+
| count |
+-------+
|  1120 |
+-------+
~~~

{% include copy-clipboard.html %}
~~~ sql
> SELECT count(*) FROM programming;
~~~
~~~
+-------+
| count |
+-------+
|  2400 |
+-------+
~~~

이제 GitHub 어딘가에 링크가 가리키는 모든 현재 항목을 검색하십시오:

{% include copy-clipboard.html %}
~~~ sql
> SELECT id FROM programming \
WHERE posts @> '{"data": {"domain": "github.com"}}';
~~~
~~~
+--------------------------------------+
|                  id                  |
+--------------------------------------+
| 0036d489-3fe3-46ec-8219-2eaee151af4b |
| 00538c2f-592f-436a-866f-d69b58e842b6 |
| 00aff68c-3867-4dfe-82b3-2a27262d5059 |
| 00cc3d4d-a8dd-4c9a-a732-00ed40e542b0 |
| 00ecd1dd-4d22-4af6-ac1c-1f07f3eba42b |
| 012de443-c7bf-461a-b563-925d34d1f996 |
| 014c0ac8-4b4e-4283-9722-1dd6c780f7a6 |
| 017bfb8b-008e-4df2-90e4-61573e3a3f62 |
| 0271741e-3f2a-4311-b57f-a75e5cc49b61 |
| 02f31c61-66a7-41ba-854e-1ece0736f06b |
| 035f31a1-b695-46be-8b22-469e8e755a50 |
| 03bd9793-7b1b-4f55-8cdd-99d18d6cb3ea |
| 03e0b1b4-42c3-4121-bda9-65bcb22dcf72 |
| 0453bc77-4349-4136-9b02-3a6353ea155e |
...
+--------------------------------------+
(334 rows)

Time: 105.877736ms
~~~

{{site.data.alerts.callout_info}}당신이 실시간 데이터를 쿼리하고 있으므로 이 단계 및 다음 단계의 결과는 이 튜토리얼에 기록된 결과와 다를 수 있습니다.{{site.data.alerts.end}}

## 8단계. 성능 최적화를 위한 역 인덱스 생성

`JSONB` 열에서 필터링하는 쿼리의 성능을 최적화하려면 열에 [역 인덱스](inverted-indexes.html)를 생성하십시오:

{% include copy-clipboard.html %}
~~~ sql
> CREATE INVERTED INDEX ON programming(posts);
~~~

## 9단계. 쿼리 다시 실행

이제 역 인덱스가 있으므로 동일한 쿼리가 훨씬 더 빨리 실행됩니다.

{% include copy-clipboard.html %}
~~~ sql
> SELECT id FROM programming \
WHERE posts @> '{"data": {"domain": "github.com"}}';
~~~
~~~
(334 rows)

Time: 28.646769ms
~~~

이제 쿼리는 105.87736ms 대신 28.646769ms가 소요됩니다.

## 더 알아보기

CockroachDB의 다른 주요 이점 및 특징에 대해 알아보십시오:

{% include {{ page.version.version }}/misc/explore-benefits-see-also.md %}

[`JSONB`](jsonb.html) 데이터 타입과 [역 인덱스](inverted-indexes.html)에 대해 더 알고 싶을 수도 있습니다.
