---
title: Decommission Nodes
summary: Permanently remove one or more nodes from a cluster.
toc: true
toc_not_nested: true
---

클러스터에서 하나 이상의 노드를 해제 및 영구적으로 제거하는 방법을 보여줍니다. 클러스터를 축소하거나 하드웨어 장애에 대응하는 등의 경우에서 이 페이지를 참고할 수 있습니다.

노드를 임시로 중지하는 방법에 대한 자세한 내용은 [노드 정지하기](stop-a-node.html)를 참조하십시오.

## 개요

### 작동 방식

노드를 해제할 때, CockroachDB는 노드가 진행 중인 요청을 완료하고, 새로운 요청을 거부하도록 합니다. 그리고 노드가 안전하게 종료될 수 있도록 모든 **범위 복제본** 및 **범위 임대**를 노드 밖으로 전송합니다.

기본 용어:

- **범위**: CockroachDB는 모든 사용자 데이터와 거의 모든 시스템 데이터를 키 값 쌍에 대한 정렬된 대형 맵에 저장한다. 이 키스페이스는 키스페이스의 연속된 청크인 "범위"로 나뉘어져 있어서 모든 키가 항상 단일 범위에서 찾을 수 있다.
- **범위 복제본:** CockroachDB는 각 범위를 복제하여 (기본적으로 3회) 다른 노드에 저장한다.
- **범위 임대** 각 범위에 대해, 복제본 중 하나가 "범위 임대"를 보유한다. "임대자"라고 하는 이 복제본은 범위의 모든 읽기 및 쓰기 요청을 수신하고 조정하는 복제본이다.

### 고려사항

