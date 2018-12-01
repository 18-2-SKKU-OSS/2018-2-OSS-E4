---
title: Start a Local Cluster (Secure)
summary: Run a secure multi-node CockroachDB cluster locally, using TLS certificates to encrypt network communication.
toc: true
asciicast: true
---

<div class="filters filters-big clearfix">
    <a href="start-a-local-cluster.html"><button class="filter-button">Insecure</button></a>
    <a href="secure-a-cluster.html"><button class="filter-button current"><strong>Secure</strong></button></a>
</div>

[CockroachDB를 설치](install-cockroachdb.html)했다면, 네트워크 통신을 암호화하기 위한 [TLS 인증서](create-security-certificates.html)를 사용한, 즉 시큐어한 멀티노드 클러스터를 로컬에서 실행하는 것은 간단합니다.

{{site.data.alerts.callout_info}}
단일 호스트에서 여러 노드를 실행하는 것은 CockroachDB를 테스트하는 데 유용하지만 프로덕션 배포에는 권장되지 않습니다. 물리적으로 분산된 클러스터를 프로덕션 환경에서 실행하려면, [수동배포](manual-deployment.html) 또는 [오케스트레이션 배포](orchestration.html)를 참조하십시오.
{{site.data.alerts.end}}


## 시작하기 전에

[CockroachDB 설치](install-cockroachdb.html)가 필요합니다.

<!-- TODO: update the asciicast
Also, feel free to watch this process in action before going through the steps yourself. Note that you can copy commands directly from the video, and you can use **<** and **>** to go back and forward.

<asciinema-player class="asciinema-demo" src="asciicasts/secure-a-cluster.json" cols="107" speed="2" theme="monokai" poster="npt:0:52" title="Secure a Cluster"></asciinema-player>
-->

## 1단계. 보안 인증서 작성

[`cockroach cert` 명령](create-security-certificates.html)이나 [`openssl` 명령](create-security-certificates-openssl.html)을 사용하여 보안 인증서를 생성할 수 있습니다. 이 섹션에서는 `cockroach cert` 명령에 대해 설명합니다.

1. 인증서를 위한 디렉토리 및 CA 키를 위한 안전한 디렉토리 만들기:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ mkdir certs my-safe-directory
    ~~~

    기본 인증서 디렉토리 (`${HOME}/.cockroach-certs`)를 사용하는 경우, 그것이 비어 있는지 확인하십시오.

2. CA(인증 기관) 인증서 및 키페어 만들기:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-ca \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

3. `root` 사용자의 경우, 클라이언트 인증서와 키 만들기:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-client \
    root \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

    `client.root.crt`와`client.root.key`는 내장된 SQL 쉘과 클러스터 간의 통신을 보호하는 데 사용됩니다 (4 단계 참조).

4. 노드 인증서 및 키 만들기:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-node \
    localhost \
    $(hostname) \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

    `node.crt`와`node.key`는 노드 간의 통신을 보호하는 데 사용됩니다. 일반적으로 각 노드는 고유한 주소를 가지므로 여러분은 위 파일들을 각 노드마다 별도로 생성했을 것입니다. 그러나 이 경우 모든 노드가 로컬에서 실행되므로 오직 하나의 노드 인증서와 키를 생성해야 합니다.

## 2단계. 첫 노드를 실행

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--certs-dir=certs \
--listen-addr=localhost
~~~

~~~
CockroachDB node starting at 2018-09-13 01:25:57.878119479 +0000 UTC (took 0.3s)
build:               CCL {{page.release_info.version}} @ {{page.release_info.build_time}}
webui:               https://localhost:8080
sql:                 postgresql://root@ROACHs-MBP:26257?sslcert=%2FUsers%2F...
client flags:        cockroach <client cmd> --host=localhost:26257 --certs-dir=certs
logs:                cockroach/cockroach-data/logs
temp dir:            cockroach-data/cockroach-temp998550693
external I/O path:   cockroach-data/extern
store[0]:            path=cockroach-data
status:              initialized new cluster
clusterID:           2711b3fa-43b3-4353-9a23-20c9fb3372aa
nodeID:              1
~~~

이 명령어는 시큐어 모드의 노드를 실행합니다. 대부분은 [`cockroach start`](start-a-node.html) 기본값을 사용합니다.

