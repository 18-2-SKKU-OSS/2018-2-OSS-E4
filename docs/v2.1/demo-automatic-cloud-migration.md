---
title: Cross-Cloud Migration
summary: Use a local cluster to simulate migrating from one cloud platform to another.
toc: true
---

CockroachDB의 유연한 [replication controls](configure-replication-zones.html) 를 통해 클라우드 플랫폼 간에 단일 CockroachDB 클러스터를 3배 이상 쉽게 실행하고 서로 다른 클라우드 간에 서비스 중단없이 데이터를 이송할 수 있습니다. 이 페이지에서는 그러한 프로세스의 로컬 시뮬레이션에 대해 안내합니다.


## demo 영상 보기

<iframe width="560" height="315" src="https://www.youtube.com/embed/cCJkgZy6s2Q" frameborder="0" allowfullscreen></iframe>

## Step 1. 필수 구성 요소 설치하기

이 튜토리얼에서는 CockroachDB, HAProxy 로드 밸런서(load balancer) 및 Go YCSB 로드 생성기(load generator)의 CockroachDB 버전을 사용합니다. 시작하기 전에, 다음과 같이 응용 프로그램을 설치하십시오.

- Install the latest version of [CockroachDB](install-cockroachdb.html).
- Install [HAProxy](http://www.haproxy.org/). If you're on a Mac and using Homebrew, use `brew install haproxy`.
- Install [Go](https://golang.org/doc/install) version 1.9 or higher. If you're on a Mac and using Homebrew, use `brew install go`. You can check your local version by running `go version`.
- Install the [CockroachDB version of YCSB](https://github.com/cockroachdb/loadgen/tree/master/ycsb): `go get github.com/cockroachdb/loadgen/ycsb`

또한 클러스터의 데이터 파일 및 로그를 추적하려면 새 디렉토리 (e.g., `mkdir cloud-migration`) 를 생성하고, 생성된 디렉토리에서 모든 노드를 시작할 수 있습니다.

## Step 2. "cloud 1" 에서 3-노드 클러스터 시작하기

이미 로컬 클러스터를 시작한 경우 [started a local cluster](start-a-local-cluster.html), 노드의 시작을 위한 명령어를 숙지해야 합니다. 새로운 플래그는  [`--locality`](configure-replication-zones.html#descriptive-attributes-assigned-to-nodes) 로, 노드의 지형도를 묘사하는 키 값의 쌍을 받아들입니다. 이 경우, 첫 3개의 노드가 클라우드 1에서 실행 중임을 명시하는 플래그를 사용합니다.

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

## Step 3. 클러스터 초기화하기

새로운 터미널에서, [`cockroach init`](initialize-a-cluster.html) 명령어를 사용하여 클러스터를 초기화합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach init \
--insecure \
--host=localhost:26257
~~~

## Step 4. HAProxy 로드 밸런싱 설정하기

You're now running 3 nodes in a simulated cloud. Each of these nodes is an equally suitable SQL gateway to your cluster, but to ensure an even balancing of client requests across these nodes, you can use a TCP load balancer. 이제 시뮬레이션 클라우드에서 3개의 노드를 실행하고 있습니다. 이러한 각 노드는 클러스터에 동일하게 적합한 SQL 게이트웨이이지만 이러한 노드 간에 클라이언트 요청의 균형을 맞추기 위해 TCP 로드 밸런서를 사용할 수 있습니다. 앞서 설치한 [HAProxy](http://www.haproxy.org/) 로드 밸런서(load balancer) 오픈소스를 사용하십시오. 

In a new terminal, run the [`cockroach gen haproxy`](generate-cockroachdb-resources.html) command, specifying the port of any node:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach gen haproxy \
--insecure \
--host=localhost:26257
~~~

이 명령은 실행 중인 클러스터의 3개 노드에서 작동하도록 자동으로 구성된 `haproxy.cfg` 파일을 생성합니다. 이 파일에서, `bind :26257` 를 `bind :26000` 으로 변경합니다. 이는 This changes the port on which HAProxy accepts requests to a port that is not already in use by a node and that will not be used by the nodes you'll add later.

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

Start HAProxy, with the `-f` flag pointing to the `haproxy.cfg` file:

{% include copy-clipboard.html %}
~~~ shell
$ haproxy -f haproxy.cfg
~~~

## Step 5. load generator 시작하기

Now that you have a load balancer running in front of your cluster, let's use the YCSB load generator that you installed earlier to simulate multiple client connections, each performing mixed read/write workloads.

In a new terminal, start `ycsb`, pointing it at HAProxy's port:

{% include copy-clipboard.html %}
~~~ shell
$ $HOME/go/bin/ycsb -duration 20m -tolerate-errors -concurrency 10 -max-rate 1000 'postgresql://root@localhost:26000?sslmode=disable'
~~~

This command initiates 10 concurrent client workloads for 20 minutes, but limits the total load to 1000 operations per second (since you're running everything on a single machine).

## Step 6. 3개 노드에서 전체 데이터 밸런스 보기

Now open the Admin UI at `http://localhost:8080` and click **Metrics** in the left-hand navigation bar. The **Overview** dashboard is displayed. Hover over the **SQL Queries** graph at the top. After a minute or so, you'll see that the load generator is executing approximately 95% reads and 5% writes across all nodes:

<img src="{{ 'images/v2.1/admin_ui_sql_queries.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

Scroll down a bit and hover over the **Replicas per Node** graph. Because CockroachDB replicates each piece of data 3 times by default, the replica count on each of your 3 nodes should be identical:

<img src="{{ 'images/v2.1/admin_ui_replicas_migration.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 7. "cloud 2"에 노드 3개 추가하기

At this point, you're running three nodes on cloud 1. But what if you'd like to start experimenting with resources provided by another cloud vendor? Let's try that by adding three more nodes to a new cloud platform. Again, the flag to note is [`--locality`](configure-replication-zones.html#descriptive-attributes-assigned-to-nodes), which you're using to specify that these next 3 nodes are running on cloud 2.

In a new terminal, start node 4 on cloud 2:

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

In a new terminal, start node 5 on cloud 2:

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

In a new terminal, start node 6 on cloud 2:

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

## Step 8. 6개 노드에서 전체 데이터 밸런스 보기

Back on the **Overview** dashboard in Admin UI, hover over the **Replicas per Node** graph again. Because you used [`--locality`](configure-replication-zones.html#descriptive-attributes-assigned-to-nodes) to specify that nodes are running on 2 clouds, you'll see an approximately even number of replicas on each node, indicating that CockroachDB has automatically rebalanced replicas across both simulated clouds:

<img src="{{ 'images/v2.1/admin_ui_replicas_migration2.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

Note that it takes a few minutes for the Admin UI to show accurate per-node replica counts on hover. This is why the new nodes in the screenshot above show 0 replicas. However, the graph lines are accurate, and you can click **View node list** in the **Summary** area for accurate per-node replica counts as well.

## Step 9. 모든 데이터를 "cloud 2"로 이송하기

So your cluster is replicating across two simulated clouds. But let's say that after experimentation, you're happy with cloud vendor 2, and you decide that you'd like to move everything there. Can you do that without interruption to your live client traffic? Yes, and it's as simple as running a single command to add a [hard constraint](configure-replication-zones.html#replication-constraints) that all replicas must be on nodes with `--locality=cloud=2`.

In a new terminal, [edit the default replication zone](configure-zone.html):

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --execute="ALTER RANGE default CONFIGURE ZONE USING constraints='[+cloud=2]';" --insecure --host=localhost:26257
~~~

## Step 10. 데이터 이송 확인하기

Back on the **Overview** dashboard in the Admin UI, hover over the **Replicas per Node** graph again. Very soon, you'll see the replica count double on nodes 4, 5, and 6 and drop to 0 on nodes 1, 2, and 3:

<img src="{{ 'images/v2.1/admin_ui_replicas_migration3.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

This indicates that all data has been migrated from cloud 1 to cloud 2. In a real cloud migration scenario, at this point you would update the load balancer to point to the nodes on cloud 2 and then stop the nodes on cloud 1. But for the purpose of this local simulation, there's no need to do that.

## Step 11. 클러스터 정지시키기

Once you're done with your cluster, stop YCSB by switching into its terminal and pressing **CTRL-C**. Then do the same for HAProxy and each CockroachDB node.

{{site.data.alerts.callout_success}}For the last node, the shutdown process will take longer (about a minute) and will eventually force kill the node. This is because, with only 1 node still online, a majority of replicas are no longer available (2 of 3), and so the cluster is not operational. To speed up the process, press <strong>CTRL-C</strong> a second time.{{site.data.alerts.end}}

If you do not plan to restart the cluster, you may want to remove the nodes' data stores and the HAProxy config file:

{% include copy-clipboard.html %}
~~~ shell
$ rm -rf cloud1node1 cloud1node2 cloud1node3 cloud2node4 cloud2node5 cloud2node6 haproxy.cfg
~~~

## 더 알아보기

CockroachDB의 기타 주요 이점 및 기능을 살펴보세요.

{% include {{ page.version.version }}/misc/explore-benefits-see-also.md %}

You may also want to learn other ways to control the location and number of replicas in a cluster:

- [Even Replication Across Datacenters](configure-replication-zones.html#even-replication-across-datacenters)
- [Multiple Applications Writing to Different Databases](configure-replication-zones.html#multiple-applications-writing-to-different-databases)
- [Stricter Replication for a Table and Its Indexes](configure-replication-zones.html#stricter-replication-for-a-table-and-its-secondary-indexes)
- [Tweaking the Replication of System Ranges](configure-replication-zones.html#tweaking-the-replication-of-system-ranges)
