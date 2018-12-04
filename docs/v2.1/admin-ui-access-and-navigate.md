---
title: Use the CockroachDB Admin UI
summary: Learn how to access and navigate the Admin UI.
toc: true
---

빌트인 Admin UI를 통해 클러스터의 상태, 구성 및 작업에 대한 정보를 제공하여 CockroachDB를 모니터링하고 문제를 해결할 수 있습니다.

## Admin UI 접근

시큐어 클러스터의 경우, 승인된 사용자만 [Admin UI에 접근하고 볼 수 있습니다](#accessing-the-admin-ui-for-a-secure-cluster).

클러스터의 모든 노드에서 Admin UI에 액세스 할 수 있습니다.

Admin UI는 IP 주소/호스트이름 및 포트 집합에서 [각 노드를 시작](start-a-node.html)할 때, `--http-addr` 플래그를 통해 도달할 수 있다. 예를 들어, 인시큐어 클러스터의 경우 `http://<address from --http-addr>:<port from --http-addr>`, 시큐어 클러스터의 경우 `https://<address from --http-addr>:<port from --http-addr>`.

노드를 시작할 때 `--http-addr`가 지정되지 않으면, Admin UI는 `--listen-addr` 플래그와 포트 `8080`을 통해 설정된 IP 주소/호스트이름에서 도달할 수 있습니다.

클러스터 배포 컨텍스트에서 Admin UI에 액세스하는 방법에 대한 자세한 내용은 [로컬 클러스터 시작](start-a-local-cluster.html) 및 [수동 배포](manual-deployment.html)를 참조하십시오.

### 보안 클러스터의 Admin UI 접근

시큐어 클러스터의 Admin UI에 접근해야 하는 각 사용자에 대해, [비밀번호와 함께 사용자를 생성](create-user.html)하십시오. Admin UI에 접근하면, 사용자는 사용자 이름과 암호를 입력해야하는 로그인 화면을 볼 수 있습니다.

{{site.data.alerts.callout_info}}
이 로그인 정보는 클러스터의 다른 데이터처럼 복제된 시스템 테이블에 저장됩니다. 시스템 테이블 데이터의 복제본이 있는 노드의 대다수가 다운되면, Admin UI에서 사용자가 잠깁니다.
{{site.data.alerts.end}}

Admin UI에서 로그아웃하려면 왼쪽 탐색 모음 하단의 **Log Out** 링크를 클릭하십시오.

## Admin UI 탐색

왼쪽 탐색 모음을 사용하면, [클러스터 개요 페이지](admin-ui-access-and-navigate.html), [클러스터 메트릭 대시 보드](admin-ui-overview.html), [데이터베이스 페이지](admin-ui-statements-page.html), [명령문 페이지](admin-ui-jobs-page.html), [작업 페이지](admin-ui-jobs-page.html) 및 [고급 디버깅 페이지](admin-ui-debug-pages.html)로 이동할 수 있습니다.

각 페이지의 기본 패널 디스플레이가 변경됩니다:

페이지 | 메인 패널 구성 요소
-----------|------------
클러스터 개요 | <ul><li>[클러스터 개요 패널](admin-ui-cluster-overview-page.html)</li><li>[노드 리스트](admin-ui-cluster-overview-page.html#node-list) </li> <li>[기업 사용자](enterprise-licensing.html)는 활성화하고 [노드 맵](admin-ui-cluster-overview-page.html#node-map-enterprise) 보기로 전환할 수 있습니다. </li></ul>
클러스터 메트릭 | <ul><li>[시계열 그래프](admin-ui-access-and-navigate.html#cluster-metrics)</li><li>[요약 패널](admin-ui-access-and-navigate.html#summary-panel)</li><li>[이벤트 리스트](admin-ui-access-and-navigate.html#events-panel)</li></ul>
데이터베이스 | [데이터베이스](admin-ui-databases-page.html)에 있는 테이블 및 보조금에 대한 정보.
명령문 | 클러스터에서 실행중인 SQL [명령문](admin-ui-statements-page.html)에 대한 정보.
작업 | 현재 활성화된 모든 스키마 변경 및 백업/복원 [작업](admin-ui-jobs-page.html)에 대한 정보.
고급 디버깅 | 고급 모니터링 및 문제 해결 [보고서](admin-ui-debug-pages.html). 이 페이지는 실험적입니다. 문제를 발견하면, [이 채널](https://www.cockroachlabs.com/community/)을 통해 우리에게 알려주세요.

### 클러스터 메트릭

**Cluster Metrics** 대시 보드에는 데이터 트렌드를 시각화하고 모니터링하는 데 유용한 시계열 그래프가 표시됩니다. 시계열 그래프에 액세스하려면, 왼쪽의 **Metrics**를 클릭하십시오.

각 그래프 위로 마우스를 가져 가면 실제 특정 시점 값을 볼 수 있습니다.

<img src="{{ 'images/v2.1/admin_ui_hovering.gif' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

{{site.data.alerts.callout_info}}
기본적으로, CockroachDB는 지난 30일 동안의 시계열 측정 항목을 저장하지만, 시계열 저장소에 대한 간격을 줄일 수 있습니다. 또는, 시계열 모니터링을 위해 [Prometheus](monitor-cockroachdb-with-prometheus.html)와 같은 타사 도구를 독점적으로 사용하는 경우, 시계열 저장소를 완전히 비활성화할 수 있습니다. 자세한 내용은, 이 [FAQ](operational-faqs.html#can-i-reduce-or-disable-the-storage-of-timeseries-data)를 참조하십시오.
{{site.data.alerts.end}}

#### 시간 범위 변경

시간대를 클릭하여 시간 범위를 변경할 수 있습니다.
<img src="{{ 'images/v2.1/admin-ui-time-range.gif' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

{{site.data.alerts.callout_info}}Admin UI는 클러스터에 대해 다른 시간대를 설정한 경우에도, UTC로 시간을 표시합니다.{{site.data.alerts.end}}

#### 단일 노드에 대한 메트릭 보기

기본적으로, 시계열 패널에는 전체 클러스터에 대한 메트릭이 표시됩니다. 개별 노드의 메트릭을 보려면, **Graph** 드롭 다운 목록에서 노드를 선택하십시오.
<img src="{{ 'images/v2.1/admin-ui-single-node.gif' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

### 요약 패널

**클러스터 메트릭** 대시 보드에는 주요 메트릭의 **요약** 패널이 표시됩니다. **요약** 패널을 보려면, 왼쪽의 **Metrics**를 클릭하십시오.

<img src="{{ 'images/v2.1/admin_ui_summary_panel.png' | relative_url }}" alt="CockroachDB Admin UI 요약 패널" style="border:1px solid #eee;max-width:40%" />

**요약** 패널은 다음과 같은 메트릭을 제공합니다:

메트릭 | 설명
--------|----
전체 노드 | 클러스터의 총 노드 수입니다. <a href='admin-ui-cluster-overview-page.html#decommissioned-nodes'>해체된 노드</a>는 총 노드 수에 포함되지 않습니다. <br><br>[**View nodes list**](admin-ui-cluster-overview-page.html#node-list)를 클릭하여 노드 세부 정보를 자세히 확인할 수 있습니다.
데드 노드 | 클러스터의 [데드 노드](admin-ui-cluster-overview-page.html#dead-nodes) 수입니다.
사용된 용량 | 모든 노드에 할당된 총 저장소 용량의 백분율로 사용되는 저장소 용량입니다.
사용할 수 없는 범위 | 클러스터에서 사용할 수 없는 범위의 수입니다. 0이 아닌 숫자는 불안정한 클러스터를 나타냅니다.
초당 쿼리수 | 초당 실행된 SQL 쿼리 수입니다.
P50 대기 시간 | 서비스 대기 시간의 50번째 백분위 수입니다. 서비스 대기 시간은 클러스터가 쿼리를 받고 쿼리 실행을 마치는 사이의 시간으로 계산됩니다. 이 시간에는 결과를 클라이언트에 반환하는 작업은 포함되지 않습니다.
P99 대기 시간 | 서비스 대기 시간의 99번째 백분위 수입니다.

{{site.data.alerts.callout_info}}
{% include v2.1/misc/available-capacity-metric.md %}
{{site.data.alerts.end}}

### 이벤트 패널

**Cluster Metrics** 대시 보드에는 클러스터의 모든 노드에 대해 로그된 가장 최근의 10개의 이벤트를 나열하는 **이벤트** 패널이 표시됩니다. 모든 이벤트 목록을 보려면, **이벤트** 패널에서 **View all events**를 클릭하십시오.

<img src="{{ 'images/v2.1/admin_ui_events.png' | relative_url }}" alt="CockroachDB Admin UI Events" style="border:1px solid #eee;max-width:100%" />

다음과 같은 유형의 이벤트가 나열됩니다:

- 데이터베이스 생성
- 데이터베이스 삭제
- 테이블 생성
- 테이블 삭제
- 테이블 변경
- 인덱스 생성
- 인덱스 삭제
- 사진 생성
- 사진 삭제
- 스키마 변경 취소
- 스키마 변경 완료
- 노드 조인
- 노드 해체
- 노드 재시작
- 클러스터 세팅 변경

## 더 보기

- [문제 해결 개요](troubleshooting-overview.html)
- [자원 지원](support-resources.html)
- [원시 상태 ](monitoring-and-alerting.html#raw-status-endpoints)
