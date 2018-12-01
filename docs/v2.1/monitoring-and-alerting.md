---
title: Monitoring and Alerting
summary: Monitor the health and performance of a cluster and alert on critical events and metrics.
toc: true
---

CockroachDB의 다양한 [장애에 대비한 내장형 보호 장치](high-availability.html)에도 불구하고, 프로덕션에서 실행 중인 클러스터의 전체적인 상태와 성능을 능동적으로 모니터링하는 것과 조사 또는 개입이 필요한 이벤트가 있을 때 알람을 보내는 알림 규칙을 만드는 것은 중요합니다.

이 페이지에서는 사용 가능한 모니터링툴과 경고할 중요 이벤트 및 메트릭을 설명합니다.


## 모니터링 도구

### 관리 UI

[기본 제공 관리 UI](admin-ui-access-and-navigate.html)는 활성화 상태인 노드의 개수, 비활성화 상태인 노드의 개수, 의심스러운 노드의 개수와 초당 쿼리와 클러스터 전체의 서비스 지연 시간 등과 같은 클러스터 상태에 대한 필수 메트릭을 제공합니다. 디폴트에 의해 `http://<host>:<http-port>`, 또는 `http://<host>:8080`의 모든 노드에서 관리 UI에 접근할 수 있습니다.

#### 보안 클러스터의 관리 UI에 접근

