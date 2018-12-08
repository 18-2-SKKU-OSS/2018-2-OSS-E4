---
title: Build a C# (.NET) App with CockroachDB
summary: Learn how to use CockroachDB from a simple C# (.NET) application with a low-level client driver.
toc: true
twitter: true
---

이 튜토리얼에서는 PostgreSQL과 호환되는 드라이버를 사용하여 CockroachDB로 간단한 C# (.NET) 어플리케이션을 제작하는 방법을 보여줍니다.

[.NET Npgsql driver](http://www.npgsql.org/)는 **베타-레벨** 지원을 요청할 수 있을 정도로 테스트 되었고, 여기에 사용되었습니다. 만약 문제가 발생할 경우 상세 설명과 함께 [이슈 열기](https://github.com/cockroachdb/cockroach/issues/new)를 하여 저희가 전체를 지원할 수 있도록 도와주시길 부탁드립니다.


## 시작하기 전에

[설치된 CockroachDB](install-cockroachdb.html)와 해당 OS용 <a href="https://www.microsoft.com/net/download/" data-proofer-ignore>.NET SDK</a>가 설치되어 있는지 확인하시오.

## 1단계. .NET project 생성하기

{% include copy-clipboard.html %}
~~~ shell
$ dotnet new console -o cockroachdb-test-app
~~~

{% include copy-clipboard.html %}
~~~ shell
$ cd cockroachdb-test-app
~~~

`dotnet` 명령어는 `console` 형식의 새로운 앱을 생성합니다. `-o` 파라미터는 앱이 저장될 `cockroachdb-test-app`라는 디렉터리를 만들어 필요한 파일들로 채웁니다. `cd cockroachdb-test-app` 명령어를 실행하면 새로 생성된 앱 디렉터리에 들어가게 됩니다.

## 2단계. Npgsql driver 설치하기

최신 버전의 [Npgsql driver](https://www.nuget.org/packages/Npgsql/)를 built-in nuget 패키지 매니저를 이용하여 .NET 프로젝트에 설치하시오:

{% include copy-clipboard.html %}
~~~ shell
$ dotnet add package Npgsql
~~~

## 3단계. 싱글-노드 클러스터 시작하기

이 튜토리얼을 위해서, 보안되지 않은 모드에서 실행되는 CockroachDB 노드 하나만 필요합니다:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=hello-1 \
--listen-addr=localhost
~~~

## 4단계. 사용자 생성하기

새로운 터미널에서 `root` 사용자로 [`cockroach user`](create-and-manage-users.html)명령어를 사용하여 새로운 사용자인 `maxroach`를 생성하시오. 

{% include copy-clipboard.html %}
~~~ shell
$ cockroach user set maxroach --insecure
~~~

## 5단계. 데이터베이스 생성 및 권한 부여하기

`root` 사용자로서 [built-in SQL client](use-the-built-in-sql-client.html)를 사용하여 `bank` 데이터베이스를 생성하시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure -e 'CREATE DATABASE bank'
~~~

그리고 `maxroach` 사용자에 [권한 부여하기](grant.html)를 하시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure -e 'GRANT ALL ON DATABASE bank TO maxroach'
~~~

## 6단계. C# 코드 실행하기

이제 데이터베이스와 사용자가 있으므로 코드를 실행하여 표를 만들고 행을 삽입한 다음, 코드를 실행하여 원자성 [트랜잭션](transactions.html)으로 값을 읽고 업데이트합니다.

### 기초적인 명령문

`cockraochdb-test-app/Program.cs`의 내용을 다음 코드로 대체하시오:

{% include copy-clipboard.html %}
~~~ csharp
{% include {{ page.version.version }}/app/basic-sample.cs %}
~~~

그리고 다음 코드를 실행하여 `maxroach` 사용자에 연결하고, 몇 가지 기본 SQL 명령문을 실행하고, 표를 생성하고, 행을 삽입하고, 행을 읽고 인쇄하시오:

{% include copy-clipboard.html %}
~~~ shell
$ dotnet run
~~~

출력은 다음과 같아야 합니다:

~~~
Initial balances:
	account 1: 1000
	account 2: 250
~~~

### 트랜잭션 (재시도 논리 사용)

`cockraochdb-test-app/Program.cs`를 다시 열고 내용을 다음 코드로 대체하시오:

{% include copy-clipboard.html %}
~~~ csharp
{% include {{ page.version.version }}/app/txn-sample.cs %}
~~~

다음으로, 다음 코드를 실행하여 다시 `maxroach` 사용자로 연결하지만, 이번에는 포함된 모든 명령문이 커밋되거나 중단되는 한 계좌에서 다른 계좌로 자금을 이전하기 위해 원자성 트랜잭션으로 명령문 그룹을 실행할 것입니다:

{% include copy-clipboard.html %}
~~~ shell
$ dotnet run
~~~

{{site.data.alerts.callout_info}}
기본 `SERIALIZABLE` 격리 수준을 사용하면, CockroachDB는 읽기/쓰기 경합 시 [클라이언트가 트랜잭션을 다시 시도하기](transactions.html#transaction-retries)를 요구할 수 있습니다.  CockroachDB 는 트랜잭션 내에서 실행되고 필요에 따라 재시도하는 일반적인 **재시도 함수**를 제공합니다. 여기서 재시도 함수를 복사하여 코드에 붙여 넣을 수 있습니다.

{{site.data.alerts.end}}

출력은 다음과 같아야 합니다:

~~~
Initial balances:
	account 1: 1000
	account 2: 250
Final balances:
	account 1: 900
	account 2: 350
~~~

그러나, 한 계좌에서 다른 계좌로 자금이 이전되었는지 확인하려면 [built-in SQL client](use-the-built-in-sql-client.html)을 사용하시오:

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

[.NET Npgsql driver](http://www.npgsql.org/) 사용법에 대해 자세히 읽어보세요.

{% include {{ page.version.version }}/app/see-also-links.md %}
