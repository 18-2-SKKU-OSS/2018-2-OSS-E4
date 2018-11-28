---
title: Fault Tolerance & Recovery
summary: Use a local cluster to explore how CockroachDB remains available during, and recovers after, failure.
toc: true
---

이 페이지에서는 장애가 발생했을 때 CockroachDB가 어떻게 활성 상태를 유지하면서 복구되는지 간단히 설명합니다. 우선 3-노드 로컬 클러스터를 생성하고, 클러스터로부터 노드를 제거합니다. 이때 사용자는 클러스터가 중단없이 계속됨을 확인할 수 있습니다. 그런 다음 노드가 오프라인 상태일 때 데이터를 작성하고, 노드를 다시 결합시키면 이후 나머지 클러스터와 일치하게 됩니다. 마지막으로, 네 번째 노드를 추가하고 노드를 다시 제거하면 누락된 복제본이 새 노드에 어떻게 다시 복제되는지 확인할 수 있습니다.


## 시작하기 전에

CockroachDB를 이미 설치했는지 확인하십시오. [installed CockroachDB](install-cockroachdb.html)

## Step 1. 3-노드 클러스터 시작하기

[`cockroach start`](start-a-node.html) 명령어를 사용하여 세 개의 노드를 시작하십시오.

{% include copy-clipboard.html %}
~~~ shell
# In a new terminal, start node 1:
$ cockroach start \
--insecure \
--store=fault-node1 \
--listen-addr=localhost:26257 \
--http-addr=localhost:8080 \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

{% include copy-clipboard.html %}
~~~ shell
# In a new terminal, start node 2:
$ cockroach start \
--insecure \
--store=fault-node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

{% include copy-clipboard.html %}
~~~ shell
# In a new terminal, start node 3:
$ cockroach start \
--insecure \
--store=fault-node3 \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

## Step 2. 클러스터 초기화하기

In a new terminal, use the [`cockroach init`](initialize-a-cluster.html) command to perform a one-time initialization of the cluster:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach init \
--insecure \
--host=localhost:26257
~~~

## Step 3. 클러스터가 활성 상태인지 확인하기

In a new terminal, use the [`cockroach sql`](use-the-built-in-sql-client.html) command to connect the built-in SQL shell to any node:

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

Exit the SQL shell:

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

## Step 4. 일시적으로 노드 제거하기

In the terminal running node 2, press **CTRL-C** to stop the node.

Alternatively, you can open a new terminal and run the [`cockroach quit`](stop-a-node.html) command against port `26258`:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach quit --insecure --host=localhost:26258
~~~

~~~
initiating graceful shutdown of server
ok
~~~

## Step 5. 클러스터가 활성상태를 유지하는지 확인하기

Switch to the terminal for the built-in SQL shell and reconnect the shell to node 1 (port `26257`) or node 3 (port `26259`):

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure --host=localhost:26259
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

As you see, despite one node being offline, the cluster continues uninterrupted because a majority of replicas (2/3) remains available. If you were to remove another node, however, leaving only one node live, the cluster would be unresponsive until another node was brought back online.

Exit the SQL shell:

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

## Step 6. 오프라인 상태인 노드에 데이터 작성하기

In the same terminal, use the [`cockroach gen`](generate-cockroachdb-resources.html) command to generate an example `startrek` database:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach gen example-data startrek | cockroach sql --insecure --host=localhost:26257
~~~

~~~
CREATE DATABASE
SET
DROP TABLE
DROP TABLE
CREATE TABLE
INSERT 79
CREATE TABLE
INSERT 200
~~~

Then reconnect the SQL shell to node 1 (port `26257`) or node 3 (port `26259`) and verify that the new `startrek` database was added with two tables, `episodes` and `quotes`:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure --host=localhost:26259
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
| startrek      |
| system        |
+---------------+
(4 rows)
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW TABLES FROM startrek;
~~~

~~~
+------------+
| table_name |
+------------+
| episodes   |
| quotes     |
+------------+
(2 rows)
~~~

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM startrek.episodes LIMIT 10;
~~~

~~~
+----+--------+-----+--------------------------------+----------+
| id | season | num |             title              | stardate |
+----+--------+-----+--------------------------------+----------+
|  1 |      1 |   1 | The Man Trap                   |   1531.1 |
|  2 |      1 |   2 | Charlie X                      |   1533.6 |
|  3 |      1 |   3 | Where No Man Has Gone Before   |   1312.4 |
|  4 |      1 |   4 | The Naked Time                 |   1704.2 |
|  5 |      1 |   5 | The Enemy Within               |   1672.1 |
|  6 |      1 |   6 | Mudd's Women                   |   1329.8 |
|  7 |      1 |   7 | What Are Little Girls Made Of? |   2712.4 |
|  8 |      1 |   8 | Miri                           |   2713.5 |
|  9 |      1 |   9 | Dagger of the Mind             |   2715.1 |
| 10 |      1 |  10 | The Corbomite Maneuver         |   1512.2 |
+----+--------+-----+--------------------------------+----------+
(10 rows)
~~~

Exit the SQL shell:

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

## Step 7. 노드를 클러스터에 다시 결합하기

Switch to the terminal for node 2, and rejoin the node to the cluster, using the same command that you used in step 1:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start --insecure \
--store=fault-node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257
~~~

