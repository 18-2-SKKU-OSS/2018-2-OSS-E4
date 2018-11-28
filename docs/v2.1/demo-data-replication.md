---
title: Data Replication
summary: Use a local cluster to explore how CockroachDB replicates and distributes data.
toc: true
---

이 페이지에서는 CockroachDB가 데이터를 복제하고 배포하는 방법에 대해 간단히 설명합니다. 먼저 단일노드 로컬 클러스터를 시작하고, 데이터를 작성하고, 2개의 노드를 추가하고, 데이터를 자동으로 복제할 수 있습니다. 그 후 데이터를 5개로 복제하도록 클러스터를 업데이트하고, 노드를 2개 더 추가한 다음, 기존의 모든 복제본을 신규 노드에 다시 복제합니다.
## 시작하기 전에

[CockroachDB]를 이미 설치했는지 확인하십시오.(install-cockroachdb.html).

## Step 1. 단일노드 클러스터 시작하기

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=repdemo-node1 \
--listen-addr=localhost:26257 \
--http-addr=localhost:8080
~~~

## Step 2. 데이터 작성하기

새 터미널에서 [`cockroach gen`](generate-cockroachdb-resources.html) 명령어를 사용하여 `intro` 데이터베이스를 생성합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach gen example-data intro | cockroach sql --insecure --host=localhost:26257
~~~

같은 터미널에서 [built-in SQL shell](use-the-built-in-sql-client.html) 를 열고 `intro` 데이터베이스가 `mytable` 로 추가되었는지 확인합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure --host=localhost:26257
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW DATABASES;
~~~

~~~
  database_name
+---------------+
  defaultdb
  intro
  postgres
  system
(4 rows)
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW TABLES FROM intro;
~~~

~~~
  table_name
+------------+
  mytable
(1 row)
~~~

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM intro.mytable WHERE (l % 2) = 0;
~~~

~~~
  l  |                          v
+----+------------------------------------------------------+
   0 | !__aaawwmqmqmwwwaas,,_        .__aaawwwmqmqmwwaaa,,
   2 | !"VT?!"""^~~^"""??T$Wmqaa,_auqmWBT?!"""^~~^^""??YV^
   4 | !                    "?##mW##?"-
   6 | !  C O N G R A T S  _am#Z??A#ma,           Y
   8 | !                 _ummY"    "9#ma,       A
  10 | !                vm#Z(        )Xmms    Y
  12 | !              .j####mmm#####mm#m##6.
  14 | !   W O W !    jmm###mm######m#mmm##6
  16 | !             ]#me*Xm#m#mm##m#m##SX##c
  18 | !             dm#||+*$##m#mm#m#Svvn##m
  20 | !            :mmE=|+||S##m##m#1nvnnX##;     A
  22 | !            :m#h+|+++=Xmm#m#1nvnnvdmm;     M
  24 | ! Y           $#m>+|+|||##m#1nvnnnnmm#      A
  26 | !  O          ]##z+|+|+|3#mEnnnnvnd##f      Z
  28 | !   U  D       4##c|+|+|]m#kvnvnno##P       E
  30 | !       I       4#ma+|++]mmhvnnvq##P`       !
  32 | !        D I     ?$#q%+|dmmmvnnm##!
  34 | !           T     -4##wu#mm#pw##7'
  36 | !                   -?$##m####Y'
  38 | !             !!       "Y##Y"-
  40 | !
(21 rows)
~~~

SQL 셸을 종료합니다.

{% include copy-clipboard.html %}
~~~ sql
> \q
~~~

## Step 3. 노드 2개 추가하기

In a new terminal, add node 2:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=repdemo-node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257
~~~

In a new terminal, add node 3:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=repdemo-node3 \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--join=localhost:26257
~~~

## Step 4. 신규 노드로 데이터 복제하기

Open the Admin UI at <a href="http://localhost:8080" data-proofer-ignore>http://localhost:8080</a> to see that all three nodes are listed. At first, the replica count will be lower for nodes 2 and 3. Very soon, the replica count will be identical across all three nodes, indicating that all data in the cluster has been replicated 3 times; there's a copy of every piece of data on each node.

<img src="{{ 'images/v2.1/replication1.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 5. 복제 인자 증가시키기

As you just saw, CockroachDB replicates data 3 times by default. Now, in the terminal you used for the built-in SQL shell or in a new terminal, use the [`ALTER RANGE ... CONFIGURE ZONE`](configure-zone.html) statement to change the cluster's `.default` replication factor to 7:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --execute="ALTER RANGE default CONFIGURE ZONE USING num_replicas=7;" --insecure --host=localhost:26257
~~~

