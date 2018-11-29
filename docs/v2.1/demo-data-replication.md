---
title: Data Replication
summary: Use a local cluster to explore how CockroachDB replicates and distributes data.
toc: true
---

이 페이지에서는 CockroachDB가 데이터를 복제하고 배포하는 방법에 대해 간단히 설명합니다. 먼저 단일노드 로컬 클러스터를 시작하고, 데이터를 작성하고, 2개의 노드를 추가하고, 데이터를 자동으로 복제할 수 있습니다. 그 후 데이터를 5개로 복제하도록 클러스터를 업데이트하고, 노드를 2개 더 추가한 다음, 기존의 모든 복제본을 신규 노드에 다시 복제합니다.
## 시작하기 전에

CockroachDB를 이미 설치했는지 확인하십시오. [installed CockroachDB](install-cockroachdb.html)

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

같은 터미널에서 [built-in SQL shell](use-the-built-in-sql-client.html) 를 열고 `intro` 데이터베이스가  `mytable` 과 함께 추가되었는지 확인합니다.

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

새 터미널에서 node 2를 추가하기:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=repdemo-node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257
~~~

새 터미널에서 node 3를 추가하기:

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

Open the Admin UI at <a href="http://localhost:8080" data-proofer-ignore>http://localhost:8080</a> to see that all three nodes are listed. <a href="http://localhost:8080" data-proofer-ignore>http://localhost:8080</a 에서 관리(Admin) UI를 열어 세 개의 노드가 모두 나열되어 있는지 확인합니다. 처음에는 노드 2 와 노드 3의 복제본 수가 더 적습니다. 그러다 곧 복제본 수가 3개의 노드에서 모두 동일해집니다. 즉, 클러스터의 모든 데이터가 동일하게 3회 복제되었음을 나타냅니다. 각각의 노드에는 전체 데이터의 복사본이 있습니다.

<img src="{{ 'images/v2.1/replication1.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 5. 복제 인자 증가시키기

앞서 확인한 것처럼, CockroachDB는 기본적으로 데이터를 3번 복제합니다. built-in SQL 셸을 열었던 터미널 또는 새 터미널에서 [`ALTER RANGE ... CONFIGURE ZONE`](configure-zone.html) 문을 사용하여 클러스터의 '.default' 복제 팩터를 7로 변경합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --execute="ALTER RANGE default CONFIGURE ZONE USING num_replicas=7;" --insecure --host=localhost:26257
~~~

CockroachDB는 데이터베이스 및 테이블 데이터에 대한 '.default' 복제 영역 외에도 [important internal data(중요 내부 데이터)](configure-replication-zons.html#a-re-re-replication])에 대해 미리 구성된 복제 영역을 제공합니다. 미리 구성된 영역을 보려면 [SHOW ZONE CONE CONNGENTS](Show-Zone-configurations.html) 하위 명령을 사용하십시오.

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

클러스터 전체를 사용 가능하도록 유지하려면 이 중요 내부 데이터에 대한 "시스템 범위"는 항상 대부분의 복제본을 보관해야 합니다. 따라서 기본 복제 인자를 바꾸는 경우 이러한 복제 영역의 복제 인자도 바꿔야 합니다.

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

새 터미널에서 노드 4를 추가하기:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=repdemo-node4 \
--listen-addr=localhost:26260 \
--http-addr=localhost:8083 \
--join=localhost:26257
~~~

새 터미널에서 노드 5를 추가하기:

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

관리(Admin) UI로 돌아가면 5개의 노드가 나열되어 있는 것을 볼 수 있습니다. 처음에는 노드 4와 5의 복제본 수가 더 적습니다. 그러다 곧 복제 수가 모든 5개 노드에서 동일해지며, 앞에서 기본 복제 팩터를 바로 5로 변경했기 때문에 클러스터의 모든 데이터가 5번 복제되는 것을 확인할 수 있습니다.

<img src="{{ 'images/v2.1/replication2.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 8. 클러스터 정지시키기

테스트 클러스터를 완료한 후 클러스터를 정지시켜야 합니다. 각 노드에 대하여 해당 터미널로 전환하고 **CTRL-C**를 눌러 노드를 중지합니다.

{{site.data.alerts.callout_success}}
마지막 2개 노드의 경우, 종료 프로세스가 더 오래 걸리고(약 1분) 결국 노드를 강제로 삭제할 것입니다. 이는 2개의 노드만 온라인 상태일 때 대부분의 복제본은 더 이상 사용할 수 없는 상태이므로(5개 중 3개) 클러스터가 작동하지 않기 때문입니다. 종료 프로세스의 속도를 높이려면 노드 터미널에서 **CTRL-C**를 한 번 더 누르십시오.
{{site.data.alerts.end}}

클러스터를 다시 시작할 계획이 없다면 노드의 데이터 저장소를 제거해도 됩니다.

{% include copy-clipboard.html %}
~~~ shell
$ rm -rf repdemo-node1 repdemo-node2 repdemo-node3 repdemo-node4 repdemo-node5
~~~

## 더 알아보기

CockroachDB의 기타 주요 이점 및 기능을 살펴보세요.

{% include {{ page.version.version }}/misc/explore-benefits-see-also.md %}
