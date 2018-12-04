---
title: Build a Go App with CockroachDB
summary: Learn how to use CockroachDB from a simple Go application with the Go pq driver.
toc: true
twitter: false
---

<div class="filters filters-big clearfix">
    <a href="build-a-go-app-with-cockroachdb.html"><button class="filter-button current">Use <strong>pq</strong></button></a>
    <a href="build-a-go-app-with-cockroachdb-gorm.html"><button class="filter-button">Use <strong>GORM</strong></button></a>
</div>

이 튜토리얼에서는 PostgreSQL과 호환되는 드라이버나 ORM을 사용하여 CockroachDB로 간단한 Go 어플리케이션을 제작하는 방법을 보여줍니다.

[Go pq driver](https://godoc.org/github.com/lib/pq)와 [GORM ORM](http://gorm.io)는 **베타-레벨** 지원을 요청할 수 있을 정도로 테스트 되었고, 여기에 사용되었습니다. 만약 문제가 발생할 경우 상세 설명과 함께 [이슈 열기](https://github.com/cockroachdb/cockroach/issues/new)를 하여 저희가 전체를 지원할 수 있도로 도와주시길 부탁드립니다.

## 시작하기 전에

{% include {{page.version.version}}/app/before-you-begin.md %}

## 1단계. Go pq driver 설치하기

[Go pq driver](https://godoc.org/github.com/lib/pq)를 설치하려면, 다음의 명령을 실행시키시오:

{% include copy-clipboard.html %}
~~~ shell
$ go get -u github.com/lib/pq
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

## 4단계. Go 코드 실행하기

이제 데이터베이스와 사용자가 있으므로 코드를 실행하여 표를 만들고 행을 삽입한 다음, 코드를 실행하여 원자성 [트랜잭션](transactions.html)으로 값을 읽고 업데이트합니다.

### 기초적인 명령문

첫째로, 다음 코드를 이용하여 `maxroach` 사용자에 연결하고, 몇 가지 기본 SQL 명령문을 실행하고, 표를 생성하고, 행을 삽입하고, 행을 읽고 인쇄합니다.

<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/basic-sample.go" download><code>basic-sample.go</code></a> 파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{% include copy-clipboard.html %}
~~~ go
{% include {{ page.version.version }}/app/basic-sample.go %}
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ go run basic-sample.go
~~~

출력은 다음과 같아야 합니다:

~~~ shell
Initial balances:
1 1000
2 250
~~~

### 트랜잭션 (재시도 논리 사용)

다음으로, 다음 코드를 이용하여 다시 `maxroach` 사용자로 연결하지만, 이번에는 포함된 모든 명령문이 커밋되거나 중단되는 한 계좌에서 다른 계좌로 자금을 이전하기 위해 원자성 트랜잭션으로 명령문 그룹을 실행할 것입니다.

<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/txn-sample.go" download><code>txn-sample.go</code></a> 파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{% include copy-clipboard.html %}
~~~ go
{% include {{ page.version.version }}/app/txn-sample.go %}
~~~
기본 `SERIALIZABLE` 격리 수준을 사용하면, CockroachDB는 읽기/쓰기 경합 시 [클라이언트가 트랜잭션을 다시 시도하기](transactions.html#transaction-retries)를 요구할 수 있습니다. CockroachDB는 트랜잭션 내에서 실행되고 필요에 따라 재시도하는 일반적인 **재시도 함수**를 제공합니다. Go의 경우, CockroachDB 재시도 함수는 CockroachDB Go 클라이언트의 `crdb`패키지에 있습니다. 설치하려면 다음과 같이 라이브러리를 `$GOPATH`에 복제하시오:

{% include copy-clipboard.html %}
~~~ shell
$ mkdir -p $GOPATH/src/github.com/cockroachdb
~~~

{% include copy-clipboard.html %}
~~~ shell
$ cd $GOPATH/src/github.com/cockroachdb
~~~

{% include copy-clipboard.html %}
~~~ shell
$ git clone git@github.com:cockroachdb/cockroach-go.git
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ go run txn-sample.go
~~~

출력은 다음과 같아야 합니다:

~~~ shell
Success
~~~

한 계좌에서 다른 계좌로 자금이 이전되었는지 확인하려면, [built-in SQL client](use-the-built-in-sql-client.html)을 시작하시오:

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

## 3단계. Go 코드 실행하기

이제 데이터베이스와 사용자가 있으므로 코드를 실행하여 표를 만들고 행을 삽입한 다음, 코드를 실행하여 원자성 [트랜잭션](transactions.html)으로 값을 읽고 업데이트합니다.

### 기초적인 명령문

첫째로, 다음 코드를 이용하여 `maxroach` 사용자에 연결하고, 몇 가지 기본 SQL 명령문을 실행하고, 표를 생성하고, 행을 삽입하고, 행을 읽고 인쇄합니다.

<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/insecure/basic-sample.go" download><code>basic-sample.go</code></a> 파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{% include copy-clipboard.html %}
~~~ go
{% include {{ page.version.version }}/app/insecure/basic-sample.go %}
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ go run basic-sample.go
~~~

출력은 다음과 같아야 합니다:

~~~ shell
Initial balances:
1 1000
2 250
~~~

### 트랜잭션 (재시도 논리 사용)

다음으로, 다음 코드를 이용하여 다시 `maxroach` 사용자로 연결하지만, 이번에는 포함된 모든 명령문이 커밋되거나 중단되는 한 계좌에서 다른 계좌로 자금을 이전하기 위해 원자성 트랜잭션으로 명령문 그룹을 실행할 것입니다.

<a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/app/insecure/txn-sample.go" download><code>txn-sample.go</code></a> 파일을 다운로드하거나 직접 파일을 만들고 코드를 파일에다 복사합니다.

{% include copy-clipboard.html %}
~~~ go
{% include {{ page.version.version }}/app/insecure/txn-sample.go %}
~~~

CockroachDB는 읽기/쓰기 경합 시 [클라이언트가 트랜잭션을 다시 시도하기](transactions.html#transaction-retries)를 요구할 수 있습니다. CockroachDB는 트랜잭션 내에서 실행되고 필요에 따라 재시도하는 일반적인 **재시도 함수**를 제공합니다. Go의 경우, CockroachDB 재시도 함수는 CockroachDB Go 클라이언트의 `crdb` 패키지에 있습니다.

[CockroachDB Go client](https://github.com/cockroachdb/cockroach-go)를 설치하려면, 다음 명령을 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ go get -d github.com/cockroachdb/cockroach-go
~~~

그리고 코드를 실행하시오:

{% include copy-clipboard.html %}
~~~ shell
$ go run txn-sample.go
~~~

출력은 다음과 같아야 합니다:

~~~ shell
Success
~~~

한 계좌에서 다른 계좌로 자금이 이전되었는지 확인하려면, [built-in SQL client](use-the-built-in-sql-client.html)을 :

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

[Go pq driver](https://godoc.org/github.com/lib/pq) 사용에 대해 자세히 알아보세요.

{% include {{ page.version.version }}/app/see-also-links.md %}