보안클러스터의 관리 UI에 접근 할 수 있는 각 사용자들은, [암호로 사용자를 만듭니다](create-user.html#create-a-user-with-a-password) 관리 UI에 접근하면, 사용자들은 그들의 사용이름과 비밀번호를 입력해야 하는 로그인 화면을 보게 됩니다.

{{site.data.alerts.callout_danger}}관리 UI가 CockroachDB에 내장되어 있기 때문에, 만약 클러스터를 사용할 수 없는 경우, 대부분의 관리 UI도 사용할 수 없게 됩니다. 따라서, 아래에 설명된 대로 클러스터 상태를 모니터링하는 추가 방법을 계획하는 것이 중요합니다.{{site.data.alerts.end}}

### 프로메테우스 엔드포인트

CockroachDB 클러스터의 모든 노드는 `http://<host>:<http-port>/_status/vars`로 세분화된 시계열 메트릭스를 내보냅니다. 메트릭스들은 시계열 데이터를 저장, 집계 및 쿼리할 수 있는 좋은 오픈소스 도구인 [Prometheus](https://prometheus.io/)과 쉽게 통합될 수 있도록 포맷되어 있습니다. 그러나, 형식은  **easy-to-parse** 이며 다른 타사 모니터링 시스템과 함께 작동하도록 될 수 있습니다. (e.g., [Sysdig](https://sysdig.atlassian.net/wiki/plugins/servlet/mobile?contentId=64946336#content/view/64946336) and [Stackdriver](https://github.com/GoogleCloudPlatform/k8s-stackdriver/tree/master/prometheus-to-sd)).

Prometheus 사용에 대한 교육서는 [프로메테우스와 CockroachDB ](monitor-cockroachdb-with-prometheus.html)를 참고하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ curl http://localhost:8080/_status/vars
~~~

~~~
# HELP gossip_infos_received Number of received gossip Info objects
# TYPE gossip_infos_received counter
gossip_infos_received 0
# HELP sys_cgocalls Total number of cgo calls
# TYPE sys_cgocalls gauge
sys_cgocalls 3501
# HELP sys_cpu_sys_percent Current system cpu percentage
# TYPE sys_cpu_sys_percent gauge
sys_cpu_sys_percent 1.098855319644276e-10
# HELP replicas_quiescent Number of quiesced replicas
# TYPE replicas_quiescent gauge
replicas_quiescent{store="1"} 20
...
~~~

{{site.data.alerts.callout_info}}외부 시스템을 통해 클러스터를 모니터링하기 위해 내보낸 timeseries 데이터를 사용하는 것 외에도. 더 많은 디테일 참고를 위해 <a href="#events-to-alert-on">Events to Alert On</a>를 보십시오.{{site.data.alerts.end}}

### 건강 엔드포인트

CockroachDB는 개별 노드의 상태를 확인하기 위해 두개의 HTTP 엔드포인트를 제공합니다. 

#### /건강

노드가 중단된 경우, `http://<host>:<http-port>/health` 엔드포인트는 `Connnection refused`에러를 리턴합니다:

{% include copy-clipboard.html %}
~~~ shell
$ curl http://localhost:8080/health
~~~

~~~
curl: (7) Failed to connect to localhost port 8080: Connection refused
~~~

그렇지 않으면, 노드에 대한 세부정보가 포함된 HTTP `200 OK` 상태 응답 코드를 반환합니다:

~~~
{
  "nodeId": 1,
  "address": {
    "networkField": "tcp",
    "addressField": "JESSEs-MBP:26257"
  },
  "buildInfo": {
    "goVersion": "go1.9",
    "tag": "v2.0-alpha.20180212-629-gf1271b232-dirty",
    "time": "2018/02/21 04:09:53",
    "revision": "f1271b2322a4a1060461707bdccd77b6d5a1843e",
    "cgoCompiler": "4.2.1 Compatible Apple LLVM 9.0.0 (clang-900.0.39.2)",
    "platform": "darwin amd64",
    "distribution": "CCL",
    "type": "development",
    "dependencies": null
  }
}
~~~

#### /health?ready=1

`http://<node-host>:<http-port>/health?ready=1` 의 끝점은 HTTP `503 서비스를 사용할 수 없음` 상태 응답 코드를 반환하고 다음과 같은 시나리오 오류를 발생시킵니다:

- 노드가 [해체된 상태](remove-nodes.html) 또는 [중단된 상태](stop-a-node.html)의 과정에 있으므로 SQL 연결과 쿼리를 실행하는 것이 불가능할 수 있습니다. 이 기능은 특히 로드 밸런서가 존재는 하지만 "ready" 상태가 아닌 노드로 직접 전송하지 않는데에 유용할 수 있습니다. 이 기능은 [롤링 업그레이드](upgrade-cockroach-version.html) 동안에 편리한 확인입니다.
    {{site.data.alerts.callout_success}}만약 로드 밸런서의 상태 체크에서 노드가 종료되기 전의 노드 상태를 준비되지 않은 것으로 인식하지 않는 것을 발견했다면, <code>server.shutdown.drain_wait</code> <a href="cluster-settings.html">클러스터 세팅</a> 을 늘릴 수 있습니다. 이것은 심지어 셧다운을 하기 이전에 노드가 <code>503 Service Unavailable</code>를 반환하게 합니다.{{site.data.alerts.end}}
- 노드는 클러스터에 있는 다른 주요한 노드와 의사소통이 불가능할 수 있는데, 그것은 노드가 너무 많아 클러스터를 사용할 수 없기 때문입니다.

{% include copy-clipboard.html %}
~~~ shell
$ curl http://localhost:8080/health?ready=1
~~~

~~~
{
  "error": "node is not ready",
  "code": 14
}
~~~

그렇지 않으면, 비어있는 상태의 HTTP `200 OK` 상태 응답 코드를 반환합니다:

~~~
{

}
~~~

### 원시적인 상태의 엔드포인트

몇몇 엔드포인트는 `http://<host>:<http-port>/#/debug`에 있는 JSON에 원시 상태의 메트릭을 반환합니다. 이러한 엔드포인트를 자유롭게 조사하고 사용할 수 있습니다. 그러나 그것들이 바뀔 수 있다는 것을 인지하십시오.  

<img src="{{ 'images/v2.1/raw-status-endpoints.png' | relative_url }}" alt="Raw Status Endpoints" style="border:1px solid #eee;max-width:100%" />

### 노드 상태 명령

[`cockroach 노드 상태`](view-node-details.html) 명령은 각 노드의 상태에 대해 메트릭을 제공합니다.

- `--ranges` 표시에서, 비가동률과 낮은 복제 기능을 포함하여세부적인 범위와 복제본의 세부 정보를 얻습니다.
- `--stats` 표시에서, 디스크 사용량의 세부 정보를 얻습니다.
- `--decommission` 표시에서, [해체된 노드](remove-nodes.html) 프로세스의 디테일에 대해서 얻습니다.
- `--all` 표시에서, 위의 모든 것을 얻습니다.

## 경고할 이벤트

활성 모니터링은 문제를 조기에 발견하는데 도움이 되지만, 조사 또는 개입이 필요한 이벤트가 있을 때 알림을 즉시 전송하는 알림 규칙을 생성하는 것에도 중요한 역할을 합니다. 이 섹션에서는 이벤트를 탐지하는 데에 [Prometheus Endpoint](#prometheus-endpoint) 메트릭스를 사용하여 알림 규칙을 생성하는 가장 중요한 이벤트를 식별합니다.

{{site.data.alerts.callout_success}}만약 모니러팅을 위해 Prometheus을 사용한다면, <a href="https://github.com/cockroachdb/cockroach/blob/master/monitoring/rules/alerts.rules.yml">alerting rules</a> 또한 사용할 수 있습니다. 가이드로 <a href="monitor-cockroachdb-with-prometheus.html">Monitor CockroachDB with Prometheus</a>를 보십시오.{{site.data.alerts.end}}

### 노드가 중단됨

- **규칙:** 노드가 5분 이상 중단된 경우 경고를 보냅니다.

- **발견하는 방법:** 만약 노드가 중단된다면, 노드의 `_status/vars` 엔드포인트는 `Connection refused` 에러를 반환합니다. 그렇지 않으면, `liveness_livenodes` 메트릭은 클러스터에 있는 전체 활성 노드의 수가 됩니다.

### 노드가 너무 자주 다시 시작함

- **규칙:** 노드가 10분 이내에 5회 이상 재시작된 경우 알림을 보냅니다.

- **발견하는 방법:** 노드의 `_status/vars` 출력값이 0으로 재설정되는 `sys_uptime` 메트릭의 실행 횟수를 이용하여 계산합니다. `sys_uptime` 메트릭은 `cockroach` 프로세스가 실행된 시간(초 단위)를 나타냅니다.

### 노드가 디스크의 적은 공간에서 동작

- **규칙:** 노드에 남은 사용 가능한 공간이 15% 미만인 경우 경고를 보냅니다.

- **발견하는 방법:** `capacity` 메트릭을 노드의 `_status/vars` 출력값에 있는 `capacity_available` 메트릭으로 나눕니다. 

### 노드가 SQL을 실행하지 않음

- **규칙:** 노드가 연결 상태에서도 SQL을 실행하지 않는 경우 경고를 보냅니다.

- **발견하는 방법:** `sql_query_count` 메트릭이 `0`인 동안에 노드의 `_status/vars` 출력값에 있는 `sql_conns` 메트릭은  `0` 보다 커질 것입니다. `sql_select_count`, `sql_insert_count`, `sql_update_count`, 그리고 `sql_delete_count`을 이용해서 statement 유형으로 세분화할 수 있습니다.

### CA 인증서가 곧 끝남

- **규칙:** CA 인증서 만료가 1년 미만으로 남았을 때 경고를 보냅니다.

- **발견하는 방법:** 노드의 `_status/vars` 출력값에 있는 `security_certificate_expiration_ca` 메트릭을 사용하여 이 값을 계산합니다.

### 노드 인증서가 곧 끝남

- **규칙:** 노드의 인증서 만료가 1년 미만으로 남았을 때 경고를 보냅니다.

- **발견하는 방법:** 노드의 `_status/vars` 출력값에 있는 `security_certificate_expiration_node` 메트릭을 사용하여 이 값을 계산합니다.

## 또 다른 참고문헌

- [Production Checklist](recommended-production-settings.html)
- [Manual Deployment](manual-deployment.html)
- [Orchestrated Deployment](orchestration.html)
- [Test Deployment](deploy-a-test-cluster.html)
- [Local Deployment](start-a-local-cluster.html)
