---
title: Automatic Rebalancing
summary: Use a local cluster to explore how CockroachDB automatically rebalances data as you scale.
toc: true
---

이 페이지에서는 CockroachDB가 확장시 자동으로 데이터를 재조정하는 방법에 대한 간단한 설명을 보여줍니다. 3-노드 로컬 클러스터부터, CockroachDB에서 복제되는 데이터 단위인 단일 범위의 최대 크기를 줄입니다. 그런 다음 계속해서 클러스터에 데이터를 삽입하는`block_writer` 예제 프로그램을 다운로드하여 실행하고 범위가 나뉠수록 복제본 수가 빠르게 증가하는 것을 지켜볼 것입니다. 그런 다음 당신은 2개의 노드를 추가하고 CockroachDB가 복제본을 자동으로 재조정하여 사용 가능한 모든 용량을 효율적으로 사용하는 방법을 관찰할 것입니다.

## 시작하기 전에

이 튜토리얼에서는 Go 예제 프로그램을 사용하여 CockroachDB 클러스터에 데이터를 빠르게 삽입합니다. 예제 프로그램을 실행하기 위해서는, Go 1.7.1의 64 비트 버전이있는 [Go environment](http://golang.org/doc/code.html)가 있어야합니다.

- 공식 사이트에서 [Go binary](http://golang.org/doc/code.html)를 직접 다운로드할 수 있습니다.
- [여기](https://golang.org/doc/code.html#GOPATH)에 설명된 대로 '$GOPATH' 및 '$PATH' 환경 변수를 설정해야 합니다.

## Step 1. 3-노드 클러스터 시작하기

[cockroach start](start-a-node.html) 명령을 사용하여 3개의 노드를 시작합니다.

새 터미널에서 첫번째 노드를 추가하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=scale-node1 \
--listen-addr=localhost:26257 \
--http-addr=localhost:8080 \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

새 터미널에서 두번째 노드를 추가하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=scale-node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

새 터미널에서 세번째 노드를 추가하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=scale-node3 \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

## Step 2. 클러스터를 초기화하기

새 터미널에서 [`cockroach init`](initialize-a-cluster.html) 명령을 사용하여 클러스터의 1회 초기화를 수행하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach init \
--insecure \
--host=localhost:26257
~~~

## Step 3. 클러스터가 활성 상태인지 확인하기

새 터미널에서 [빌트인 SQL 쉘](built-in-sql-client.html 사용)을 모든 노드에 연결하여 클러스터가 활성 상태인지 확인합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure --host=localhost:26257
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW DATABASES;
~~~

~~~
+---------------+
| database_name |
+---------------+
| defaultdb     |
| postgres      |
| system        |
+---------------+
(3 rows)
~~~

SQL 쉘을 종료합니다

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

## Step 4. 최대 범위 크기 낮추기

CockroachDB에서는 [레플리케이션 영역](configure-replication-zones.html)을 사용하여 복제본의 수와 위치를 제어합니다. 처음에는 각 데이터 범위를 세 번 복사하도록 설정된 전체 클러스터에 대한 단일 기본 레플리케이션 영역이 있습니다. 이 데모에서는 이 기본 레플리케이션 요소가 적합합니다.

그러나 기본 레플리케이션 영역에서는 단일 범위의 데이터가 두 범위로 분할되는 크기도 정의합니다. Since you want to create many ranges quickly and then see how CockroachDB automatically rebalances them, use [`ALTER RANGE ... CONFIGURE ZONE`](configure-zone.html) to reduce the max range size from the default 67108864 bytes (64MB) to cause ranges to split more quickly: 많은 범위를 신속하게 만들고 CockroachDB가 자동으로 균형을 조정하는 방법을 보고 싶다면 [ALTER RANGE ... CONFIGURE ZONE`] (configure-zone.html)을 사용하여 최대 범위 크기를 기본 67108864 바이트 (64MB) 범위가 더 빨리 분할되게하려면 다음을 수행하십시오.
{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --execute="ALTER RANGE default CONFIGURE ZONE USING range_min_bytes=1, range_max_bytes=262144;" --insecure --host=localhost:26257
~~~

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --execute="SHOW ZONE CONFIGURATION FOR RANGE default;" --insecure
~~~

~~~
  zone_name |                config_sql
+-----------+------------------------------------------+
  .default  | ALTER RANGE default CONFIGURE ZONE USING
            |     range_min_bytes = 1,
            |     range_max_bytes = 262144,
            |     gc.ttlseconds = 90000,
            |     num_replicas = 3,
            |     constraints = '[]',
            |     lease_preferences = '[]'
(1 row)
~~~

## Step 5. `block_writer` 프로그램을 다운로드하고 실행하기

CockroachDB provides a number of [example programs in Go](https://github.com/cockroachdb/examples-go) for simulating client workloads. The program you'll use for this demonstration is called [`block_writer`](https://github.com/cockroachdb/examples-go/tree/master/block_writer). It will simulate multiple clients inserting data into the cluster.

Download and install the program:

{% include copy-clipboard.html %}
~~~ shell
$ go get github.com/cockroachdb/examples-go/block_writer
~~~

Then run the program for 1 minute, long enough to generate plenty of ranges:

{% include copy-clipboard.html %}
~~~ shell
$ block_writer -duration 1m
~~~

Once it's running, `block_writer` will output the number of rows written per second:

~~~
 1s:  776.7/sec   776.7/sec
 2s:  696.3/sec   736.7/sec
 3s:  659.9/sec   711.1/sec
 4s:  557.4/sec   672.6/sec
 5s:  485.0/sec   635.1/sec
 6s:  563.5/sec   623.2/sec
 7s:  725.2/sec   637.7/sec
 8s:  779.2/sec   655.4/sec
 9s:  859.0/sec   678.0/sec
10s:  960.4/sec   706.1/sec
~~~

## Step 6. 복제 수가 증가하는 것을 지켜보기

Open the Admin UI at `http://localhost:8080` and you’ll see the bytes, replica count, and other metrics increase as the `block_writer` program inserts data.

<img src="{{ 'images/v2.1/scalability1.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 7. 2개의 노드를 추가하기

Adding capacity is as simple as starting more nodes and joining them to the running cluster:

{% include copy-clipboard.html %}
~~~ shell
# In a new terminal, start node 4:
$ cockroach start \
--insecure \
--store=scale-node4 \
--listen-addr=localhost:26260 \
--http-addr=localhost:8083 \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

{% include copy-clipboard.html %}
~~~ shell
# In a new terminal, start node 5:
$ cockroach start \
--insecure \
--store=scale-node5 \
--listen-addr=localhost:26261 \
--http-addr=localhost:8084 \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

## Step 8. 5개 노드 모두에서 데이터 재조정보기

Back in the Admin UI, you'll now see 5 nodes listed. At first, the bytes and replica count will be lower for nodes 4 and 5. Very soon, however, you'll see those metrics even out across all nodes, indicating that data has been automatically rebalanced to utilize the additional capacity of the new nodes.

<img src="{{ 'images/v2.1/scalability2.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 9.  클러스터를 중지하기

Once you're done with your test cluster, stop each node by switching to its terminal and pressing **CTRL-C**.

{{site.data.alerts.callout_success}}For the last node, the shutdown process will take longer (about a minute) and will eventually force kill the node. This is because, with only 1 node still online, a majority of replicas are no longer available (2 of 3), and so the cluster is not operational. To speed up the process, press <strong>CTRL-C</strong> a second time.{{site.data.alerts.end}}

If you do not plan to restart the cluster, you may want to remove the nodes' data stores:

{% include copy-clipboard.html %}
~~~ shell
$ rm -rf scale-node1 scale-node2 scale-node3 scale-node4 scale-node5
~~~

## 더 알아보기

Explore other core CockroachDB benefits and features:

{% include {{ page.version.version }}/misc/explore-benefits-see-also.md %}
