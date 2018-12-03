---
title: Start a Cluster in Docker (Insecure)
summary: Run an insecure multi-node CockroachDB cluster across multiple Docker containers on a single host.
toc: true
asciicast: true
allowed_hashes: [os-mac, os-linux, os-windows]
---

<!--
To link directly to the linux or windows tab, append #os-linux or #os-windows to the url.
-->

<div id="os-tabs" class="filters clearfix">
    <button id="mac" class="filter-button" data-scope="os-mac">Mac</button>
    <button id="linux" class="filter-button" data-scope="os-linux">Linux</button>
    <button id="windows" class="filter-button" data-scope="os-windows">Windows</button>
</div>

[CockroachDB를 설치](install-cockroachdb.html)했다면, Docker 볼륨을 사용하여 노드 데이터를 유지하면서 단일 호스트의 멀티 Docker 컨테이너에서 인시큐어한 멀티노드 클러스터를 실행하는 것은 간단합니다.

{{site.data.alerts.callout_danger}}
Docker에서 CockroachDB와 같은 상태 저장 애플리케이션을 실행하는 것은 Docker를 사용하는 것보다 훨씬 복잡하고 오류가 발생하기 쉽기에 프로덕션 배포에는 권장되지 않습니다. 물리적으로 분산된 클러스터를 컨테이너 환경에서 실행하려면 Kubernetes 또는 Docker Swarm과 같은 오케스트레이션 도구를 사용하십시오. 자세한 내용은 [Orchestration](orchestration.html)을 참조하십시오.
{{site.data.alerts.end}}

<div id="toc" style="display: none"></div>

<div class="filter-content current" markdown="1" data-scope="os-mac">
{% include {{ page.version.version }}/start-in-docker/mac-linux-steps.md %}

## 5단계. 클러스터 모니터링

첫 번째 컨테이너/노드를 시작할 때 노드의 기본 HTTP 포트 `8080`이 호스트의 포트 `8080`에 매핑되었습니다. 클러스터의 Admin UI 치수(metrics)를 확인하려면 브라우저가 `localhost`의 해당 포트(예: `http://localhost:8080`)를 가리키도록 한 다음 왼쪽의 네비게이션 바에서 **Metrics**을 클릭합니다.

앞서 언급했듯이 CockroachDB는 백그라운드에서 자동으로 데이터를 복제합니다. 이전 단계에서 기록한 데이터가 성공적으로 복제되었는지 확인하려면 아래로 스크롤하여 **Replicas per Node**를 확인하십시오.

<img src="{{ 'images/v2.1/admin_ui_replicas_per_node.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

각 노드의 레플리카 카운트가 동일한 것은, 클러스터에 있는 모든 데이터가 3회 복제되었음(기본값)을 나타냅니다.

{{site.data.alerts.callout_success}}CockroachDB의 자동 레플리케이션, 리밸런싱, 장애 내성 등에 더 자세히 알고 싶으면, <a href="demo-data-replication.html">레플리케이션</a>, <a href="demo-automatic-rebalancing.html">리밸런싱</a>, <a href="demo-fault-tolerance-and-recovery.html">장애 내성</a>을 참조하십시오.{{site.data.alerts.end}}

## 6단계. 클러스터 중지

`docker stop`과`docker rm` 명령을 사용하여 컨테이너와 클러스터를 중지하고 제거하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ docker stop roach1 roach2 roach3
~~~

{% include copy-clipboard.html %}
~~~ shell
$ docker rm roach1 roach2 roach3
~~~

클러스터를 다시 시작하지 않으려는 경우, 노드의 데이터 저장소를 제거할 수 있습니다.

{% include copy-clipboard.html %}
~~~ shell
$ rm -rf cockroach-data
~~~
</div>

<div class="filter-content" markdown="1" data-scope="os-linux">
{% include {{ page.version.version }}/start-in-docker/mac-linux-steps.md %}

## 5단계. 클러스터 모니터링

첫 번째 컨테이너/노드를 시작할 때 노드의 기본 HTTP 포트 `8080`이 호스트의 포트 `8080`에 매핑되었습니다. 클러스터의 Admin UI 크기(metrics)를 확인하려면 브라우저가 `localhost`의 해당 포트(예: `http://localhost:8080`)를 가리키도록 한 다음 왼쪽의 네비게이션 바에서 **Metrics**을 클릭합니다.

앞서 언급했듯이 CockroachDB는 백그라운드에서 자동으로 데이터를 복제합니다. 이전 단계에서 기록한 데이터가 성공적으로 복제되었는지 확인하려면 아래로 스크롤하여 **Replicas per Node**를 확인하십시오.

