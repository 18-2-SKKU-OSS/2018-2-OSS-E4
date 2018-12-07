---
title: Configure Replication Zones
summary: In CockroachDB, you use replication zones to control the number and location of replicas for specific sets of data.
keywords: ttl, time to live, availability zone
toc: true
---

Replication zones give you the power to control what data goes where in your CockroachDB cluster.  Specifically, they are used to control the number and location of replicas for data belonging to the following objects:

- Databases
- Tables
- Rows ([enterprise-only](enterprise-licensing.html))
- Indexes ([enterprise-only](enterprise-licensing.html))
- All data in the cluster, including internal system data ([via the default replication zone](#view-the-default-replication-zone))

For each of the above objects you can control:

- How many copies of each range to spread through the cluster.
- Which constraints are applied to which data, e.g., "table X's data can only be stored in the German datacenters".
- The maximum size of ranges (how big ranges get before they are split).
- How long old data is kept before being garbage collected.
- <span class="version-tag">New in v2.1:</span> Where you would like the leaseholders for certain ranges to be located, e.g., "for ranges that are already constrained to have at least one replica in `region=us-west`, also try to put their leaseholders in `region=us-west`".

This page explains how replication zones work and how to use the [`CONFIGURE ZONE`](configure-zone.html) statement to manage them.

{{site.data.alerts.callout_info}}
Currently, only members of the `admin` role can configure replication zones. By default, the `root` user belongs to the `admin` role.
{{site.data.alerts.end}}

## 개요

Every [range](architecture/overview.html#glossary) in the cluster is part of a replication zone.  Each range's zone configuration is taken into account as ranges are rebalanced across the cluster to ensure that any constraints are honored.

클러스터가 시작되면, 두 가지 범주의 복제 영역이 있습니다:

1. Pre-configured replication zones that apply to internal system data.
2. A single default replication zone that applies to the rest of the cluster.

You can adjust these pre-configured zones as well as add zones for individual databases, tables, rows, and secondary indexes as needed.  Note that adding zones for rows and secondary indexes is [enterprise-only](enterprise-licensing.html).

For example, you might rely on the [default zone](#view-the-default-replication-zone) to spread most of a cluster's data across all of your datacenters, but [create a custom replication zone for a specific database](#create-a-replication-zone-for-a-database) to make sure its data is only stored in certain datacenters and/or geographies.

## 복제 영역 수준

There are five replication zone levels for [**table data**](architecture/distribution-layer.html#table-data) in a cluster, listed from least to most granular:

Level | Description
------|------------
Cluster | CockroachDB comes with a pre-configured `.default` replication zone that applies to all table data in the cluster not constrained by a database, table, or row-specific replication zone. This zone can be adjusted but not removed. See [View the Default Replication Zone](#view-the-default-replication-zone) and [Edit the Default Replication Zone](#edit-the-default-replication-zone) for more details.
Database | You can add replication zones for specific databases. See [Create a Replication Zone for a Database](#create-a-replication-zone-for-a-database) for more details.
Table | You can add replication zones for specific tables. See [Create a Replication Zone for a Table](#create-a-replication-zone-for-a-table).
Index ([Enterprise-only](enterprise-licensing.html)) | The [secondary indexes](indexes.html) on a table will automatically use the replication zone for the table. However, with an enterprise license, you can add distinct replication zones for secondary indexes. See [Create a Replication Zone for a Secondary Index](#create-a-replication-zone-for-a-secondary-index) for more details.
Row ([Enterprise-only](enterprise-licensing.html)) | You can add replication zones for specific rows in a table or secondary index by [defining table partitions](partitioning.html). See [Create a Replication Zone for a Table Partition](#create-a-replication-zone-for-a-table-or-secondary-index-partition) for more details.

### 시스템 데이터

In addition, CockroachDB stores internal [**system data**](architecture/distribution-layer.html#monolithic-sorted-map-structure) in what are called system ranges. There are two replication zone levels for this internal system data, listed from least to most granular:

Level | Description
------|------------
Cluster | The `.default` replication zone mentioned above also applies to all system ranges not constrained by a more specific replication zone.
System Range | CockroachDB comes with pre-configured replication zones for the "meta" and "liveness" system ranges. If necessary, you can add replication zones for the "timeseries" range and other "system" ranges as well. See [Create a Replication Zone for a System Range](#create-a-replication-zone-for-a-system-range) for more details.<br><br>CockroachDB also comes with a pre-configured replication zone for one internal table, `system.jobs`, which stores metadata about long-running jobs such as schema changes and backups. Historical queries are never run against this table and the rows in it are updated frequently, so the pre-configured zone gives this table a lower-than-default `ttlseconds`.

### 우선 순위

When replicating data, whether table or system, CockroachDB always uses the most granular replication zone available. For example, for a piece of user data:

1. If there's a replication zone for the row, CockroachDB uses it.
2. If there's no applicable row replication zone and the row is from a secondary index, CockroachDB uses the secondary index replication zone.
3. If the row isn't from a secondary index or there is no applicable secondary index replication zone, CockroachDB uses the table replication zone.
4. If there's no applicable table replication zone, CockroachDB uses the database replication zone.
5. If there's no applicable database replication zone, CockroachDB uses the `.default` cluster-wide replication zone.

{{site.data.alerts.callout_danger}}
{% include {{page.version.version}}/known-limitations/system-range-replication.md %}
{{site.data.alerts.end}}

## 복제 영역 관리

Use the [`CONFIGURE ZONE`](configure-zone.html) statement to [add](#create-a-replication-zone-for-a-system-range), [modify](#edit-the-default-replication-zone), [reset](#reset-a-replication-zone), and [remove](#remove-a-replication-zone) replication zones.

### 복제 영역 변수

Use the [`ALTER ... CONFIGURE ZONE`](configure-zone.html) [statement](sql-statements.html) to set a replication zone:

{% include copy-clipboard.html %}
~~~ sql
> ALTER TABLE t CONFIGURE ZONE USING range_min_bytes = 0, range_max_bytes = 90000, gc.ttlseconds = 89999, num_replicas = 5, constraints = '[-region=west]';
~~~

{% include v2.1/zone-configs/variables.md %}

### 복제 제약 조건

The location of replicas, both when they are first added and when they are rebalanced to maintain cluster equilibrium, is based on the interplay between descriptive attributes assigned to nodes and constraints set in zone configurations.

{{site.data.alerts.callout_success}}For demonstrations of how to set node attributes and replication constraints in different scenarios, see <a href="#scenario-based-examples">Scenario-based Examples</a> below.{{site.data.alerts.end}}

#### 노드에 할당된 설명 속성

When starting a node with the [`cockroach start`](start-a-node.html) command, you can assign the following types of descriptive attributes:

Attribute Type | Description
---------------|------------
**Node Locality** | Using the `--locality` flag, you can assign arbitrary key-value pairs that describe the locality of the node. Locality might include country, region, datacenter, rack, etc. The key-value pairs should be ordered from most inclusive to least inclusive (e.g., country before datacenter before rack), and the keys and the order of key-value pairs must be the same on all nodes. It's typically better to include more pairs than fewer. For example:<br><br>`--locality=region=east,datacenter=us-east-1`<br>`--locality=region=east,datacenter=us-east-2`<br>`--locality=region=west,datacenter=us-west-1`<br><br>CockroachDB attempts to spread replicas evenly across the cluster based on locality, with the order determining the priority. However, locality can be used to influence the location of data replicas in various ways using replication zones.<br><br>When there is high latency between nodes, CockroachDB also uses locality to move range leases closer to the current workload, reducing network round trips and improving read performance. See [Follow-the-workload](demo-follow-the-workload.html) for more details.
**Node Capability** | Using the `--attrs` flag, you can specify node capability, which might include specialized hardware or number of cores, for example:<br><br>`--attrs=ram:64gb`
**Store Type/Capability** | Using the `attrs` field of the `--store` flag, you can specify disk type or capability, for example:<br><br>`--store=path=/mnt/ssd01,attrs=ssd`<br>`--store=path=/mnt/hda1,attrs=hdd:7200rpm`

#### 제약 조건의 유형

The node-level and store-level descriptive attributes mentioned above can be used as the following types of constraints in replication zones to influence the location of replicas. However, note the following general guidance:

- When locality is the only consideration for replication, it's recommended to set locality on nodes without specifying any constraints in zone configurations. In the absence of constraints, CockroachDB attempts to spread replicas evenly across the cluster based on locality.
- Required and prohibited constraints are useful in special situations where, for example, data must or must not be stored in a specific country or on a specific type of machine.

Constraint Type | Description | Syntax
----------------|-------------|-------
**Required** | When placing replicas, the cluster will consider only nodes/stores with matching attributes or localities. When there are no matching nodes/stores, new replicas will not be added. | `+ssd`
**Prohibited** | When placing replicas, the cluster will ignore nodes/stores with matching attributes or localities. When there are no alternate nodes/stores, new replicas will not be added. | `-ssd`

#### 제약 조건의 범위

Constraints can be specified such that they apply to all replicas in a zone or such that different constraints apply to different replicas, meaning you can effectively pick the exact location of each replica.

Constraint Scope | Description | Syntax
-----------------|-------------|-------
**All Replicas** | Constraints specified using JSON array syntax apply to all replicas in every range that's part of the replication zone. | `constraints = '[+ssd, -region=west]'`
**Per-Replica** | Multiple lists of constraints can be provided in a JSON object, mapping each list of constraints to an integer number of replicas in each range that the constraints should apply to.<br><br>The total number of replicas constrained cannot be greater than the total number of replicas for the zone (`num_replicas`). However, if the total number of replicas constrained is less than the total number of replicas for the zone, the non-constrained replicas will be allowed on any nodes/stores. | `constraints: '{+ssd,-region=west: 2, +region=east: 1}'`

### 노드/복제본 권장 사항

See [Cluster Topography](recommended-production-settings.html#cluster-topology) recommendations for production deployments.

## 복제 영역 보기

Use the [`SHOW ZONE CONFIGURATIONS`](#view-all-replication-zones) statement to view details about existing replication zones.

## 기본 예제

이 예제는 영역 구성 작업을 위한 기본 접근법 및 구문에 중점을 둡니다. 제약 조건을 사용하는 방법을 보여주는 예제는, [시나리오 기반 예제](#scenario-based-examples)를 참조하십시오.

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html) 및 [`SHOW ZONE CONFIGURATIONS`](show-zone-configurations.html)을 참조하십시오.

### 모든 복제 영역 보기

{% include v2.1/zone-configs/view-all-replication-zones.md %}

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html)을 참조하십시오.

### 기본 복제 영역 보기

{% include v2.1/zone-configs/view-the-default-replication-zone.md %}

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html)을 참조하십시오.

### 기본 복제 영역 편집

{% include v2.1/zone-configs/edit-the-default-replication-zone.md %}

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html)을 참조하십시오.

### 시스템 범위에 대한 복제 영역 생성

{% include v2.1/zone-configs/create-a-replication-zone-for-a-system-range.md %}

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html)을 참조하십시오.

### 데이터베이스의 복제 영역 생성

{% include v2.1/zone-configs/create-a-replication-zone-for-a-database.md %}

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html)을 참조하십시오.

### 테이블의 복제 영역 생성

{% include v2.1/zone-configs/create-a-replication-zone-for-a-table.md %}

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html)을 참조하십시오.

### 보조 인덱스에 대한 복제 영역 생성

{% include v2.1/zone-configs/create-a-replication-zone-for-a-secondary-index.md %}

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html)을 참조하십시오.

### 테이블 또는 보조 인덱스 파티션에 대한 복제 영역 생성

{% include v2.1/zone-configs/create-a-replication-zone-for-a-table-partition.md %}

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html)을 참조하십시오.

### 복제 영역 재설정

{% include v2.1/zone-configs/reset-a-replication-zone.md %}

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html)을 참조하십시오.

### 복제 영역 제거

{% include v2.1/zone-configs/remove-a-replication-zone.md %}

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html)을 참조하십시오.

### 특정 데이터 센터로 리스 홀더 제한

{% include v2.1/zone-configs/constrain-leaseholders-to-specific-datacenters.md %}

자세한 내용은, [`CONFIGURE ZONE`](configure-zone.html)을 참조하십시오.

## 시나리오 기반 예제

### 데이터 센터 전반의 복제

**시나리오:**

- 3개의 데이터 센터에 6개의 노드가 있고, 각 데이터 센터에 2개의 노드가 있습니다.
- 3개의 데이터 센터 전체에 균등하게 균형 잡힌 복제본을 사용하여 데이터를 3번 복제해야합니다.

**접근:**

`--locality` 플래그로 지정된 데이터 센터 위치로 각 노드를 시작하십시오:

~~~ shell
# Start the two nodes in datacenter 1:
$ cockroach start --insecure --advertise-addr=<node1 hostname> --locality=datacenter=us-1 \
--join=<node1 hostname>,<node3 hostname>,<node5 hostname>
$ cockroach start --insecure --advertise-addr=<node2 hostname> --locality=datacenter=us-1 \
--join=<node1 hostname>,<node3 hostname>,<node5 hostname>

# Start the two nodes in datacenter 2:
$ cockroach start --insecure --advertise-addr=<node3 hostname> --locality=datacenter=us-2 \
--join=<node1 hostname>,<node3 hostname>,<node5 hostname>
$ cockroach start --insecure --advertise-addr=<node4 hostname> --locality=datacenter=us-2 \
--join=<node1 hostname>,<node3 hostname>,<node5 hostname>

# Start the two nodes in datacenter 3:
$ cockroach start --insecure --advertise-addr=<node5 hostname> --locality=datacenter=us-3 \
--join=<node1 hostname>,<node3 hostname>,<node5 hostname>
$ cockroach start --insecure --advertise-addr=<node6 hostname> --locality=datacenter=us-3 \
--join=<node1 hostname>,<node3 hostname>,<node5 hostname>

# Initialize the cluster:
$ cockroach init --insecure --host=<any node hostname>
~~~

영역 구성을 변경할 필요가 없습니다. 기본적으로, 클러스터는 데이터를 세 번 복제하도록 구성되며, 명시적 제약 조건 없이도, 클러스터는 노드 지역에 걸쳐 복제본을 다양화할 것을 목표로 합니다.

### 특정 데이터 센터에 대한 복제 당 제약조건

**시나리오:**

- 3개의 지역에 5개의 데이터 센터에 5개의 노드가 있고, 각 데이터 센터에 1개의 노드가 있습니다.
- West Coast 중심의 West Coast 데이터를 보유한 데이터베이스에 대한 복제본 쿼럼과 전체 국가에 복제된 전국 데이터에 대한 데이터베이스를 사용하여 데이터를 3번 복제하려고 합니다.

**접근:**

1. `--locality` 플래그로 지정된 지역과 데이터 센터 위치로 각 노드를 시작하십시오:

    ~~~ shell
    # Start the four nodes:
    $ cockroach start --insecure --advertise-addr=<node1 hostname> --locality=region=us-west1,datacenter=us-west1-a \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>,<node4 hostname>,<node5 hostname>
    $ cockroach start --insecure --advertise-addr=<node2 hostname> --locality=region=us-west1,datacenter=us-west1-b \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>,<node4 hostname>,<node5 hostname>
    $ cockroach start --insecure --advertise-addr=<node3 hostname> --locality=region=us-central1,datacenter=us-central1-a \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>,<node4 hostname>,<node5 hostname>
    $ cockroach start --insecure --advertise-addr=<node4 hostname> --locality=region=us-east1,datacenter=us-east1-a \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>,<node4 hostname>,<node5 hostname>
    $ cockroach start --insecure --advertise-addr=<node5 hostname> --locality=region=us-east1,datacenter=us-east1-b \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>,<node4 hostname>,<node5 hostname>

    # Initialize the cluster:
    $ cockroach init --insecure --host=<any node hostname>
    ~~~

2. 모든 노드에서, [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)를 여십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure
    ~~~

3. West Coast 어플리케이션에 대한 데이터베이스를 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE west_app_db;
    ~~~

4. 데이터베이스의 복제 영역을 구성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER DATABASE west_app_db CONFIGURE ZONE USING constraints = '{+region=us-west1: 2, +region=us-central1: 1}';
    ~~~

    ~~~
    CONFIGURE ZONE 1
    ~~~

5. 복제 영역을 보십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW ZONE CONFIGURATION FOR DATABASE west_app_db;
    ~~~

    ~~~
       zone_name  |                   config_sql
    +-------------+--------------------------------------------------------------------+
      west_app_db | ALTER DATABASE west_app_db CONFIGURE ZONE USING
                  |     range_min_bytes = 1048576,
                  |     range_max_bytes = 67108864,
                  |     gc.ttlseconds = 90000,
                  |     num_replicas = 3,
                  |     constraints = '{+region=us-west1: 2, +region=us-central1: 1}',
                  |     lease_preferences = '[]'
    (1 row)
    ~~~

    Two of the database's three replicas will be put in `region=us-west1` and its remaining replica will be put in `region=us-central1`. This gives the application the resilience to survive the total failure of any one datacenter while providing low-latency reads and writes on the West Coast because a quorum of replicas are located there.

6. No configuration is needed for the nation-wide database. The cluster is configured to replicate data 3 times and spread them as widely as possible by default. Because the first key-value pair specified in each node's locality is considered the most significant part of each node's locality, spreading data as widely as possible means putting one replica in each of the three different regions.

### 다른 데이터베이스에 작성하는 여러 어플리케이션

**시나리오:**

- 서로 다른 데이터베이스를 사용하는 동일한 CockroachDB 클러스터에 연결된 2개의 독립 어플리케이션이 있습니다.
- 2개의 데이터 센터에 6개의 노드가 있고, 각 데이터 센터에 3개의 노드가 있습니다.
- 어플리케이션 1의 데이터를 5번 복제하고, 복제본을 두 데이터 센터에서 균형있게 유지하려고 합니다.
- 어플리케이션 2의 데이터를 단일 데이터 센터의 모든 복제본과 함께 3번 복제하려고 합니다.

**접근:**

1. `--locality` 플래그로 지정된 데이터 센터 위치로 각 노드를 시작하십시오:

    ~~~ shell
    # Start the three nodes in datacenter 1:
    $ cockroach start --insecure --advertise-addr=<node1 hostname> --locality=datacenter=us-1 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>,<node4 hostname>,<node5 hostname>,<node6 hostname>
    $ cockroach start --insecure --advertise-addr=<node2 hostname> --locality=datacenter=us-1 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>,<node4 hostname>,<node5 hostname>,<node6 hostname>
    $ cockroach start --insecure --advertise-addr=<node3 hostname> --locality=datacenter=us-1 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>,<node4 hostname>,<node5 hostname>,<node6 hostname>

    # Start the three nodes in datacenter 2:
    $ cockroach start --insecure --advertise-addr=<node4 hostname> --locality=datacenter=us-2 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>,<node4 hostname>,<node5 hostname>,<node6 hostname>
    $ cockroach start --insecure --advertise-addr=<node5 hostname> --locality=datacenter=us-2 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>,<node4 hostname>,<node5 hostname>,<node6 hostname>
    $ cockroach start --insecure --advertise-addr=<node6 hostname> --locality=datacenter=us-2 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>,<node4 hostname>,<node5 hostname>,<node6 hostname>

    # Initialize the cluster:
    $ cockroach init --insecure --host=<any node hostname>
    ~~~

2. 모든 노드에서, [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)를 여십시오. 

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure
    ~~~

3. 어플리케이션 1의 데이터베이스 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE app1_db;
    ~~~

4. 어플리케이션 1에서 사용되는 데이터베이스의 복제 영역 구성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER DATABASE app1_db CONFIGURE ZONE USING num_replicas = 5;
    ~~~

    ~~~
    CONFIGURE ZONE 1
    ~~~

5. 복제 영역 보십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW ZONE CONFIGURATION FOR DATABASE app1_db;
    ~~~

    ~~~
        zone_name  |                config_sql
    +--------------+---------------------------------------------+
         app1_db   | ALTER DATABASE app1_db CONFIGURE ZONE USING
                   |     range_min_bytes = 1048576,
                   |     range_max_bytes = 67108864,
                   |     gc.ttlseconds = 90000,
                   |     num_replicas = 5,
                   |     constraints = '[]',
                   |     lease_preferences = '[]'
    (1 row)
    ~~~

    어플리케이션 1의 데이터에는 다른 것이 필요하지 않습니다. 모든 노드가 데이터 센터 지역을 지정하기 때문에, 클러스터는 데이터 센터 1과 2 사이에서 어플리케이션 1이 사용하는 데이터베이스의 데이터의 균형을 유지하려고합니다.

6. 여전히 SQL 클라이언트에서, 어플리케이션 2에 대한 데이터베이스를 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE app2_db;
    ~~~

7. 어플리케이션 2에서 사용되는 데이터베이스의 복제 영역 구성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER DATABASE app2_db CONFIGURE ZONE USING constraints = '[+datacenter=us-2]';
    ~~~

8. 복제 영역:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW ZONE CONFIGURATION FOR DATABASE app2_db;
    ~~~

    ~~~
        zone_name  |                config_sql
    +--------------+---------------------------------------------+
         app2_db   | ALTER DATABASE app2_db CONFIGURE ZONE USING
                   |     range_min_bytes = 1048576,
                   |     range_max_bytes = 67108864,
                   |     gc.ttlseconds = 90000,
                   |     num_replicas = 3,
                   |     constraints = '[+datacenter=us-2]',
                   |     lease_preferences = '[]'
    (1 row)
    ~~~

    필수 제약 조건은 어플리케이션 2의 데이터가 `us-2` 데이터 센터 내에서만 복제되도록 합니다.

### 테이블 및 보조 인덱스에 대한 더욱 엄격한 복제

**시나리오:**

- 노드 7개, SSD 드라이브 5개, HDD 드라이브 2개가 있습니다.
- 기본적으로 데이터를 3번 복제하려고합니다.
- Speed and availability are important for a specific table and its indexes, which are queried very frequently, however, so you want the data in the table and secondary indexes to be replicated 5 times, preferably on nodes with SSD drives.


**접근:**

1. 저장소 속성으로 지정된 `ssd` 또는 `hdd`로 각 노드를 시작하십시오:

    ~~~ shell
    # Start the 5 nodes with SSD storage:
    $ cockroach start --insecure --advertise-addr=<node1 hostname> --store=path=node1,attrs=ssd \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>
    $ cockroach start --insecure --advertise-addr=<node2 hostname> --store=path=node2,attrs=ssd \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>
    $ cockroach start --insecure --advertise-addr=<node3 hostname> --store=path=node3,attrs=ssd \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>
    $ cockroach start --insecure --advertise-addr=<node4 hostname> --store=path=node4,attrs=ssd \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>
    $ cockroach start --insecure --advertise-addr=<node5 hostname> --store=path=node5,attrs=ssd \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>

    # Start the 2 nodes with HDD storage:
    $ cockroach start --insecure --advertise-addr=<node6 hostname> --store=path=node6,attrs=hdd \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>
    $ cockroach start --insecure --advertise-addr=<node7 hostname> --store=path=node7,attrs=hdd \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>

    # Initialize the cluster:
    $ cockroach init --insecure --host=<any node hostname>
    ~~~

2. 모든 노드에서, [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)를 여십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure
    ~~~

3. 데이터베이스 및 테이블 생성:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE db;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE db.important_table;
    ~~~

4. 보다 엄격하게 복제해야 하는 테이블의 복제 영역을 구성:

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER TABLE db.important_table CONFIGURE ZONE USING num_replicas = 5, constraints = '[+ssd]'
    ~~~

5. 복제 영역 보기:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW ZONE CONFIGURATION FOR TABLE db.important_table;
    ~~~

    ~~~
              zone_name       |                config_sql
    +-------------------------+---------------------------------------------+
         db.important_table   | ALTER DATABASE app2_db CONFIGURE ZONE USING
                              |     range_min_bytes = 1048576,
                              |     range_max_bytes = 67108864,
                              |     gc.ttlseconds = 90000,
                              |     num_replicas = 5,
                              |     constraints = '[+ssd]',
                              |     lease_preferences = '[]'
    (1 row)
    ~~~

    테이블의 보조 인덱스는 테이블의 복제 영역을 사용하므로, 테이블의 모든 데이터가 5번 복제되고, 필수 제약 조건은 `ssd` 드라이브가 있는 노드에 데이터를 배치합니다.

### 시스템 범위의 복제 조정

**시나리오:**

- 7개의 데이터 센터에 노드가 분산되어 있습니다.
- 기본적으로 데이터를 5번 복제하려고 합니다.
- 성능 향상을 위해, 모든 데이터 센터에서 메타 범위의 복사본을 원합니다.
- 디스크 공간을 절약하기 위해, 기본적으로 내부 시계열 데이터가 3번만 복제되기를 원합니다.

**접근:**

1. 각 노드를 다른 지역 속성으로 시작하십시오:

    ~~~ shell
    # Start the nodes:
    $ cockroach start --insecure --advertise-addr=<node1 hostname> --locality=datacenter=us-1 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>   
    $ cockroach start --insecure --advertise-addr=<node2 hostname> --locality=datacenter=us-2 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>
    $ cockroach start --insecure --advertise-addr=<node3 hostname> --locality=datacenter=us-3 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>
    $ cockroach start --insecure --advertise-addr=<node4 hostname> --locality=datacenter=us-4 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>
    $ cockroach start --insecure --advertise-addr=<node5 hostname> --locality=datacenter=us-5 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>
    $ cockroach start --insecure --advertise-addr=<node6 hostname> --locality=datacenter=us-6 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>
    $ cockroach start --insecure --advertise-addr=<node7 hostname> --locality=datacenter=us-7 \
    --join=<node1 hostname>,<node2 hostname>,<node3 hostname>

    # Initialize the cluster:
    $ cockroach init --insecure --host=<any node hostname>
    ~~~

2. 모든 노드에서, [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)을 여십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure
    ~~~

3. 기본 복제 영역 구성:

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER RANGE default CONFIGURE ZONE USING num_replicas = 5;
    ~~~

4. 복제 영역 보기:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW ZONE CONFIGURATION FOR RANGE default;
    ~~~
    ~~~
      zone_name |                config_sql
    +-----------+------------------------------------------+
      .default  | ALTER RANGE default CONFIGURE ZONE USING
                |     range_min_bytes = 1048576,
                |     range_max_bytes = 67108864,
                |     gc.ttlseconds = 90000,
                |     num_replicas = 5,
                |     constraints = '[]',
                |     lease_preferences = '[]'
    (1 row)
    ~~~
    
    클러스터의 모든 데이터는 SQL 데이터와 내부 시스템 데이터를 포함하여 5번 복제됩니다.

5. `.meta` 복제 영역을 구성하십시오 :

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER RANGE meta CONFIGURE ZONE USING num_replicas = 7;
    ~~~

    ~~~
      zone_name |              config_sql
    +-----------+---------------------------------------+
      .meta     | ALTER RANGE meta CONFIGURE ZONE USING
                |     range_min_bytes = 1048576,
                |     range_max_bytes = 67108864,
                |     gc.ttlseconds = 3600,
                |     num_replicas = 7,
                |     constraints = '[]',
                |     lease_preferences = '[]'
    (1 row)
    ~~~

    `.meta` 주소 지정 범위는 하나의 복사본이 모든 7개의 데이터 센터에 복제되고, 다른 모든 데이터는 5번 복제됩니다. 

6. `.timeseries` 복제 영역을 구성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER RANGE timeseries CONFIGURE ZONE USING num_replicas = 3;
    ~~~

    ~~~
       zone_name  |                 config_sql
    +-------------+---------------------------------------------+
      .timeseries | ALTER RANGE timeseries CONFIGURE ZONE USING
                  |     range_min_bytes = 1048576,
                  |     range_max_bytes = 67108864,
                  |     gc.ttlseconds = 90000,
                  |     num_replicas = 3,
                  |     constraints = '[]',
                  |     lease_preferences = '[]'
    (1 row)
    ~~~

    시계열 데이터는 다른 모든 데이터의 구성에 영향을 미치지 않고 3번만 복제됩니다.

## 더 보기

- [`영역 구성 표시`](show-zone-configurations.html)
- [`복제 영역`](configure-zone.html)
- [SQL 명령문](sql-statements.html)
- [테이블 파티셔닝](partitioning.html)
