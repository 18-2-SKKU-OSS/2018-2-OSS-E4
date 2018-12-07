---
title: Enable the Node Map
summary: Learn how to enable the node map in the Admin UI.
toc: true
---

**노드 맵**은 세계지도에 노드 지역을 플로팅하여 다중 지역 클러스터의 지리적 구성을 시각화합니다. 또한 **노드 맵**은 클러스터 상태와 성능을 모니터링하고 문제 해결을 위해 개별 노드로 드릴다운할 수 있는 기능을 통해 실시간 클러스터 메트릭스를 제공합니다. 

이 페이지는 **노드 맵**을 설정하고 활성화하는 과정을 안내합니다.

{{site.data.alerts.callout_info}}<b>노드 맵</b>은 <a href="enterprise-licensing.html"> 엔터프라이즈 전용 </a> 기능입니다. 그러나, <a href="https://www.cockroachlabs.com/pricing/request-a-license/"> 평가판 라이센스를 요청</a>하여 시험 사용해 볼 수 있습니다. {{site.data.alerts.end}}

<img src="{{ 'images/v2.1/admin-ui-node-map.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## 노드 맵 설정 및 활성화

**노드 맵**을  가능하게 하려면, 올바른`--locality` 플래그로 클러스터를 시작하고 각 지역에 대한 위도와 경도를 지정해야 합니다.

{{site.data.alerts.callout_info}}<b>노드 맵</b>은 올바른 <code>--지역성</code> 플래그로 시작하고 모든 지역에는 해당하는 위도와 경도가 할당될 때까지 표시되지 않는다. {{site.data.alerts.end}}


다음과 같은 구성의 4 노드 지리적-분산 클러스터 시나리오를 고려하십시오:

|  노드 | 지역 | 데이터센터 |
|  ------ | ------ | ------ |
|  노드1 | us-east-1 | us-east-1a |
|  노드2 | us-east-1 | us-east-1b |
|  노드3 | us-west-1 | us-west-1a |
|  노드4 | eu-west-1 | eu-west-1a |

### 1단계. CockroachDB 버전이 2.0 이상인지 확인하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach version
~~~

~~~
Build Tag:   {{page.release_info.version}}
Build Time:  {{page.release_info.build_time}}
Distribution: CCL
Platform:     darwin amd64 (x86_64-apple-darwin13)
Go Version:   go1.10
C Compiler:   4.2.1 Compatible Clang 3.8.0 (tags/RELEASE_380/final)
Build SHA-1:  367ad4f673b33694df06caaa2d7fc63afaaf3053
Build Type:   release
~~~

어떤 노드가 이전 버전을 실행중인 경우, [CockroachDB v2.0으로 업그레이드](upgrade-cockroach-version.html)하십시오. 

### 2단계. 올바른 `--locality` 플래그로 노드를 시작하십시오.

올바른 `--locality` 플래그로 새로운 클러스터를 시작하려면:

노드 1 시작:

{% include copy-clipboard.html %}
~~~
$ cockroach start \
--insecure \
--locality=region=us-east-1,datacenter=us-east-1a  \
--advertise-addr=<node1 address> \
--cache=.25 \
--max-sql-memory=.25 \
--join=<node1 address>,<node2 address>,<node3 address>,<node4 address>
~~~

노드 2 시작:

{% include copy-clipboard.html %}
~~~
$ cockroach start \
--insecure \
--locality=region=us-east-1,datacenter=us-east-1b \
--advertise-addr=<node2 address> \
--cache=.25 \
--max-sql-memory=.25 \
--join=<node1 address>,<node2 address>,<node3 address>,<node4 address>
~~~

노드 3 시작:

{% include copy-clipboard.html %}
~~~
$ cockroach start \
--insecure \
--locality=region=us-west-1,datacenter=us-west-1a \
--advertise-addr=<node3 address> \
--cache=.25 \
--max-sql-memory=.25 \
--join=<node1 address>,<node2 address>,<node3 address>,<node4 address>
~~~

노드 4 시작:

{% include copy-clipboard.html %}
~~~
$ cockroach start \
--insecure \
--locality=region=eu-west-1,datacenter=eu-west-1a \
--advertise-addr=<node4 address> \
--cache=.25 \
--max-sql-memory=.25 \
--join=<node1 address>,<node2 address>,<node3 address>,<node4 address>
~~~

[`cockroach init`](initialize-a-cluster.html) 명령을 사용하여 클러스터의 일회 초기화를 수행하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach init --insecure --host=<address of any node>
~~~

