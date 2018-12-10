---
title: Diagnostics Reporting
summary: Learn about the diagnostic details that get shared with CockroachDB and how to opt out of sharing.
toc: true
---

기본적으로 관리 UI와 CockroachDB 클러스터의 각 노드는 익명 사용 상세 내역을 Cockroach Labs와 공유합니다. 식별 가능한 정보를 완전히 스크러빙하는 이러한 세부 사항들은 우리가 실제 시나리오에서 시스템이 어떻게 동작하는지를 이해하고 개선하는 데 크게 도움이 됩니다.

이 페이지에는 공유되는 세부 사항, 직접 세부 정보를 보는 방법, 공유에서 제외하는 방법을 요약하여 설명합니다.

{{site.data.alerts.callout_success}}
클러스터의 성능과 상태에 대해 이해하려면 기본 제공되는 [관리 UI](admin-ui-overview.html) 또는 [Prometheus](monitor-cockroachdb-with-prometheus.html).와 같은 타사 모니터링 도구를 사용하십시오.
{{site.data.alerts.end}}

## 공유되는 세부 사항

진단 보고가 활성화되어 있을 때 CockroachDB 클러스터의 각 노드는 익명으로 된 세부 정보를 매 시간마다 공유하며, 여기에는 다음 데이터가 포함됩니다:

- 노드에 있는 스토어
- 노드가 실행중인 하드웨어
- 노드에 저장된 테이블 구조
- 노드에서 실행한 SQL 쿼리 유형
- 노드에 적용되는 [복제 영역](configure-replication-zones.html)
- 변경된 [`CLUSTER SETTINGS`](cluster-settings.html)
- 노드에 의해 보고된 캐시
- 관리 UI의 사용자 정보 및 페이지 뷰

{{site.data.alerts.callout_info}}
모든 경우에 이름 및 기타 문자열 값은 스크러빙되고 밑줄로 대체됩니다. 또한 공유되는 세부 사항은 시간이 지남에 따라 변경될 수 있으며, 변경 사항이 발생한 경우 우리는 릴리스 노트에서 변경 사항을 알립니다.
{{site.data.alerts.end}}

## 직접 세부 정보를 보는 방법

노드가 Cockroach Labs에 보고하는 진단 세부 정보를 보려면 `http://<node-address>:<http-port>/_status/diagnostics/local` JSON 엔드포인드를 사용하십시오.

## 공유에서 제외하는 방법

### 클러스터 초기화 시

클러스터의 첫 번째 노드를 시작하기 전에 진단 세부 정보가 공유되지 않도록 환경 변수를 `COCKROACH_SKIP_ENABLING_DIAGNOSTIC_REPORTING=true`로 설정하십시오. 클러스터의 첫 번째 노드를 시작하기 전에 설정한 경우에만 이 기능이 작동한다는 점에 유의하십시오. 이미 클러스터가 실행된 경우에는 아래에 설명된 `SET CLUSTER SETTING`방법을 사용해야 합니다.

### 클러스터 초기화 이후

클러스터가 실행된 이후에 Cockroach Labs로 진단 세부 정보를 전송하는 것을 중지하기 위해서는 [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)를 사용하여 `diagnostics.reporting.enabled` [클러스터 설정](cluster-settings.html) 을 `false` 로 전환하는 [`SET CLUSTER SETTING`](set-cluster-setting.html) 문을 실행하십시오.

{% include copy-clipboard.html %}
~~~ sql
> SET CLUSTER SETTING diagnostics.reporting.enabled = false;
~~~

이 변경은 클러스터의 다른 노드에 전파될 때까지 어느 정도 시간이 걸릴 수 있습니다.

## 진단 보고의 상태 확인

진단 보고의 상태를 확인하려면 [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)를 사용하여 [`SHOW CLUSTER SETTING`](show-cluster-setting.html) 명령문을 실행하십시오.

{% include copy-clipboard.html %}
~~~ sql
> SHOW CLUSTER SETTING diagnostics.reporting.enabled;
~~~

~~~
+-------------------------------+
| diagnostics.reporting.enabled |
+-------------------------------+
| false                         |
+-------------------------------+
(1 row)
~~~

이 설정이 `false` 상태이면 진단 보고가 비활성 상태임을 뜻하고 `true`이면 진단 보고가 활성 상태임을 뜻합니다.

## 더 알아보기

- [클러스터 설정](cluster-settings.html)
- [노드 시작하기](start-a-node.html)