<img src="{{ 'images/v2.1/admin_ui_replicas_per_node.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

각 노드의 레플리카 카운트가 동일한 것은, 클러스터에 있는 모든 데이터가 3회 복제되었음(기본값)을 나타냅니다.

{{site.data.alerts.callout_success}}
CockroachDB의 자동 레플리케이션, 리밸런싱, 장애 내성 등에 더 자세히 알고 싶으면, [레플리케이션](demo-data-replication.html), [리밸런싱](demo-automatic-rebalancing.html), [장애 내성](demo-fault-tolerance-and-recovery.html)을 참조하십시오. 
{{site.data.alerts.end}}

## 6단계. 클러스터 중지

`docker stop`과`docker rm` 명령을 사용하여 컨테이너와 클러스터를 중지하고 제거하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ docker stop roach1 roach2 roach3
~~~

{% include copy-clipboard.html %}
~~~ shell
$ docker rm roach1 roach2 roach3
~~~

클러스터를 다시 시작하지 않으려는 경우, 노드의 데이터 저장소를 제거할 수 있습니다.

{% include copy-clipboard.html %}
~~~ shell
$ rm -rf cockroach-data
~~~
</div>

<div class="filter-content" markdown="1" data-scope="os-windows">

## 시작하기 전에

공식 CockroachDB Docker 이미지를 아직 설치하지 않았다면 [CockroachDB를 설치](install-cockroachdb.html)로 이동하여 **Use Docker**의 지침을 따르십시오.

## 1단계. 브리지망(bridge network) 생성

