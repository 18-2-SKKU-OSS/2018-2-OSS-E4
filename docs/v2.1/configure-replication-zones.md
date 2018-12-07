---
title: Configure Replication Zones
summary: In CockroachDB, you use replication zones to control the number and location of replicas for specific sets of data.
keywords: ttl, time to live, availability zone
toc: true
---

복제 영역을 통해 CockroachDB 클러스터에서 어떤 데이터가 이동하는지 제어할 수 있습니다. 특히, 다음 객체에 속한 데이터의 복제본 수와 위치를 제어하는 데 사용됩니다:

- 데이터베이스
- 테이블
- 행 ([엔터프라이즈 전용](enterprise-licensing.html))
- 인덱스 ([엔터프라이즈-전용](enterprise-licensing.html))
- 내부 시스템 데이터를 포함한 클러스터의 모든 데이터 ([기본 복제 영역을 통한](#view-the-default-replication-zone))


위의 각 개체에 대해 다음을 제어 할 수 있습니다:

- 클러스터를 통해 확산되는 각 범위의 복사본 수.
- 어떤 제약조건이 어떤 데이터에 적용되는가, (예 : "테이블 X의 데이터는 독일 데이터 센터에만 저장 가능하다").
- 범위의 최대 크기 (큰 범위가 분할되기 전에 얻는 방법).
- 폐기물을 수집하기 전에 얼마나 오래 데이터를 보관하는가. 
- <span class="version-tag">v2.1의 새로운 기능</span> 특정 범위의 리스 홀더를 배치하려는 경우, 예를 들어 `region=us-west`에 적어도 한 개 이상의 복제본을 보유하도록 이미 제한된 범위의 경우, 리스 홀더도 `region=us-west`에 넣으려고 시도한다.



이 페이지에서는 복제 영역의 작동 방식과 [`CONFIGURE ZONE`](configure-zone.html) 명령문을 사용하여 관리하는 방법에 대해 설명합니다.

{{site.data.alerts.callout_info}}
현재, `admin` 역할의 구성원만 복제 영역을 구성할 수 있습니다. 기본적으로, `root` 사용자는 `admin` 역할을 담당합니다.
{{site.data.alerts.end}}

## 개요

클러스터의 모든 [범위](architecture/overview.html#glossary)는 복제 영역의 일부입니다. 각 범위의 영역 구성은 모든 제약 조건이 준수되도록 범위가 클러스터에서 재조정될 때 고려된다. 


클러스터가 시작되면, 두 가지 범주의 복제 영역이 있습니다:

1. 내부 시스템 데이터에 적용되는 미리 구성된 복제 영역.
2. 나머지 클러스터에 적용되는 단일 기본 복제 영역.

이러한 미리 구성된 영역을 조정하고 필요에 따라 개별 데이터베이스, 테이블, 행 및 보조 인덱스의 영역을 추가할 수 있습니다. 행 및 보조 색인에 대한 영역을 추가하는 것은 [엔터프라이즈-전용](enterprise-licensing.html)입니다.

예를 들어, [기본 영역](#view-the-default-replication-zone)에 의존하여 클러스터 데이터의 대부분을 모든 데이터 센터로 분산할 수 있지만, [특정 데이터베이스에 대한 커스텀 복제 영역 생성](#create-a-replication-zone-for-a-database)을 통해 해당 데이터가 특정 데이터 센터 및/또는 지역에만 저장되어 있는지 확인할 수 있습니다. 

## 복제 영역 수준

클러스터에 [**테이블 데이터**](architecture/distribution-layer.html#table-data)에 대한 5개의 복제 영역 수준이 있으며, 최소에서 가장 세분화된 수준까지 나열됩니다:

수준 | 설명
------|------------
클러스터 | CockroachDB는 데이터베이스, 테이블 또는 특정-행 복제 영역에 의해 제약받지 않는 클러스터의 모든 테이블 데이터에 적용되는 미리 구성된 `.default` 복제 영역과 함께 제공됩니다. 이 영역은 조정할 수는 있지만 제거할 수는 없습니다. 자세한 내용은 [기본 복제 영역 보기](#view-the-default-replication-zone) 및 [기본 복제 영역 편집](#edit-the-default-replication-zone)을 참조하십시오.
데이터베이스 | 특정 데이터베이스에 복제 영역을 추가할 수 있습니다. 자세한 내용은 [데이터베이스의 복제 영역 생성](#create-a-replication-zone-for-a-database)을 참조하십시오.
테이블 |특정 테이블에 복제 영역을 추가할 수 있습니다. [테이블에 대한 복제 영역 생성](#create-a-replication-zone-for-a-table)을 참조하십시오.  
인덱스 ([엔터프라이즈-전용](enterprise-licensing.html)) | 테이블의 [보조 인덱스](indexes.html)는 자동으로 테이블의 복제 영역을 사용합니다. 그러나, 엔터프라이즈 라이센스를 사용하면, 보조 인덱스에 별도의 복제 영역을 추가 할 수 있습니다. 자세한 내용은 [보조 인덱스용 복제 영역 생성](#create-a-replication-zone-for-a-secondary-index)을 참조하십시오.
행 ([엔터프라이즈-전용](enterprise-licensing.html)) | [테이블 파티션 정의](partitioning.html)를 사용하여 테이블 또는 보조 인덱스의 특정 행에 대한 복제 영역을 추가할 수 있습니다. 자세한 내용은 [테이블 파티션용 복제 영역 생성](#create-a-replication-zone-for-a-table-or-secondary-index-partition)을 참조하십시오.

### 시스템 데이터

또한, CockroachDB는 시스템 범위라고 불리는 내부 [**시스템 데이터**](architecture/distribution-layer.html#monolithic-sorted-map-structure)를 저장합니다. 이 내부 시스템 데이터에 대한 복제 영역 레벨은 최소값에서 가장 세부적인 수준까지 두 복제 영역 수준이 있다. 

수준 | 설명
------|------------
클러스터 | 위에서 언급한 `.default` 복제 영역은 보다 특정한 복제 영역에 의해 제약받지 않는 모든 시스템 범위에도 적용됩니다.
시스템 범위 | CockroachDB는 "메타" 및 "활성" 시스템 범위에 대해 미리 구성된 복제 영역과 함께 제공됩니다. 필요한 경우, "시계열" 범위와 다른 "시스템" 범위에 대한 복제 영역을 추가할 수 있습니다. 자세한 내용은 [시스템 범위에 대한 복제 영역 생성](#create-a-replication-zone-for-a-system-range)을 참조하십시오.<br><br>또한 CockroachDB는 스키마 변경 및 백업과 같은 장기 실행 작업에 대한 메타 데이터를 저장하는 하나의 내부 테이블인 `system.jobs`에 대해 미리 구성된 복제 영역을 제공합니다. 히스토리 쿼리는 이 테이블에 대해 실행되지 않으며 테이블의 행이 자주 업데이트되므로, 미리 구성된 영역은 이 테이블에 기본값보다 낮은 `ttlseconds`를 제공합니다.

### 우선 순위

테이블 또는 시스템에 관계없이, 데이터를 복제할 때, CockroachDB는 항상 가장 세부적인 복제 영역을 사용합니다. 예를 들어, 사용자 데이터 조각의 경우:

1. 행에 대한 복제 영역이 있으면, CockroachDB가 그것을 사용합니다.
2. 적용 가능한 행 복제 영역이 없고 행이 보조 인덱스의 것이면, CockroachDB는 보조 인덱스 복제 영역을 사용합니다.
3. 행이 보조 인덱스에 있지 않거나 적용 가능한 보조 인덱스 복제 영역이 없는 경,우 CockroachDB는 테이블 복제 영역을 사용합니다.
4. 적용 가능한 테이블 복제 영역이 없으면, CockroachDB는 데이터베이스 복제 영역을 사용합니다.
5. 해당 데이터베이스 복제 영역이 없을 경우, CockroachDB는 `.default` 클러스터 전체 복제 영역을 사용합니다. 

{{site.data.alerts.callout_danger}}
{% include {{page.version.version}}/known-limitations/system-range-replication.md %}
{{site.data.alerts.end}}

## 복제 영역 관리

복제 영역을 [추가](#create-a-replication-zone-for-a-system-range), [수정](#edit-the-default-replication-zone), [재설정](#reset-a-replication-zone) 및 [제거](#remove-a-replication-zone)하려면 [`CONFIGURE ZONE`](configure-zone.html) 명령문을 사용하십시오.

### 복제 영역 변수

복제 영역을 설정하려면, [`ALTER ... CONFIGURE ZONE`] [명령문](sql-statements.html)을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> ALTER TABLE t CONFIGURE ZONE USING range_min_bytes = 0, range_max_bytes = 90000, gc.ttlseconds = 89999, num_replicas = 5, constraints = '[-region=west]';
~~~

{% include v2.1/zone-configs/variables.md %}

### 복제 제약 조건

복제본의 위치는 처음 추가 될 때와 클러스터 균형을 유지하기 위해 재조정될 때, 노드에 지정된 설명 속성과 영역 구성에 설정된 제약 조건 사이의 상호 작용을 기반으로 합니다.

{{site.data.alerts.callout_success}}다양한 시나리오에서 노드 속성 및 복제 제약 조건을 설정하는 방법에 대한 설명은 아래의 <a href="#scenario-based-examples">시나리오-기반 예제</a>를 참조하십시오.{{site.data.alerts.end}}



#### 노드에 할당된 설명 속성

[`cockroach start`] 명령으로 노드를 시작할 때, 다음과 같은 유형의 설명 속성을 할당할 수 있습니다:

속성 타입 | 설명
---------------|------------
**노드 지역성** | `--locality` 플래그를 사용하여, 노드의 지역성을 설명하는 임의의 키-값 쌍을 할당할 수 있습니다. 지역은 국가, 지역, 데이터 센터, 랙 등을 포함할 수 있습니다. 키-값 쌍은 가장 포괄적인 항목부터 가장 포괄적이 아닌 항목까지 정렬해야 합니다, (예 : 랙 이전의 데이터 센터 이전 국가) 키와 키-값 쌍의 순서는 모든 노드에서 동일해야 합니다. 일반적으로 적은 수의 쌍을 포함하는 것이 좋습니다. 예를 들어:<br><br>`--locality=region=east,datacenter=us-east-1`<br>`--locality=region=east,datacenter=us-east-2`<br>`--locality=region=west,datacenter=us-west-1`<br><br>CockroachDB는 우선 순위를 결정하는 순서로 지역에 따라 클러스터 전체에 고르게 복제본을 전파하려고 시도합니다. 그러나, 복제 영역을 사용하는 다양한 방법으로 데이터 복제본 위치에 영향을 미치는 데 지역성을 사용할 수 있다.<br><br>노드 간 대기 시간이 길면, CockroachDB는 지역성을 사용하여 현재 워크로드에 가까운 레인지 리스를 이동시켜 네트워크 왕복을 줄이고 읽기 성능을 향상시킵니다. 자세한 내용은 [팔로우 워크로드](demo-follow-the-workload.html)를 참조하십시오.
**노드 용량** | `--attrs` 플래그를 사용하여, 특수 하드웨어나 코어 수를 포함하는 노드 용량을 지정할 수 있습니다, 예를 들어:<br><br>`--attrs=ram:64gb`
**스토어 타입/용량** | `--store` 플래그의 `attrs` 필드를 사용하여, 디스크 타입이나 용량을 지정할 수 있습니다, 예를 :<br><br>`--store=path=/mnt/ssd01,attrs=ssd`<br>`--store=path=/mnt/hda1,attrs=hdd:7200rpm`

#### 제약 조건의 유형

위에서 언급한 노드 수준 및 저장소 수준의 설명 속성은 복제 영역에서 복제본 위치에 영향을 주는 다음 유형의 제약 조건으로 사용할 수 있습니다. 그러나, 다음 일반 지침에 유의하십시오:

- 지역성이 복제에 대한 유일한 고려 사항인 경우, 영역 구성에서 제약 조건을 지정하지 않고 노드에 지역성을 설정하는 것이 좋습니다. 제약 조건이 없는 경우, CockroachDB는 지역성에 따라 클러스터 전체에 복제본을 고르게 분산하려고 시도합니다.
- 필수 및 금지 제약 조건은, 예를 들어 데이터를 특정 국가 또는 특정 유형의 기계에 저장해야 하거나 저장하지 않아야하는 특수 상황에서 유용합니다.

속성 타입 | 설명 | 구문
----------------|-------------|-------
**요구됨** | 복제본을 배치할 때, 클러스터는 속성 또는 지역이 일치하는 노드/저장소만 고려합니다. 일치하는 노드/저장소가 없으면, 새 복제본이 추가되지 않습니다. | `+ssd`
**금지됨** | 복제본을 배치할 때, 클러스터는 일치하는 속성 또는 지역이 있는 노드/저장소를 무시합니다. 대체 노드/저장소가 없으면, 새 복제본이 추가되지 않습니다. | `-ssd`

#### 제약 조건의 범위

제약 조건은 영역의 모든 복제본에 적용되거나 다른 복제본이 다른 복제본에 적용되도록 지정할 수 있습니다. 즉, 각 복제본의 정확한 위치를 효과적으로 선택할 수 있습니다.


제약 범위 | 설명 | 구문
-----------------|-------------|-------
**모든 복제본** | JSON 배열 구문을 사용하여 지정된 제약 조건은 복제 영역의 일부인 모든 범위의 모든 복제본에 적용됩니다.| `constraints = '[+ssd, -region=west]'`
**복제본-당** | 여러 제약 조건 목록을 JSON 객체에 제공하여, 각 제약 조건 목록을 제약 조건이 적용되어야 하는 각 범위의 정수 개수의 복제본에 매핑 할 수 있습니다. <br><br>제한된 복제본의 총 수는 영역(`num_replicas`)의 복제본 총 수보다 클 수 없습니다.  그러나, 제한된 복제본의 총 수가 영역의 복제본 총 수보다 적으면, 제한이 없는 복제본은 모든 노드/저장소에서 허용됩니다. | `constraints: '{+ssd,-region=west: 2, +region=east: 1}'`

### 노드/복제본 권장 사항

프로덕션 배포를 위한 [클러스터 토폴로지](recommended-production-settings.html#cluster-topology) 권장 사항을 참조하십시오.

## 복제 영역 보기

[`SHOW ZONE CONFIGURATIONS`] 명령문을 사용하면 기존 복제 영역에 대한 세부 정보를 볼 수 있습니다.

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

    데이터베이스의 세 복제본 중 두 개는 `region = us-west1`에 저장되고 남은 복제본은 `region = us-central1`에 저장됩니다. 이를 통해, 어느 한 데이터 센터의 모든 장애를 극복할 수 있는 어플리케이션의 탄력성을 유지하면서 복제본 쿼럼이 West Coast에 있으므로 대기 시간이 짧은 읽기 및 쓰기를 제공합니다.

6. 국가 전체 데이터베이스에는 구성이 필요하지 않습니다. 클러스터는 기본적으로 데이터를 3번 복제하고 가능한 한 광범위하게 전파하도록 구성됩니다. 각 노드의 지역에 지정된 첫 번째 키-값 쌍이 각 노드의 지역성에서 가장 중요한 부분으로 간주되기 때문에, 가능한 한 광범위하게 데이터를 분산한다는 것은 세 가지 다른 영역 각각에 하나의 복제본을 넣는 것을 의미합니다.

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

7. 어플리케이션 2에서 사용되는 데이터베이스의 복제 영역을 구성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER DATABASE app2_db CONFIGURE ZONE USING constraints = '[+datacenter=us-2]';
    ~~~

8. 복제 영역을 보십시오:

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
- 그러나 속도와 가용성은 매우 자주 쿼리되는 특정 테이블과 해당 인덱스에 중요합니다. 따라서 테이블 및 보조 인덱스의 데이터를 SSD 드라이브가 있는 노드에서 5번 복제하기를 원할 것입니다. 

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

3. 데이터베이스 및 테이블을 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE db;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE db.important_table;
    ~~~

4. 보다 엄격하게 복제해야 하는 테이블의 복제 영역을 구성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER TABLE db.important_table CONFIGURE ZONE USING num_replicas = 5, constraints = '[+ssd]'
    ~~~

5. 복제 영역을 보십시오:

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

3. 기본 복제 영역을 구성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER RANGE default CONFIGURE ZONE USING num_replicas = 5;
    ~~~

4. 복제 영역을 보십시오:

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
