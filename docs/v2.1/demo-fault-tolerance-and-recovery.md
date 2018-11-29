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

새로운 터미널에서, [`cockroach init`](initialize-a-cluster.html) 명령어를 사용하여 클러스터를 초기화합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach init \
--insecure \
--host=localhost:26257
~~~

## Step 3. 클러스터가 활성 상태인지 확인하기

새로운 터미널에서, [`cockroach sql`](use-the-built-in-sql-client.html) 명령어를 사용하여 built-in SQL shell을 노드 중 하나에 연결합니다.

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

SQL 셸을 종료합니다.

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

## Step 4. 일시적으로 노드 제거하기

노드 2에 해당하는 터미널에서 **CTRL-C** 를 눌러 노드를 중지합니다.

또는, 새로운 터미널을 열어 포트 `26258` 에 대해 [`cockroach quit`](stop-a-node.html) 명령어를 실행합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach quit --insecure --host=localhost:26258
~~~

~~~
initiating graceful shutdown of server
ok
~~~

## Step 5. 클러스터가 활성상태를 유지하는지 확인하기

built-in SQL shell을 연결하기 위해 사용했던 터미널로 전환하고, node 1 (port `26257`) 또는 node 3 (port `26259`) 에 셸을 다시 연결합니다.

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

한 개의 노드가 오프라인 상태인 상황에서도 대부분의 복제본(2/3)을 사용할 수 있기 때문에 클러스터가 중단 없이 계속됩니다. 만일 노드를 하나 더 제거하여 한 개의 노드만 활성 상태가 된다면, 다른 노드가 다시 온라인 상태가 될 때까지 클러스터가 응답하지 않습니다.

SQL 셸을 중지합니다.

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

## Step 6. 오프라인 상태인 노드에 데이터 작성하기

같은 터미널에서, [`cockroach gen`](generate-cockroachdb-resources.html) 명령어를 사용하여 `startrek` 데이터베이스를 생성합니다.

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

SQL 셸을 node 1 (port `26257`) 또는 node 3 (port `26259`) 에 다시 연결하고,`startrek` 데이터베이스가 `episodes` 와 `quotes` 두 개의 테이블과 함께 추가되었는지 확인합니다.

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

SQL 셸을 종료합니다.

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

## Step 7. 노드를 클러스터에 다시 결합하기

node 2에 해당하는 터미널로 전환한 다음, step 1에서 사용한 것과 동일한 명령어를 사용하여 노드를 클러스터에 다시 연결합니다.

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

built-in SQL shell 에 해당하는 터미널로 전환한 후 셸을 클러스터에 다시 결합한 node 2 (port `26258`)에 연결합니다. 그리고 노드가 오프라인 상태일 때 추가된 `startrek` 데이터를 확인합니다.

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

초기에, 노드 2가 데이터를 가지고 있는 다른 노드 중 하나에 대한 프록시 역할을 하면서 다른 노드들을 따라잡습니다. 이는 데이터의 복사본이 노드에 대해 로컬이 아니더라도 원활하게 액세스할 수 있음을 보여줍니다.

얼마 지나지 않아 노드 2가 다른 노드들을 완전히 따라잡습니다. http://localhost:8080에서 관리(Admin) UI를 열어 세 개 노드의 복제본 수가 동일해졌는지 확인할 수 있습니다. 클러스터의 모든 데이터는 동일하게 3번 복제되고, 각각의 노드에는 전체 데이터의 복사본이 있게 됩니다.

{{site.data.alerts.callout_success}} CockroachDB는 기본적으로 데이터를 3번 복제합니다. 다음의 복제 영역을 사용하여 전체 클러스터 또는 특정 데이터 set에 대한 복제본의 수와 위치를 사용자가 임의로 정의할 수 있습니다. <a href="configure-replication-zones.html">replication zones</a>.{{site.data.alerts.end}}

<img src="{{ 'images/v2.1/recovery1.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 9. 다른 노드 추가하기

영구적인 노드 장애에 대비하기 위해 클러스터를 준비시킵니다. 새 터미널을 열고 다음 네 번째 노드를 추가하십시오.

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

노드 2에 해당하는 터미널로 전환한 후 **CTRL-C** 를 눌러 노드를 중지시킵니다.

또는, 새로운 터미널을 열어 포트 `26258` 에 대해 [`cockroach quit`](stop-a-node.html) 명령어를 실행합니다.

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

관리(Admin UI)로 다시 돌아가면, 4개의 노드가 나열된 것을 확인할 수 있습니다. 약 1분 후 노드 2 옆의 동그란 점은 노드가 응답하지 않음을 나타내는 노란색으로 바뀔 것입니다.

<img src="{{ 'images/v2.1/recovery2.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

약 10분 후, 노드 2가 **Dead Nodes** 섹션으로 이동하여 노드가 다시 돌아올 것으로 예상되지 않음을 나타냅니다. 이때 **Live Nodes** 섹션에서 노드 4에 대한 **Replicas** 개수가 노드 1과 노드 3의 개수와 일치하는지 확인해야 합니다. 이는 누락된 복제본(노드 2에 있던 복제본)이 모두 노드 4로 다시 복제되었음을 나타냅니다.

<img src="{{ 'images/v2.1/recovery3.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 12.  클러스터 정지시키기

테스트 클러스터를 완료한 후 클러스터를 정지시켜야 합니다. 각 노드에 대하여 해당 터미널로 전환하고 CTRL-C를 눌러 노드를 중지합니다.

{{site.data.alerts.callout_success}}마지막 노드의 경우, 종료 프로세스가 더 오래 걸리고(약 1분) 결국 노드를 강제로 삭제할 것입니다. 이는 1개의 노드만 온라인 상태일 때 대부분의 복제본은 더 이상 사용할 수 없는 상태이므로(3개 중 2개) 클러스터가 작동하지 않기 때문입니다. 종료 프로세스의 속도를 높이려면 <strong>CTRL-C</strong>를 한 번 더 누르십시오.{{site.data.alerts.end}}

클러스터를 다시 시작할 계획이 없다면 노드의 데이터 저장소를 제거해도 됩니다.

{% include copy-clipboard.html %}
~~~ shell
$ rm -rf fault-node1 fault-node2 fault-node3 fault-node4 fault-node5
~~~

## 더 알아보기

CockroachDB의 기타 주요 이점 및 기능을 살펴보세요.

{% include {{ page.version.version }}/misc/explore-benefits-see-also.md %}