제거할 노드에서 복제본을 가져올 수 있는 노드가 있는지 확인하십시오. 사용 가능한 다른 노드가 없는 경우 해체 프로세스는 무기한 중단됩니다. 위의 [Example scenarios](#examples)를 참조하십시오.

### 예제

#### 3-노드 클러스터, 3-way 복제

이 시나리오에서는 각 범위가 서로 다른 노드에 있는 각 복제본과 함께 3회 복제됩니다.

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-scenario1.1.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

이때 노드 해제를 시도하면 클러스터가 사용 중지된 노드의 복제본을 이미 각 범위의 복제본이 있는 다른 2개 노드로 이동시킬 수 없으므로 프로세스가 무기한 중단됩니다.

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-scenario1.2.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

노드를 성공적으로 폐기하려면 먼저 4번째 노드를 추가해야 합니다.

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-scenario1.3.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

#### 5-노드 클러스터, 3-way 복제

위의 시나리오와 같이 이 시나리오에서는 각 범위가 서로 다른 노드의 각 복제본과 함께 3회 복제됩니다.

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-scenario2.1.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

이번에는 클러스터가 범위 복제본을 두 배로 늘리지 않아도 노드의 복제본을 다른 노드로 이동할 수 있기 때문에 프로세스가 성공적으로 실행됩니다.

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-scenario2.2.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

#### 5-노드 클러스터, 특정 테이블에 대한 5-way 복제

이 시나리오에서는 [복제 영역 사용자 지정](configure-replication-zones.html#create-a-replication-zone-for-a-table)을 사용하여 특정 테이블(범위 6)을 5번 복제하도록 지정합니다. 이때 특정 테이블을 제외한 다른 모든 데이터는 기본값 그대로 3회 복제됩니다.

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-scenario3.1.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

이때 노드 해제를 시도하면 클러스터는 성공적으로 범위 6를 제외한 모든 범위의 균형을 재조정할 수 있습니다. 그러나 범위 6에는 5개의 복제본이 필요하며(테이블별 복제 영역 기반), CockroachDB는 단일 노드에서 어떤 범위도 복제본을 1개 이상 가지는 것을 허용하지 않으므로 해체 프로세스는 무한정 중단됩니다.

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-scenario3.2.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

노드를 성공적으로 폐기하려면 6번째 노드를 추가하십시오.

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-scenario3.3.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

## 단일 노드 제거(활성)

<div class="filters clearfix">
  <button style="width: 15%" class="filter-button" data-scope="secure">Secure</button>
  <button style="width: 15%" class="filter-button" data-scope="insecure">Insecure</button>
</div>

### 시작하기 전에

제거할 노드에서 복제본을 가져올 수 있는 노드가 있는지 확인하십시오. 위의 [Example scenarios](#examples)를 참조하십시오.

### 1단계. 해제 전 노드 확인

관리 UI를 열고 왼쪽의 **Metrics**를 클릭하고 **복제본** 대시보드를 선택한 다음 **스토어 당 복제본** **스토어 당 임대자** 그래프 위로 마우스를 옮기십시오.

<div style="text-align: center;"><img src="{{ 'images/v2.1/before-decommission2.png' | relative_url }}" alt="Decommission a single live node" style="border:1px solid #eee;max-width:100%" /></div>

<div style="text-align: center;"><img src="{{ 'images/v2.1/before-decommission1.png' | relative_url }}" alt="Decommission a single live node" style="border:1px solid #eee;max-width:100%" /></div>

### 2단계. 노드 해제 및 삭제

노드가 실행 중인 머신에 대해 SSH 하고 [`cockroach quit`](stop-a-node.html)명령어를 `--decommission` 플래그 및 기타 필수 플래그와 함께 실행하십시오. 

<div class="filter-content" markdown="1" data-scope="secure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach quit --decommission --certs-dir=certs --host=<address of node to remove>
~~~
</div>

<div class="filter-content" markdown="1" data-scope="insecure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach quit --decommission --insecure --host=<address of node to remove>
~~~
</div>

해제 상태가 `stderr`로 출력되는 것을 볼 수 있습니다.

~~~
 id | is_live | replicas | is_decommissioning | is_draining  
+---+---------+----------+--------------------+-------------+
  4 |  true   |       73 |        true        |    false     
(1 row)
~~~


노드가 완전히 해제되고 중지되면, 확인 메시지를 받을 수 있습니다.

~~~
 id | is_live | replicas | is_decommissioning | is_draining  
+---+---------+----------+--------------------+-------------+
  4 |  true   |        0 |        true        |    false     
(1 row)

No more data reported on target nodes. Please verify cluster health before removing the nodes.
ok
~~~

### 3단계. 해제 후 노드 및 클러스터 확인

다시 관리 UI로 가서 **복제본** 대시보드에서 **스토어 당 복제본** 및 **스토어 당 임대자** 그래프 위에 마우스를 올려 놓으십시오. 해제한 노드의 경우 카운트는 0이어야 합니다:

<div style="text-align: center;"><img src="{{ 'images/v2.1/after-decommission1.png' | relative_url }}" alt="Decommission a single live node" style="border:1px solid #eee;max-width:100%" /></div>

<div style="text-align: center;"><img src="{{ 'images/v2.1/after-decommission2.png' | relative_url }}" alt="Decommission a single live node" style="border:1px solid #eee;max-width:100%" /></div>

그런 다음 **Overview** 페이지에서 **노드 목록**을 보고 제거한 노드가 아닌 나머지 노드가 정상인지 확인하십시오(녹색):

<div style="text-align: center;"><img src="{{ 'images/v2.1/cluster-status-after-decommission1.png' | relative_url }}" alt="Decommission a single live node" style="border:1px solid #eee;max-width:100%" /></div>

약 5분 후에 **해제된 노드** 아래에 제거된 노드가 표시될 것입니다.

<div style="text-align: center;"><img src="{{ 'images/v2.1/cluster-status-after-decommission2.png' | relative_url }}" alt="Decommission a single live node" style="border:1px solid #eee;max-width:100%" /></div>

이 시점에서 노드가 활성 상태인 시간 범위를 보지 않는 한 제거된 노드는 더 이상 타임시리즈 그래프에 나타나지 않습니다. 그리고 제거한 노드는  **해제된 노드** 목록에서 절대 사라지지 않습니다.

또한 노드가 재시작되어도 클라이언트 연결을 수락하지 않고, 클러스터가 데이터를 재조정하지 않습니다. 따라서 클러스터가 노드를 다시 활용하도록 하려면 노드를 [다시 활성화](#recommission-nodes) 해야 합니다.

## 단일 노드 제거 (비활성)

<div class="filters clearfix">
  <button style="width: 15%" class="filter-button" data-scope="secure">Secure</button>
  <button style="width: 15%" class="filter-button" data-scope="insecure">Insecure</button>
</div>

노드가 5분 동안 비활성 상태가 되면, CockroachDB는 노드의 범위 복제본과 범위 임대자를 사용 가능한 실시간 노드로 자동으로 전송합니다. 하지만 다시 노드를 재시작하면 클러스터가 복제본과 임대자의 균형을 재조정합니다.

노드가 다시 온라인 상태가 되어도 클러스터가 비활성 노드로 데이터를 재조정하지 않도록 하려면 다음을 수행하십시오.

### 1단계. 비활성 노드 ID 식별

관리 UI를 열고 **Node List** 보기를 선택하십시오. **Dead Nodes**에 나열된 노드의 ID를 기록해 두십시오.

<div style="text-align: center;"><img src="{{ 'images/v2.1/remove-dead-node1.png' | relative_url }}" alt="Decommission a single dead node" style="border:1px solid #eee;max-width:100%" /></div>

### 2단계. 비활성 노드 표시

클러스터에 있는 임의의 활성화 노드에 대해 SSH 하고 [`cockroach node decommission`](view-node-details.html) 명령을 실행하여 노드를 공식적으로 해제하십시오.


<div class="filter-content" markdown="1" data-scope="secure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach node decommission 4 --certs-dir=certs --host=<address of live node>
~~~
</div>

<div class="filter-content" markdown="1" data-scope="insecure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach node decommission 4 --insecure --host=<address of live node>
~~~
</div>

~~~
 id | is_live | replicas | is_decommissioning | is_draining  
+---+---------+----------+--------------------+-------------+
  4 |  false  |        0 |        true        |    true      
(1 row)

No more data reported on target nodes. Please verify cluster health before removing the nodes.
~~~

**Nodes List** 페이지로 돌아가면 약 5분 후에 노드가 -**Dead Nodes**에서 **Decommissioned Nodes** 목록으로 이동하는 것을 볼 수 있습니다. 이 시점에서 노드가 활성 상태인 시간 범위를 보지 않는 한 노드가 더 이상 타임시리즈 그래프에 나타나지 않습니다. 그리고 노드가 **Decommissioned Nodes** 목록에서 절대 사라지지 않습니다.

<div style="text-align: center;"><img src="{{ 'images/v2.1/cluster-status-after-decommission2.png' | relative_url }}" alt="Decommission a single live node" style="border:1px solid #eee;max-width:100%" /></div>

또한 노드가 재시작되어도 클라이언트 연결을 수락하지 않고, 클러스터가 데이터를 재조정하지 않습니다. 따라서 클러스터가 노드를 다시 활용하도록 하려면 노드를 [다시 활성화](#recommission-nodes) 해야 합니다.

## 여러 개의 노드 제거

<div class="filters clearfix">
  <button style="width: 15%" class="filter-button" data-scope="secure">Secure</button>
  <button style="width: 15%" class="filter-button" data-scope="insecure">Insecure</button>
</div>

### 시작하기 전에

제거할 노드들에서 복제본을 가져올 수 있는 노드들이 있는지 확인하십시오. 위의 [Example scenarios](#examples)를 참조하십시오.

### 1단계. 해제할 노드 ID 식별

관리 UI를 열고 **Node List**보기를 선택하거나 왼쪽의  **Metrics**로 이동한 후 **Summary** 영역에서 **View nodes list** 를 클릭하십시오. 해제할 노드의 ID를 확인하십시오.

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-multiple1.png' | relative_url }}" alt="Decommission multiple nodes" style="border:1px solid #eee;max-width:100%" /></div>

### 2단계. 해제 전 노드 확인

**복제본** 대시보드를 선택한 다음 **스토어 당 복제본** **스토어 당 임대자** 그래프 위로 마우스를 옮기십시오.

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-multiple2.png' | relative_url }}" alt="Decommission multiple nodes" style="border:1px solid #eee;max-width:100%" /></div>

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-multiple3.png' | relative_url }}" alt="Decommission multiple nodes" style="border:1px solid #eee;max-width:100%" /></div>

### 3단계. 노드 해제

클러스터에 있는 임의의 활성화 노드에 대해 SSH 하고 해제할 노드의 ID와 함께 [`cockroach node decommission`](view-node-details.html) 명령을 실행하여 노드를 공식적으로 해제하십시오.

<div class="filter-content" markdown="1" data-scope="secure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach node decommission 4 5 --certs-dir=certs --host=<address of live node>
~~~
</div>

<div class="filter-content" markdown="1" data-scope="insecure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach node decommission 4 5 --insecure --host=<address of live node>
~~~
</div>

해제 상태가 `stderr`로 출력되는 것을 볼 수 있습니다.

~~~
 id | is_live | replicas | is_decommissioning | is_draining  
+---+---------+----------+--------------------+-------------+
  4 |  true   |       18 |        true        |    false     
  5 |  true   |       16 |        true        |    false     
(2 rows)
~~~

노드가 완전히 해제되고 중지되면, 확인 메시지를 받을 수 있습니다.

~~~
 id | is_live | replicas | is_decommissioning | is_draining  
+---+---------+----------+--------------------+-------------+
  4 |  true   |        0 |        true        |    false     
  5 |  true   |        0 |        true        |    false     
(2 rows)

No more data reported on target nodes. Please verify cluster health before removing the nodes.
~~~

### 4단계. 해제 후 노드 및 클러스터 확인

다시 관리 UI로 가서 **복제본** 대시보드에서 **스토어 당 복제본** 및 **스토어 당 임대자** 그래프 위에 마우스를 올려 놓으십시오. 해제한 노드의 경우 카운트는 0이어야 합니다:

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-multiple4.png' | relative_url }}" alt="Decommission multiple nodes" style="border:1px solid #eee;max-width:100%" /></div>

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-multiple5.png' | relative_url }}" alt="Decommission multiple nodes" style="border:1px solid #eee;max-width:100%" /></div>

그런 다음 **Summary** 영역에서 **View nodes list**을 클릭하여 제거한 노드가 0개의 복제본을 가지며, 나머지 노드는 모두 정상인 것을 확인하십시오(녹색):

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-multiple6.png' | relative_url }}" alt="Decommission multiple nodes" style="border:1px solid #eee;max-width:100%" /></div>

약 5분 후에 노드가 **Decommissioned Nodes** 목록으로 이동하는 것을 볼 수 있으며, 노드가 활성 상태인 시간 범위를 보기 전에는 노드가 더 이상 타임시리즈 그래프에 나타나지 않습니다. 그리고 제거된 노드는 **Decommissioned Nodes** 목록에서 절대 사라지지 않습니다.

<div style="text-align: center;"><img src="{{ 'images/v2.1/decommission-multiple7.png' | relative_url }}" alt="Decommission multiple nodes" style="border:1px solid #eee;max-width:100%" /></div>

### 5단계. 해제된 노드 제거

이 시점에서 해제된 노드가 활성 상태이더라도 클러스터는 이들 노드로의 데이터 균형을 재조정하지 않으며, 노드는 어떠한 클라이언트 연결도 허용하지 않습니다. 그러나 클러스터에서 노드를 공식적으로 제거하려면 노드를 중지해야 합니다.

해제된 각 노드에 대해 해당 노드를 실행하는 머신에 SSH 하고 `cockroach quit` 명령을 실행하십시오.

<div class="filter-content" markdown="1" data-scope="secure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach quit --certs-dir=certs --host=<address of decommissioned node>
~~~
</div>

<div class="filter-content" markdown="1" data-scope="insecure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach quit --insecure --host=<address of decommissioned node>
~~~
</div>

## 노드 재활성화

<div class="filters clearfix">
  <button style="width: 15%" class="filter-button" data-scope="secure">Secure</button>
  <button style="width: 15%" class="filter-button" data-scope="insecure">Insecure</button>
</div>

실수로 노드를 해제했거나 해제된 노드가 클러스터에 활성화된 노드로서 다시 결합하도록 하려면 다음을 수행하십시오.

### Step 1. 해제된 노드 ID 식별

관리 UI를 열고 **Node List** 보기를 선택하십시오. **Decommissioned Nodes**에 나열된 노드의 ID를 기록하십시오.

<div style="text-align: center;"><img src="{{ 'images/v2.1/cluster-status-after-decommission2.png' | relative_url }}" alt="Decommission a single dead node" style="border:1px solid #eee;max-width:100%" /></div>

### Step 2. 노드 재활성화

활성 노드 중 하나에 SSH 하고 재활성화할 노드의 ID와 함께 [`cockroach node recommission`](view-node-details.html) 명령을 실행하십시오.

<div class="filter-content" markdown="1" data-scope="secure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach node recommission 4 --certs-dir=certs --host=<address of live node>
~~~
</div>

<div class="filter-content" markdown="1" data-scope="insecure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach node recommission 4 --insecure --host=<address of live node>
~~~
</div>

~~~
 id | is_live | replicas | is_decommissioning | is_draining  
+---+---------+----------+--------------------+-------------+
  4 |  false  |        0 |       false        |    true      
(1 row)
The affected nodes must be restarted for the change to take effect.
~~~

### Step 3. 재활성화 노드 다시 시작

재활성화된 각 노드에 해당하는 머신에 SSH 하고 `cockroach start` 명령어를 실행합니다.

<div class="filter-content" markdown="1" data-scope="secure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach start --certs-dir=certs --advertise-addr=<address of node to restart> --join=<address of node 1> --background
~~~
</div>

<div class="filter-content" markdown="1" data-scope="insecure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach start --insecure --advertise-addr=<address of node to restart> --join=<address of node 1> --background
~~~
</div>

**Nodes List** 페이지의 **Live Nodes**에 나열된 재활성 노드를 확인할 수 있으며, 몇 분 후 복제본이 해당 노드로 재조정되는 것을 볼 수 있습니다.

## 재활성화 노드의 상태 확인

해제되는 노드의 진행 상황을 확인하려면, `--decommission` 플래그를 사용하여 `cockroach node status` 명령을 실행하십시오.

<div class="filters clearfix">
  <button style="width: 15%" class="filter-button" data-scope="secure">Secure</button>
  <button style="width: 15%" class="filter-button" data-scope="insecure">Insecure</button>
</div><br>

<div class="filter-content" markdown="1" data-scope="secure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach node status --decommission --certs-dir=certs --host=<address of any live node>
~~~
</div>

<div class="filter-content" markdown="1" data-scope="insecure">
{% include copy-clipboard.html %}
~~~ shell
$ cockroach node status --decommission --insecure --host=<address of any live node>
~~~
</div>

~~~
 id |        address         |  build  |            started_at            |            updated_at            | is_available | is_live | gossiped_replicas | is_decommissioning | is_draining  
+---+------------------------+---------+----------------------------------+----------------------------------+--------------+---------+-------------------+--------------------+-------------+
  1 | 165.227.60.76:26257    | 91a299d | 2018-10-01 16:53:10.946245+00:00 | 2018-10-02 14:04:39.280249+00:00 |         true |  true   |                26 |       false        |    false     
  2 | 192.241.239.201:26257  | 91a299d | 2018-10-01 16:53:24.22346+00:00  | 2018-10-02 14:04:39.415235+00:00 |         true |  true   |                26 |       false        |    false     
  3 | 67.207.91.36:26257     | 91a299d | 2018-10-01 17:34:21.041926+00:00 | 2018-10-02 14:04:39.233882+00:00 |         true |  true   |                25 |       false        |    false     
  4 | 138.197.12.74:26257    | 91a299d | 2018-10-01 17:09:11.734093+00:00 | 2018-10-02 14:04:37.558204+00:00 |         true |  true   |                25 |       false        |    false     
  5 | 174.138.50.192:26257   | 91a299d | 2018-10-01 17:14:01.480725+00:00 | 2018-10-02 14:04:39.293121+00:00 |         true |  true   |                 0 |        true        |    false          
(5 rows)
~~~

## 더 알아보기

- [임시로 노드 정지하기](stop-a-node.html)