하나의 컨테이너 당 하나의 CockroachDB 노드를 사용하여 단일 호스트에서 여러 개의 Docker 컨테이너를 실행하게되므로, Docker가 [브리지망](https://docs.docker.com/engine/userguide/networking/#/a-bridge-network)이라고 부르는 것을 생성해야합니다. 브리지망를 사용하면 컨테이너를 외부 네트워크와 격리된 상태에서 단일 클러스터로 통신할 수 있습니다.

<div class="language-powershell highlighter-rouge"><pre class="highlight"><code><span class="nb">PS </span>C:\Users\username&gt; docker network create -d bridge roachnet</code></pre></div>

여기서는 네트워크 이름으로 `roachnet`을 썼지만, 네트워크에 원하는 이름을 지정해도 됩니다.

## 2단계. 첫 노드를 실행

{{site.data.alerts.callout_info}}<code>-v</code> 플래그의 <code>&#60;username&#62;</code>을 실제 사용자 이름으로 바꾸십시오.{{site.data.alerts.end}}

<div class="language-powershell highlighter-rouge"><pre class="highlight"><code><span class="nb">PS </span>C:\Users\username&gt; docker run -d <span class="sb">`</span>
--name<span class="o">=</span>roach1 <span class="sb">`</span>
--hostname<span class="o">=</span>roach1 <span class="sb">`</span>
--net<span class="o">=</span>roachnet <span class="sb">`</span>
-p 26257:26257 -p 8080:8080 <span class="sb">`</span>
-v <span class="s2">"//c/Users/&lt;username&gt;/cockroach-data/roach1:/cockroach/cockroach-data"</span> <span class="sb">`</span>
{{page.release_info.docker_image}}:{{page.release_info.version}} <span class="nb">start</span> --insecure</code></pre></div>

이 명령은 컨테이너를 생성하고 그 안에서 첫 번째 CockroachDB 노드를 실행합니다. 각 부분을 살펴 보겠습니다.

- `docker run`: 새 컨테이너를 실행시키는 Docker 명령.
- `-d`: 이 플래그는 백그라운드에서 컨테이너를 실행하므로 같은 쉘에서 다음 단계를 계속할 수 있습니다.
- `--name`: 컨테이너의 이름. 선택 사항이지만 사용자 정의 이름을 사용하면 다른 명령(예: 컨테이너에서 Bash 세션을 열거나 컨테이너를 중지할 때)에서 컨테이너를 쉽게 참조 할 수 있습니다.
- `--hostname`: 컨테이너의 호스트 명. 이를 사용하여 다른 컨테이너 / 노드를 클러스터에 연결할 수 있습니다.
- `- net`: 컨테이너가 접속할 브리지망. 자세한 내용은 1 단계를 참조하십시오.
- `-p 26257:26257 -p 8080:8080`: 이 플래그는 컨테이너에서 호스트로 노드 간 및 클라이언트 노드 통신을 위한 기본 포트 (`26257`)와 Admin UI (`8080`)에 대한 HTTP 요청의 기본 포트를 매핑합니다. 이는 컨테이너 간 통신을 가능하게 하며 브라우저에서 Admin UI를 불러올 수 있게 합니다.
- `-v "//c/Users/<username>/cockroach-data/roach1:/cockroach/cockroach-data"`: 이 플래그는 호스트 디렉토리를 데이터 볼륨으로 마운트합니다. 즉, 이 노드의 데이터와 로그는 호스트의 `Users/<username>/cockroach-data/roach1`에 저장되고 컨테이너가 정지되거나 삭제된 후에도 유지됩니다. 자세한 내용은 Docker의 <a href="https://docs.docker.com/engine/admin/volumes/bind-mounts/">Bind Mounts</a> 항목을 참조하십시오.
- `{{page.release_info.docker_image}}:{{page.release_info.version}} start --insecure`: 인시큐어한 모드의 컨테이너에서 [노드를 실행하는](start-a-node.html) CockroachDB 명령.

## 3단계. 클러스터에 노드 추가하기

이 시점에, 클러스터는 활성화되어 작동 중입니다. 하나의 노드만 있어도 SQL 클라이언트를 연결하고 데이터베이스 구축을 시작할 수 있습니다. 그러나 실제 환경에서는 최소 3개 이상의 노드가 있어야 CockroachDB의 [자동 레플리케이션](demo-data-replication.html), [리밸런싱](demo-automatic-rebalancing.html), [장애 내성 기능](demo-fault-tolerance-and-recovery.html)을 사용할 수 있습니다.

실제 환경을 시뮬레이션하려면 두 개의 노드를 더 추가하여 클러스터를 확장하십시오.

{{site.data.alerts.callout_info}}다시 한번, <code>-v</code> 플래그의 <code>&#60;username&#62;</code>을 실제 사용자 이름으로 바꾸십시오.{{site.data.alerts.end}}

<div class="language-powershell highlighter-rouge"><pre class="highlight"><code><span class="c1"># Start the second container/node:</span>
<span class="nb">PS </span>C:\Users\username&gt; docker run -d <span class="sb">`</span>
--name<span class="o">=</span>roach2 <span class="sb">`</span>
--hostname<span class="o">=</span>roach2 <span class="sb">`</span>
--net<span class="o">=</span>roachnet <span class="sb">`</span>
-v <span class="s2">"//c/Users/&lt;username&gt;/cockroach-data/roach2:/cockroach/cockroach-data"</span> <span class="sb">`</span>
{{page.release_info.docker_image}}:{{page.release_info.version}} <span class="nb">start</span> --insecure --join<span class="o">=</span>roach1

<span class="c1"># Start the third container/node:</span>
<span class="nb">PS </span>C:\Users\username&gt; docker run -d <span class="sb">`</span>
--name<span class="o">=</span>roach3 <span class="sb">`</span>
--hostname<span class="o">=</span>roach3 <span class="sb">`</span>
--net<span class="o">=</span>roachnet <span class="sb">`</span>
-v <span class="s2">"//c/Users/&lt;username&gt;/cockroach-data/roach3:/cockroach/cockroach-data"</span> <span class="sb">`</span>
{{page.release_info.docker_image}}:{{page.release_info.version}} <span class="nb">start</span> --insecure --join<span class="o">=</span>roach1</code></pre></div>

이 명령들은 컨테이너를 두 개 더 추가하고 CockroachDB 노드를 실행하여 첫 번째 노드에 연결합니다. 2단계에서 유의할 점이 몇 가지 있습니다.

- `-v`: 이 플래그는 호스트 디렉토리를 데이터 볼륨으로 마운트합니다. 이 노드에 대한 데이터와 로그는 호스트의 `Users/<username>/cockroach-data/roach2`와 `Users/<username>/cockroach-data/roach3`에 저장되며 컨테이너가 정지되거나 삭제된 후에도 유지됩니다.
- `--join`: 이 플래그는 첫 번째 컨테이너의 `hostname`을 사용하여 새로운 노드를 클러스터에 연결합니다. 각 노드는 고유한 컨테이너에 있으므로 동일한 기본 포트를 사용하더라도 충돌이 발생하지 않습니다.

## 4단계. 클러스터 테스트

이제 3개의 노드로 확장되었습니다. 어떤 노드라도 클러스터를 사용하기 위한 SQL 게이트웨이로 사용가능합니다. 이것을 보기 위해 `docker exec` 명령을 사용하여 첫 번째 컨테이너에서 [빌트인 SQL 쉘](use-the-built-in-sql-client.html)를 시작하십시오.

<div class="language-powershell highlighter-rouge"><pre class="highlight"><code><span class="nb">PS </span>C:\Users\username&gt; docker <span class="nb">exec</span> -it roach1 ./cockroach sql --insecure
<span class="c1"># Welcome to the cockroach SQL interface.</span>
<span class="c1"># All statements must be terminated by a semicolon.</span>
<span class="c1"># To exit: CTRL + D.</span></code></pre></div>

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
+----+---------+
| id | balance |
+----+---------+
|  1 |  1000.5 |
+----+---------+
(1 row)
~~~

노드 1의 SQL 쉘에서 나옵니다.

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

두 번째 컨테이너에서 SQL 쉘을 시작합니다.

<div class="language-powershell highlighter-rouge"><pre class="highlight"><code><span class="nb">PS </span>C:\Users\username&gt; docker <span class="nb">exec</span> -it roach2 ./cockroach sql --insecure
<span class="c1"># Welcome to the cockroach SQL interface.</span>
<span class="c1"># All statements must be terminated by a semicolon.</span>
<span class="c1"># To exit: CTRL + D.</span></code></pre></div>

동일한 `SELECT` 쿼리를 실행합니다.

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM bank.accounts;
~~~

~~~
+----+---------+
| id | balance |
+----+---------+
|  1 |  1000.5 |
+----+---------+
(1 row)
~~~

보시다시피, 노드 1과 노드 2의 SQL 게이트웨이는 동일하게 동작합니다.

노드 2의 SQL 클라이언트에서 나옵니다.

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

## 5단계. 클러스터 모니터링

첫 번째 컨테이너/노드를 시작할 때 노드의 기본 HTTP 포트 `8080`이 호스트의 포트 `8080`에 매핑되었습니다. 클러스터의 [Admin UI](admin-ui-overview.html) 크기(metrics)를 확인하려면 브라우저가 `localhost`의 해당 포트(예: `http://localhost:8080`)를 가리키도록 한 다음 왼쪽의 네비게이션 바에서 **Metrics**을 클릭합니다.

앞서 언급했듯이 CockroachDB는 백그라운드에서 자동으로 데이터를 복제합니다. 이전 단계에서 기록한 데이터가 성공적으로 복제되었는지 확인하려면 아래로 스크롤하여 **Replicas per Node**를 확인하십시오.

<img src="{{ 'images/v2.1/admin_ui_replicas_per_node.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

각 노드의 레플리카 카운트가 동일한 것은, 클러스터에 있는 모든 데이터가 3회 복제되었음(기본값)을 나타냅니다.

{{site.data.alerts.callout_success}}
CockroachDB의 자동 레플리케이션, 리밸런싱, 장애 내성 등에 더 자세히 알고 싶으면, [레플리케이션](demo-data-replication.html), [리밸런싱](demo-automatic-rebalancing.html), [장애 내성](demo-fault-tolerance-and-recovery.html)을 참조하십시오.
{{site.data.alerts.end}}

## 6단계. 클러스터 중지

`docker stop`과`docker rm` 명령을 사용하여 컨테이너와 클러스터를 중지하고 제거하십시오.

<div class="language-powershell highlighter-rouge"><pre class="highlight"><code><span class="c1"># Stop the containers:</span>
<span class="nb">PS </span>C:\Users\username&gt; docker stop roach1 roach2 roach3

<span class="c1"># Remove the containers:</span>
<span class="nb">PS </span>C:\Users\username&gt; docker rm roach1 roach2 roach3</code></pre></div>

클러스터를 다시 시작하지 않으려는 경우, 노드의 데이터 저장소를 제거할 수 있습니다.

<div class="language-powershell highlighter-rouge"><pre class="highlight"><code><span class="nb">PS </span>C:\Users\username&gt; rm -rf cockroach-data</span></code></pre></div>

</div>

## 다음은?

- [CockroachDB SQL](learn-cockroachdb-sql.html)과 [빌트인 SQL client](use-the-built-in-sql-client.html)에 대해 더 알아보기
- 선호하는 언어를 사용하여 [클라이언트 드라이버 설치](install-client-drivers.html)
- [애플리케이션을 CockroachDB와 만들기](build-an-app-with-cockroachdb.html)
- 자동 레플리케이션, 리밸런싱, 장애 내성, 클라우드 통합과 같은 [CockroachDB 코어 기능](demo-data-replication.html) 둘러보기.