- `--certs-dir` 디렉토리는 인증서와 키를 가지고있는 디렉토리를 가리킵니다.
- 순수하게 로컬 클러스터이면, `--listen-addr=localhost`는 노드가 오직 `localhost`만 받으며, 내부 클라이언트 트래픽은 포트(`26257`), 운영 UI의 HTTP 요청은(`8080`)을 사용합니다.
- 노드 데이터는 `cockroach-data` 디렉토리에 저장됩니다.
- [standard output](start-a-node.html#standard-output)은 CockroachDB 버전, Admin UI URL, SQL URL 등의 유용한 세부정보를 보여줍니다.

## 3단계. 클러스터에 노드 추가하기

이 시점에, 클러스터는 활성화되어 작동 중입니다. 하나의 노드만 있어도 SQL 클라이언트를 연결하고 데이터베이스 구축을 시작할 수 있습니다. 그러나 실제 환경에서는 최소 3개 이상의 노드가 있어야 CockroachDB의 [자동 레플리케이션](demo-data-replication.html), [리밸런싱](demo-automatic-rebalancing.html), [장애 내성 기능](demo-fault-tolerance-and-recovery.html)을 사용할 수 있습니다. 이 단계는 실제 배포를 로컬에서 시뮬레이션합니다.

새 터미널에서 두번째 노드를 추가하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--certs-dir=certs \
--store=node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257
~~~

새 터미널에서 세번째 노드를 추가하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--certs-dir=certs \
--store=node3 \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--join=localhost:26257
~~~

이 명령어의 주요 차이점은 `--join`로 첫 번째 노드의 주소와 포트를 지정(이 경우에는 `localhost:26257`)하여 새 노드를 클러스터에 연결하는 것입니다. 동일한 시스템에서 모든 노드가 실행되므로, `--store`, `--listen-addr`, `--http-addr`로 다른 노드가 사용하지 않는 위치와 포트를 설정합니다. 실제 환경에서는 각 노드가 다른 시스템에 있으므로 기본값을 사용할 수 있습니다.

## 4단계. 클러스터 테스트

이제 3개의 노드로 확장되었습니다. 어떤 노드라도 클러스터를 사용하기 위한 SQL 게이트웨이로 사용가능합니다. 이것을 보기 위해 새 터미널에서 [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)로 노드 1에 연결합니다.

{{site.data.alerts.callout_info}}SQL 클라이언트는 <code>cockroach</code> 바이너리에 내장되어 있으므로 추가 설치는 필요하지 않습니다.{{site.data.alerts.end}}

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--certs-dir=certs \
--host=localhost:26257
~~~

기본 [CockroachDB SQL](learn-cockroachdb-sql.html) 몇가지를 실행해봅시다.

{% include copy-clipboard.html %}
~~~ sql
> CREATE DATABASE bank;
~~~

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
~~~

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO bank.accounts VALUES (1, 1000.50);
~~~

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM bank.accounts;
~~~

~~~
  id | balance
+----+---------+
   1 | 1000.50
(1 row)
~~~

노드 1의 SQL 클라이언트에서 나옵니다.

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

노드 2의 SQL 클라이언트에 연결합니다. 이번에는 포트를 지정하여야 합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--certs-dir=certs \
--host=localhost:26258
~~~

{{site.data.alerts.callout_info}}실제 환경에서는 모든 노드가 기본 포트 <code>26257</code>을 사용하기 때문에 <code>--port</code>를 설정할 필요가 없습니다.{{site.data.alerts.end}}

동일한 `SELECT` 쿼리를 실행합니다.

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM bank.accounts;
~~~

~~~
  id | balance
+----+---------+
   1 | 1000.50
(1 row)
~~~

보시다시피, 노드 1과 노드 2의 SQL 게이트웨이는 동일하게 동작합니다.

마지막으로, [비밀번호로 사용자를 생성](create-user.html#create-a-user-with-a-password)합니다. 다음 단계에서 [Admin UI](admin-ui-overview.html)에 접근하기 위해 필요할 것입니다.

{% include copy-clipboard.html %}
~~~ sql
> CREATE USER roach WITH PASSWORD 'Q7gc8rEdS';
~~~

노드 2의 SQL 클라이언트에서 나옵니다.

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

## 5단계. 클러스터 모니터링

브라우저의 `http://localhost:8080` 또는, 노드 시작 시 표준 출력에서 `admin` 필드의 주소를 이용하여 [운영 UI](admin-ui-overview.html)에 접근할 수 있습니다. 브라우저는 CockroachDB 생성 인증서가 무효하다고 판단할 것입니다; UI로 들어가기 위해서 경고 메세지를 클릭해야 합니다.

[클러스터 테스트](#step-4-test-the-cluster) 단계에서 만든 사용자 이름과 비밀번호로 로그인하십시오. 그런 다음 왼쪽의 네비게이션 바에서 **Metrics**을 클릭합니다.

<img src="{{ 'images/v2.1/admin_ui_overview_dashboard.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

앞서 언급했듯이 CockroachDB는 백그라운드에서 자동으로 데이터를 복제합니다. 이전 단계에서 기록한 데이터가 성공적으로 복제되었는지 확인하려면 아래로 스크롤하여 **Replicas per Node**를 확인하십시오.

<img src="{{ 'images/v2.1/admin_ui_replicas_per_node.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

각 노드의 레플리카 카운트가 동일한 것은, 클러스터에 있는 모든 데이터가 3회 복제되었음(기본값)을 나타냅니다.

{{site.data.alerts.callout_info}} 단일 시스템에서 여러 노드를 실행하는 경우 용량 지표가 잘못될 수 있습니다. 자세한 내용은 [제한사항](known-limitations.html#available-capacity-metric-in-the-admin-ui)을 참조하십시오. {{site.data.alerts.end}}

{{site.data.alerts.callout_success}} CockroachDB의 자동 레플리케이션, 리밸런싱, 장애 내성 등에 더 자세히 알고 싶으면, [레플리케이션](demo-data-replication.html), [리밸런싱](demo-automatic-rebalancing.html), [장애 내성](demo-fault-tolerance-and-recovery.html)을 참조하십시오. {{site.data.alerts.end}}

## 6단계. 클러스터 중지

테스트 클러스터를 마치려면 첫 번째 노드를 실행하는 터미널로 전환하여 **CTRL-C**를 누릅니다.

현재 2개의 노드가 온라인 상태여서 대부분의 복제본을 사용할 수 있으므로 클러스터가 계속 동작합니다. 클러스터에서 이 `실패`를 허용했는지 확인하려면 SQL 쉘을 노드 2 또는 3에 연결하십시오. 사용중인 터미널 또는 새 터미널에서 할 수 있습니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--certs-dir=certs \
--host=localhost:26258
~~~

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM bank.accounts;
~~~

~~~
  id | balance
+----+---------+
   1 | 1000.50
(1 row)
~~~

SQL 쉘에서 나옵니다.

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

터미널로 전환하여 **CTRL-C**를 눌러 노드 2, 3을 중지합니다.

{{site.data.alerts.callout_success}}노드 3의 경우 종료 프로세스가 더 오래 걸리고(약 1분) 노드가 강제로 종료됩니다. 3개 노드 중 1개만 남아 있으면 대부분의 복제본을 사용할 수 없으므로 클러스터가 더 이상 동작하지 않습니다. 종료 프로세스를 더 빨리 하려면 **CTRL-C**를 한번 더 누르십시오.{{site.data.alerts.end}}

클러스터를 다시 시작할 계획이 없는 경우 노드의 데이터 저장소를 제거할 수 있습니다.

{% include copy-clipboard.html %}
~~~ shell
$ rm -rf cockroach-data node2 node3
~~~

## 7단계. 클러스터 재시작

클러스터를 추가 테스트에 사용하려면 3개 노드 중 적어도 2개를 저장소 데이터와 함께 재시작해야 합니다.

첫 번째 노드를 `cockroach-data/`의 상위 디렉토리에서 다시 시작합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--certs-dir=certs \
--listen-addr=localhost
~~~

{{site.data.alerts.callout_info}}
단 하나의 노드만 온라인이 되면 클러스터가 아직 동작하지 않으므로 두 번째 노드를 다시 시작할 때까지 위 명령의 응답을 볼 수 없습니다.
{{site.data.alerts.end}}

새 터미널을 열고 두 번째 노드를 `node2/`의 상위 디렉토리에서 다시 시작합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--certs-dir=certs \
--store=node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257
~~~

새 터미널을 열고 세 번째 노드를 `node3/`의 상위 디렉토리에서 다시 시작합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--certs-dir=certs \
--store=node3 \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--join=localhost:26257
~~~

## 다음은?

- [CockroachDB SQL](learn-cockroachdb-sql.html)과 [빌트인 SQL client](use-the-built-in-sql-client.html)에 대해 더 알아보기
- 선호하는 언어를 사용하여 [클라이언트 드라이버 설치](install-client-drivers.html)
- 앱을 시큐어한 클러스터에 연결하기 위해 [클라이언트 연결 인자](connection-parameters.html)의 사용 방법 알아보기
- [애플리케이션을 CockroachDB와 만들기](build-an-app-with-cockroachdb.html)
- 자동 레플리케이션, 리밸런싱, 장애 내성, 클라우드 통합과 같은 [CockroachDB 코어 기능](demo-data-replication.html) 둘러보기.