In addition to the `.default` replication zone for database and table data, CockroachDB comes with pre-configured replication zones for [important internal data](configure-replication-zones.html#create-a-replication-zone-for-a-system-range). To view these pre-configured zones, use the [`SHOW ZONE CONFIGURATIONS`](show-zone-configurations.html) subcommand:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --execute="SHOW ALL ZONE CONFIGURATIONS;" --insecure --host=localhost:26257
~~~

~~~
   zone_name  |                     config_sql
+-------------+-----------------------------------------------------+
  .default    | ALTER RANGE default CONFIGURE ZONE USING
              |     range_min_bytes = 1048576,
              |     range_max_bytes = 67108864,
              |     gc.ttlseconds = 90000,
              |     num_replicas = 7,
              |     constraints = '[]',
              |     lease_preferences = '[]'
  system      | ALTER DATABASE system CONFIGURE ZONE USING
              |     range_min_bytes = 1048576,
              |     range_max_bytes = 67108864,
              |     gc.ttlseconds = 90000,
              |     num_replicas = 5,
              |     constraints = '[]',
              |     lease_preferences = '[]'
  system.jobs | ALTER TABLE system.public.jobs CONFIGURE ZONE USING
              |     range_min_bytes = 1048576,
              |     range_max_bytes = 67108864,
              |     gc.ttlseconds = 600,
              |     num_replicas = 5,
              |     constraints = '[]',
              |     lease_preferences = '[]'
  .meta       | ALTER RANGE meta CONFIGURE ZONE USING
              |     range_min_bytes = 1048576,
              |     range_max_bytes = 67108864,
              |     gc.ttlseconds = 3600,
              |     num_replicas = 5,
              |     constraints = '[]',
              |     lease_preferences = '[]'
  .system     | ALTER RANGE system CONFIGURE ZONE USING
              |     range_min_bytes = 1048576,
              |     range_max_bytes = 67108864,
              |     gc.ttlseconds = 90000,
              |     num_replicas = 5,
              |     constraints = '[]',
              |     lease_preferences = '[]'
  .liveness   | ALTER RANGE liveness CONFIGURE ZONE USING
              |     range_min_bytes = 1048576,
              |     range_max_bytes = 67108864,
              |     gc.ttlseconds = 600,
              |     num_replicas = 5,
              |     constraints = '[]',
              |     lease_preferences = '[]'
(6 rows)
~~~

For the cluster as a whole to remain available, the "system ranges" for this internal data must always retain a majority of their replicas. Therefore, if you increase the default replication factor, be sure to also increase the replication factor for these replication zones as well:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --execute="ALTER RANGE liveness CONFIGURE ZONE USING num_replicas=7;" --insecure --host=localhost:26257
~~~

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --execute="ALTER RANGE meta CONFIGURE ZONE USING num_replicas=7;" --insecure --host=localhost:26257
~~~

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --execute="ALTER RANGE system CONFIGURE ZONE USING num_replicas=7;" --insecure
~~~

## Step 6. 노드 2개 추가하기

In a new terminal, add node 4:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=repdemo-node4 \
--listen-addr=localhost:26260 \
--http-addr=localhost:8083 \
--join=localhost:26257
~~~

In a new terminal, add node 5:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=repdemo-node5 \
--listen-addr=localhost:26261 \
--http-addr=localhost:8084 \
--join=localhost:26257
~~~

## Step 7. 신규 노드로 데이터 복제하기

Back in the Admin UI, you'll see that there are now 5 nodes listed. Again, at first, the replica count will be lower for nodes 4 and 5. But because you changed the default replication factor to 5, very soon, the replica count will be identical across all 5 nodes, indicating that all data in the cluster has been replicated 5 times.

<img src="{{ 'images/v2.1/replication2.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 8. 클러스터 정지시키기

Once you're done with your test cluster, stop each node by switching to its terminal and pressing **CTRL-C**.

{{site.data.alerts.callout_success}}
For the last 2 nodes, the shutdown process will take longer (about a minute) and will eventually force kill the nodes. This is because, with only 2 nodes still online, a majority of replicas are no longer available (3 of 5), and so the cluster is not operational. To speed up the process, press **CTRL-C** a second time in the nodes' terminals.
{{site.data.alerts.end}}

If you do not plan to restart the cluster, you may want to remove the nodes' data stores:

{% include copy-clipboard.html %}
~~~ shell
$ rm -rf repdemo-node1 repdemo-node2 repdemo-node3 repdemo-node4 repdemo-node5
~~~

## 더 알아보기

CockroachDB의 기타 주요 이점 및 기능을 살펴보세요.

{% include {{ page.version.version }}/misc/explore-benefits-see-also.md %}