~~~
CockroachDB node starting at {{page.release_info.start_time}}
build:      CCL {{page.release_info.version}} @ {{page.release_info.build_time}}
admin:      http://localhost:8081
sql:        postgresql://root@localhost:26258?sslmode=disable
logs:       node2/logs
store[0]:   path=fault-node2
status:     restarted pre-existing node
clusterID:  {5638ba53-fb77-4424-ada9-8a23fbce0ae9}
nodeID:     2
~~~

## Step 8. 다시 결합된 노드 확인하기

Switch to the terminal for the built-in SQL shell, connect the shell to the rejoined node 2 (port `26258`), and check for the `startrek` data that was added while the node was offline:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure --host=localhost:26258
~~~

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM startrek.episodes LIMIT 10;
~~~

~~~
+----+--------+-----+--------------------------------+----------+
| id | season | num |             title              | stardate |
+----+--------+-----+--------------------------------+----------+
|  1 |      1 |   1 | The Man Trap                   |   1531.1 |
|  2 |      1 |   2 | Charlie X                      |   1533.6 |
|  3 |      1 |   3 | Where No Man Has Gone Before   |   1312.4 |
|  4 |      1 |   4 | The Naked Time                 |   1704.2 |
|  5 |      1 |   5 | The Enemy Within               |   1672.1 |
|  6 |      1 |   6 | Mudd's Women                   |   1329.8 |
|  7 |      1 |   7 | What Are Little Girls Made Of? |   2712.4 |
|  8 |      1 |   8 | Miri                           |   2713.5 |
|  9 |      1 |   9 | Dagger of the Mind             |   2715.1 |
| 10 |      1 |  10 | The Corbomite Maneuver         |   1512.2 |
+----+--------+-----+--------------------------------+----------+
(10 rows)
~~~

At first, while node 2 is catching up, it acts as a proxy to one of the other nodes with the data. This shows that even when a copy of the data is not local to the node, it has seamless access.

Soon enough, node 2 catches up entirely. To verify, open the Admin UI at `http://localhost:8080` to see that all three nodes are listed, and the replica count is identical for each. This means that all data in the cluster has been replicated 3 times; there's a copy of every piece of data on each node.

{{site.data.alerts.callout_success}}CockroachDB replicates data 3 times by default. You can customize the number and location of replicas for the entire cluster or for specific sets of data using <a href="configure-replication-zones.html">replication zones</a>.{{site.data.alerts.end}}

<img src="{{ 'images/v2.1/recovery1.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 9. 다른 노드 추가하기

Now, to prepare the cluster for a permanent node failure, open a new terminal and add a fourth node:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=fault-node4 \
--listen-addr=localhost:26260 \
--http-addr=localhost:8083 \
--join=localhost:26257,localhost:26258,localhost:26259
~~~

~~~
CockroachDB node starting at {{page.release_info.start_time}}
build:      CCL {{page.release_info.version}} @ {{page.release_info.build_time}}
admin:      http://localhost:8083
sql:        postgresql://root@localhost:26260?sslmode=disable
logs:       node4/logs
store[0]:   path=fault-node4
status:     initialized new node, joined pre-existing cluster
clusterID:  {5638ba53-fb77-4424-ada9-8a23fbce0ae9}
nodeID:     4
~~~

## Step 10. 노드를 영구적으로 제거하기

Again, switch to the terminal running node 2 and press **CTRL-C** to stop it.

Alternatively, you can open a new terminal and run the [`cockroach quit`](stop-a-node.html) command against port `26258`:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach quit --insecure --host=localhost:26258
~~~

~~~
initiating graceful shutdown of server
ok
server drained and shutdown completed
~~~

## Step 11. 클러스터가 누락된 복제본을 다시 복제하는지 확인하기

Back in the Admin UI, you'll see 4 nodes listed. After about 1 minute, the dot next to node 2 will turn yellow, indicating that the node is not responding.

<img src="{{ 'images/v2.1/recovery2.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

After about 10 minutes, node 2 will move into a **Dead Nodes** section, indicating that the node is not expected to come back. At this point, in the **Live Nodes** section, you should also see that the **Replicas** count for node 4 matches the count for node 1 and 3, the other live nodes. This indicates that all missing replicas (those that were on node 2) have been re-replicated to node 4.

<img src="{{ 'images/v2.1/recovery3.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 12.  클러스터 정지시키기

Once you're done with your test cluster, stop each node by switching to its terminal and pressing **CTRL-C**.

{{site.data.alerts.callout_success}}For the last node, the shutdown process will take longer (about a minute) and will eventually force kill the node. This is because, with only 1 node still online, a majority of replicas are no longer available (2 of 3), and so the cluster is not operational. To speed up the process, press <strong>CTRL-C</strong> a second time.{{site.data.alerts.end}}

If you do not plan to restart the cluster, you may want to remove the nodes' data stores:

{% include copy-clipboard.html %}
~~~ shell
$ rm -rf fault-node1 fault-node2 fault-node3 fault-node4 fault-node5
~~~

## 더 알아보기

CockroachDB의 기타 주요 이점 및 기능을 살펴보세요.

{% include {{ page.version.version }}/misc/explore-benefits-see-also.md %}