[Admin UI에 접근](admin-ui-access-and-navigate.html#access-the-admin-ui)합니다. 다음 페이지가 표시됩니다:

<img src="{{ 'images/v2.1/admin-ui-node-map-before-license.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

### 3단계. [엔터프라이즈 라이센스 설정](enterprise-licensing.html) 및 Admin UI 새로 고침

다음 페이지가 표시되어야 합니다:

<img src="{{ 'images/v2.1/admin-ui-node-map-after-license.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

### 4단계. 지역에 대한 위도와 경도 설정

빌트인 SQL 클라이언트 시작:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure --host=<address of any node>
~~~

각 영역의 대략적인 위도와 경도를 `system.locations` 테이블에 삽입하십시오:

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO system.locations VALUES
  ('region', 'us-east-1', 37.478397, -76.453077),
  ('region', 'us-west-1', 38.837522, -120.895824),
  ('region', 'eu-west-1', 53.142367, -7.692054);
~~~

{{site.data.alerts.callout_info}}<b>노드 맵</b>은 모든 지역에 해당 위도와 경도가 할당될 때까지 표시되지 않습니다. {{site.data.alerts.end}}


AWS, Azure 및 Google Cloud 지역의 위도와 경도에 대해서는, [참조용 위치 좌표](#location-coordinates-for-reference)를 보십시오. 

### 5단계. 노드 맵 보기

[ **개요 페이지**를 엽니다](admin-ui-access-and-navigate.html) 그리고 **View** 드롭다운 메뉴에서 **Node Map**를 선택하십시오. **노드 맵**이 표시됩니다:

<img src="{{ 'images/v2.1/admin-ui-node-map-complete.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

### 6단계. 노드 맵 탐색

`us-east-1` 지역의 `us-east-1a` 데이터 센터에 있는 노드 2로 이동한다고 가정해 봅시다:

1. **노드 맵**에서 **region=us-east-1**로 표시된 맵 구성 요소를 클릭하십시오. 데이터 센터 보기가 표시됩니다.
2. **datacenter=us-east-1a**로 표시된 데이터 센터 구성 요소를 클릭하십시오. 개별 노드 구성 요소가 표시됩니다.
3. 클러스터 뷰로 돌아가려면, **노드 맵** 상단의 빵 부스러기 흔적에서 **클러스터**를 클릭하거나, **Up to region=us-east-1**을 클릭하고 그런 다음 **노드 맵**의 왼쪽 하단에있는 **클러스터까지**를 클릭하십시오.

<img src="{{ 'images/v2.1/admin-ui-node-map-navigation.gif' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## 노드 맵 문제 해결

### 노드 맵이 표시되지 않음

**노드 맵**은 모든 노드에 지역이 있고 해당 위도와 경도가 할당 될 때까지 표시되지 않습니다. 모든 노드에 할당된 위도와 경도는 물론 지역을 할당했는지 확인하려면, Admin UI에서 지역성 디버그 페이지 (`https://<address of any node>:8080/#/reports/localities`)로 이동하십시오 

지역성 디버그 페이지는 다음을 표시합니다:

- `--locality` 플래그를 사용하여 노드를 시작하는 동안 설정한 지역성 구성.
- 각 지역에 해당하는 노드.
- 각 지역/노드에 대한 위도와 경도 좌표

페이지에서, 모든 노드에 위도/경도 좌표뿐만 아니라 지역이 할당되어 있는지 확인하십시오.

### 모든 지역 수준에 노드 맵이 표시되지 않음

**노드 맵**은 위도/경도 좌표가 할당된 지역 수준에서만 표시됩니다:

- 지역 레벨에서 위도/경도 좌표를 할당하면, **노드 맵**은 세계 지도상의 지역을 보여줍니다. 그러나, 데이터 센터로 드릴 다운하고 개별 노드로 더 이동하면, 세계 지도가 사라지고 데이터 센터/노드는 원형 레이아웃으로 표시된다.
- 데이터 센터 수준에서 위도/경도 좌표를 할당하는 경우, **노드 맵**에는 데이터 센터에 할당된 동일한 위치에 단일 데이터 센터가 있는 영역이 표시되고, 여러 데이터 센터를 가진 영역은 해당 지역의 데이터 센터 좌표 중앙에 표시된다. 데이터 센터 수준까지 드릴 다운하면, **노드 맵**에 할당된 좌표의 데이터 센터가 표시됩니다. 개별 노드로 추가 드릴 다운하면 노드가 원형 레이아웃으로 표시됩니다.

**노드 맵**에서 보려는 지역 수준의 [위도/경도 좌표를 지정](#step-4-set-the-latitudes-and-longitudes-for-the-localities)합니다.

## 알려진 제한 사항

### 지역에 위도/경도 좌표를 할당할 수 없음

{% include {{ page.version.version }}/known-limitations/node-map.md %}

### **사용된 용량** 구성된 용량을 초과함

{% include v2.1/misc/available-capacity-metric.md %}

## 참조용 위치 좌표

### AWS 로케이션

{% include {{ page.version.version }}/misc/aws-locations.md %}

### Azure 로케이션

{% include {{ page.version.version }}/misc/azure-locations.md %}

### 구글 클라우드 로케이션

{% include {{ page.version.version }}/misc/gce-locations.md %}
