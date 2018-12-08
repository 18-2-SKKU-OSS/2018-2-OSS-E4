---
title: Cross-Cloud Migration
summary: Use a local cluster to simulate migrating from one cloud platform to another.
toc: true
---

CockroachDB의 유연한 [복제 제어](configure-replication-zones.html) 를 통해 클라우드 플랫폼 간에 단일 CockroachDB 클러스터를 3배 이상 쉽게 실행하고 서로 다른 클라우드 간에 서비스 중단없이 데이터를 이송할 수 있습니다. 이 페이지에서는 그러한 프로세스의 로컬 시뮬레이션에 대해 안내합니다.


## 데모 영상 보기

<iframe width="560" height="315" src="https://www.youtube.com/embed/cCJkgZy6s2Q" frameborder="0" allowfullscreen></iframe>

## 1단계. 필수 구성 요소 설치하기

이 튜토리얼에서는 CockroachDB, HAProxy 로드 밸런서(load balancer) 및 Go YCSB 로드 생성기(load generator)의 CockroachDB 버전을 사용합니다. 시작하기 전에, 다음과 같이 응용 프로그램을 설치하십시오.

- 최신 버전의 [CockroachDB](install-cockroachdb.html)를 설치하십시오.
- [HAProxy](http://www.haproxy.org/)를 설치하십시오. Mac을 사용하고 Homebrew를 사용한다면, `brew install haproxy`를 사용하십시오.
- [Go](https://golang.org/doc/install) 버전 1.9 이상을 설치하십시오. Mac을 사용하고 Homebrew를 사용한다면, `brew install go`를 사용하십시오. `go version`을 실행하여 로컬 버전을 확인할 수 있습니다.
- [CockroachDB version of YCSB](https://github.com/cockroachdb/loadgen/tree/master/ycsb)을 설치하십시오: `go get github.com/cockroachdb/loadgen/ycsb`

또한 클러스터의 데이터 파일 및 로그를 추적하려면 새 디렉토리 (e.g., `mkdir cloud-migration`) 를 생성하고, 생성된 디렉토리에서 모든 노드를 시작할 수 있습니다.

## 2단계. "cloud 1" 에서 3-노드 클러스터 시작하기

이미 [로컬 클러스터를 시작](start-a-local-cluster.html)한 경우, 노드의 시작을 위한 명령어를 숙지해야 합니다. 새로운 플래그는  [`--locality`](configure-replication-zones.html#descriptive-attributes-assigned-to-nodes) 로, 노드의 지형도를 묘사하는 키 값의 쌍을 받아들입니다. 이 경우, 첫 3개의 노드가 클라우드 1에서 실행 중임을 명시하는 플래그를 사용합니다.

새로운 터미널을 열고, 클라우드 1에서 노드 1을 시작합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--locality=cloud=1 \
--store=cloud1node1 \
--listen-addr=localhost:26257 \
--http-addr=localhost:8080 \
--cache=100MB \
--join=localhost:26257,localhost:26258,localhost:26259
~~~~

새로운 터미널을 열고, 클라우드 1에서 노드 2을 시작합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--locality=cloud=1 \
--store=cloud1node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--cache=100MB \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

새로운 터미널을 열고, 클라우드 1에서 노드 3을 시작합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--locality=cloud=1 \
--store=cloud1node3 \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--cache=100MB \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

## 3단계. 클러스터 초기화하기

새로운 터미널에서, [`cockroach init`](initialize-a-cluster.html) 명령어를 사용하여 클러스터를 초기화합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach init \
--insecure \
--host=localhost:26257
~~~

## 4단계. HAProxy 로드 밸런싱 설정하기

이제 시뮬레이션 클라우드에서 3개의 노드를 실행하고 있습니다. 이러한 각 노드는 클러스터에 동일하게 적합한 SQL 게이트웨이이지만 이러한 노드 간에 클라이언트 요청의 균형을 맞추기 위해 TCP 로드 밸런서를 사용할 수 있습니다. 앞서 설치한 [HAProxy](http://www.haproxy.org/) 로드 밸런서(load balancer) 오픈소스를 사용하십시오. 

새로운 터미널에서 임의의 노드에 포트를 지정하는 [`cockroach gen haproxy`](generate-cockroachdb-resources.html) 명령을 실행합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach gen haproxy \
--insecure \
--host=localhost:26257
~~~

이 명령은 실행 중인 클러스터의 3개 노드에서 작동하도록 자동으로 구성된 `haproxy.cfg` 파일을 생성합니다. 이 파일에서, `bind :26257` 를 `bind :26000` 으로 변경합니다. 이는 HAProxy가 요청을 수락하는 포트를 노드에서 아직 사용되지 않은 포트 혹은 나중에 추가할 노드에서 사용되지 않을 포트로 변화시킵니다.
~~~
global
  maxconn 4096

defaults
    mode                tcp
    # Timeout values should be configured for your specific use.
    # See: https://cbonte.github.io/haproxy-dconv/1.8/configuration.html#4-timeout%20connect
    timeout connect     10s
    timeout client      1m
    timeout server      1m
    # TCP keep-alive on client side. Server already enables them.
    option              clitcpka

listen psql
    bind :26000
    mode tcp
    balance roundrobin
    option httpchk GET /health?ready=1
    server cockroach1 localhost:26257 check port 8080
    server cockroach2 localhost:26258 check port 8081
    server cockroach3 localhost:26259 check port 8082
~~~

`haproxy.cfg` 파일을 가리키는 `-f` 플래그로 HAProxy를 시작합니다.

{% include copy-clipboard.html %}
~~~ shell
$ haproxy -f haproxy.cfg
~~~

## 5단계. 로드 생성기 시작하기

이제 클러스터 앞에서 로드 밸런서(load balancer)가 실행되고 있습니다. 앞서 설치한 YCSB 로드 생성기(load generator)를 사용하여 혼합된 읽기/쓰기 워크로드를 각각 수행하는 여러 개의 클라이언트 연결을 시뮬레이션합니다.

새로운 터미널에서, HAProxy의 포트를 가리키며 `ycsb` 를 시작합니다.

{% include copy-clipboard.html %}
~~~ shell
$ $HOME/go/bin/ycsb -duration 20m -tolerate-errors -concurrency 10 -max-rate 1000 'postgresql://root@localhost:26000?sslmode=disable'
~~~

이 명령은 동시 클라이언트 워크로드 10개를 20분 동안 개시하지만, 총합 로드를 초당 1000개의 작업으로 제한합니다.(한 개의 머신에서 모든 작업을 실행하기 때문에)

## 6단계. 3개 노드에서 전체 데이터 밸런스 보기

`http://localhost:8080` 에서 관리 UI를 열고 왼쪽 탐색 모음(nevigation bar)에서 **Metrics** 를 클릭하십시오. **Overview** 대시보드가 표시될 것입니다. 맨 위에 있는 **SQL Queries** 그래프 위로 마우스를 가져가십시오. 1분 정도 지나면 로드 생성기가 모든 노드에서 약 95% 읽기 및 5% 쓰기 작업을 실행하고 있음을 볼 수 있습니다.

<img src="{{ 'images/v2.1/admin_ui_sql_queries.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

이제 약간 아래로 스크롤 하여 **Replicas per Node** 그래프 위로 마우스를 가져가십시오. CockroachDB는 기본적으로 각 데이터 조각을 3회 복제하므로 3개 노드의 복제본 수는 동일해야 합니다.

<img src="{{ 'images/v2.1/admin_ui_replicas_migration.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## 7단계. "cloud 2"에 노드 3개 추가하기

현재 시점에서 클라우드 1은 3개의 노드를 실행하고 있습니다. 하지만 만약 다른 클라우드에서 리소스를 제공받는다면 어떨까요? 그럼 이제 새 클라우드 플랫폼에 노드를 3개 더 추가해 보겠습니다. 다시 한 번 강조하지만, [`--locality`](configure-replication-zones.html#descriptive-attributes-assigned-to-nodes) 플래그를 사용하여 다음 3개 노드가 클라우드 2에서 실행 중임을 지정합니다.

새로운 터미널을 열어 클라우드 2에서 노드 4를 시작합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--locality=cloud=2 \
--store=cloud2node4 \
--listen-addr=localhost:26260 \
--http-addr=localhost:8083 \
--cache=100MB \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

새로운 터미널을 열어 클라우드 2에서 노드 5를 시작합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--locality=cloud=2 \
--store=cloud2node5 \
--advertise-addr=localhost:26261 \
--http-addr=localhost:8084 \
--cache=100MB \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

새로운 터미널을 열어 클라우드 2에서 노드 6을 시작합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--locality=cloud=2 \
--store=cloud2node6 \
--advertise-addr=localhost:26262 \
--http-addr=localhost:8085 \
--cache=100MB \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

## 8단계. 6개 노드에서 전체 데이터 밸런스 보기

관리(Admin) UI의 **Overview** 대시보드로 돌아가, **Replicas per Node** (노드 당 복제본) 그래프로 다시 이동합니다. [`--locality`](configure-replication-zones.html#descriptive-attributes-assigned-to-nodes) 를 사용하여 노드가 2개의 클라우드에서 실행되고 있음을 지정했기 때문에, CockroachDB가 두 개의 클라우드 간 복제본의 균형을 자동으로 재조정하여 각 노드에 거의 동일한 수의 복제본이 나타나는 것을 확인할 수 있다.

<img src="{{ 'images/v2.1/admin_ui_replicas_migration2.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

관리 UI가 호버에서 노드당 복제본 수를 정확하게 표시하는 데 몇 분 정도 걸립니다. 따라서 위의 스크린샷의 새 노드가 0개의 복제본을 표시하는 것입니다. 그러나 그래프의 선은 정확하며, **Summary** 영역의 **View node list** 를 클릭하여 정확한 노드당 복제본 수를 확인할 수 있습니다.

## 9단계. 모든 데이터를 "cloud 2"로 이송하기

따라서 클러스터가 두 개의 클라우드 간에 복제되는 것입니다. 그런데 이후 사용자가 클라우드 2에 만족하여 모든 것을 해당 클라우드 공급업체로 전환하기로 결정했다고 가정해 보겠습니다. 실시간 클라이언트 트래픽을 중단하지 않고 그 작업을 수행할 수 있을까요? 답은 그렇다, 입니다. 단일 명령을 실행하여 모든 복제본이 `--locality=cloud=2` 를 만족하는 노드에 있어야 하는 [엄격한 제한](configure-replication-zones.html#replication-constraints)을 추가하는 간단한 과정을 통해 모든 데이터를 클라우드 2로 이송할 수 있습니다.

새 터미널에서, [기본 복제 영역을 편집](configure-zone.html)합니다:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --execute="ALTER RANGE default CONFIGURE ZONE USING constraints='[+cloud=2]';" --insecure --host=localhost:26257
~~~

## 10단계. 데이터 이송 확인하기

관리(Admin) UI의 **Overview** 대시보드로 돌아가, **Replicas per Node** (노드 당 복제본) 그래프로 다시 이동합니다. 노드 4, 5, 6에서는 복제본 수가 2배로 늘어나고 노드 1, 2, 3에서는 0으로 떨어지는 것을 확인할 수 있습니다.

<img src="{{ 'images/v2.1/admin_ui_replicas_migration3.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

이는 모든 데이터가 클라우드 1에서 클라우드 2로 이송되었음을 의미합니다. 실제 클라우드 간 이송 시나리오에서는 로드 밸런서를 업데이트하여 클라우드 2의 노드를 가리킨 다음 클라우드 1의 노드를 중지합니다. 하지만 이 로컬 시뮬레이션에서는 그런 작업이 필요하지 않습니다.

## 11단계. 클러스터 정지시키기

 모든 작업을 완료했다면, YCSB 에 해당하는 터미널로 전환한 후 **CTRL-C** 를 눌러 YCSB를 중지합니다. HAProxy 및 각 CockroachDB 노드에 대해 동일한 작업을 수행합니다.

{{site.data.alerts.callout_success}}마지막 노드의 경우, 종료 프로세스가 더 오래 걸리고(약 1분) 결국 노드를 강제로 삭제할 것입니다. 이는 1개의 노드만 온라인 상태일 때 대부분의 복제본은 더 이상 사용할 수 없는 상태이므로(3개 중 2개) 클러스터가 작동하지 않기 때문입니다. 종료 프로세스의 속도를 높이려면 <strong>CTRL-C</strong>를 한 번 더 누르십시오.{{site.data.alerts.end}}

클러스터를 다시 시작할 계획이 없다면 노드의 데이터 저장소와 HAProxy 구성 파일을 제거해도 됩니다.

{% include copy-clipboard.html %}
~~~ shell
$ rm -rf cloud1node1 cloud1node2 cloud1node3 cloud2node4 cloud2node5 cloud2node6 haproxy.cfg
~~~

## 더 알아보기

CockroachDB의 기타 주요 이점 및 기능을 살펴보세요.

{% include {{ page.version.version }}/misc/explore-benefits-see-also.md %}

클러스터에서 복제본 위치 및 수를 제어하는 다른 방법을 배우고 싶다면 아래를 참고하십시오.

- [데이터 센터 간의 복제](configure-replication-zones.html#even-replication-across-datacenters)
- [다른 데이터베이스에 작성하는 여러 어플리케이션](configure-replication-zones.html#multiple-applications-writing-to-different-databases)
- [테이블 및 인덱스에 대한 더욱 엄격한 복제](configure-replication-zones.html#stricter-replication-for-a-table-and-its-secondary-indexes)
- [시스템 범위의 복제 조정](configure-replication-zones.html#tweaking-the-replication-of-system-ranges)
