---
title: Monitoring and Alerting
summary: Monitor the health and performance of a cluster and alert on critical events and metrics.
toc: true
---

CockroachDB의 다양한 [built-in safeguards against failure](high-availability.html)에도 불구하고, 프로덕션에서 실행 중인 클러스터의 전반적인 상태 및 성능을 능동적으로 모니터링하고 즉시 알림을 보내는 경고 규칙을 만드는 것이 중요합니다.

이 페이지에서는 사용 가능한 모니터링 툴과 경고할 중요 이벤트 및 메트릭을 설명합니다.


## Monitoring tools

### Admin UI

[built-in Admin UI](admin-ui-access-and-navigate.html) 는 실시간, 비활성 및 의심스러운 노드 수 등 클러스터 상태에 대한 필수 메트릭을 제공합니다. 기본적으로 UI 또는 `http://<host>:<http-port>`, `http://<host>:8080` 의 모든 노드에서 관리 UI에 엑세스할 수 있습니다.

#### Accessing the Admin UI for a secure cluster

보안클러스터의 관리 UI에 액세스 할 수 있는 각 사용자에 대해 [create a user with a password](create-user.html#create-a-user-with-a-password) 를 사용하여 사용자 생성, 관리 UI에 액세스하면 로그인 화면이 나타나고 사용자 이름과 암호를 임력해야 합니다.

{{site.data.alerts.callout_danger}}관리 UI가 CockroachDB에 내장되어 있기 때문에 틀러스터를 사용할 수 없는 경우 대부분의 관리 UI도 사용할 수 없게 됩니다. 따라서 아래에 설명된 대로 클러스터 상태를 모니터링하는 추가 방법을 계획하는 것이 중요합니다.{{site.data.alerts.end}}

### Prometheus endpoint

CockroachDB 클러스터의 모든 노드는 `http://<host>:<http-port>/_status/vars`에서 세분화된 시간 측정 지표를 내보냅니다. 메트릭은 [Prometheus](https://prometheus.io/)과 쉽게 통합될 수 있도록 포맷되어 있습니다. 타임리어 데이터를 저장, 집계 및 쿼리하기 위한 오픈 소스 도구이지만, 형식은 is **easy-to-parse** 이며 다른 타사 모니터링 시스템과 함께 작동하도록 될 수 있습니다. (e.g., [Sysdig](https://sysdig.atlassian.net/wiki/plugins/servlet/mobile?contentId=64946336#content/view/64946336) and [Stackdriver](https://github.com/GoogleCloudPlatform/k8s-stackdriver/tree/master/prometheus-to-sd)).

Prometheus 사용에 대한 교육서는 [Monitor CockroachDB with Prometheus](monitor-cockroachdb-with-prometheus.html)를 참고하십시오.

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

노드가 중단된 경우, `http://<host>:<http-port>/health` 엔드포인트는 `Connnection refused` 에러를 리턴합니다:

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

`http://<node-host>:<http-port>/health?ready=1` 의 끝점은 HTTP `503 서비스를 사용할 수 없음` 상태 응답 코드를 반환하고 다음과 같은 시나리오 오류를 발생시킵니다;

- 노드가 [decommissioned](remove-nodes.html) ㄸ;ㅗ는 [shutting down](stop-a-node.html)의 과정에 있으므로 SQL 연결과 쿼리를 실행하는 것이 불가능할 수 있습니다. 이 기능은 특히 로드 밸런서가 존재는 하지만 "ready" 상태가 아닌 노드로 직접 전송하지 않는데에 유용할 수 있습니다. 이 기능은 [롤링 업그레이드](upgrade-cockroach-version.html) 동안에 편리한 확인입니다.
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

그렇지 않으면, 비어있는 상태의 HTTP `200 OK` 상태 응답 코드를 반환합니다. :

~~~
{

}
~~~

### 원시적인 상태의 엔드포인트

몇몇 엔드포인트는 `http://<host>:<http-port>/#/debug`에 있는 JSON에 원시 상태의 메트릭을 반환합니다. 이러한 엔드포인트를 자유롭게 조사하고 사용할 수 있습니다. 그러나 그것들이 바뀔 수 있다는 것을 인지하십시오.  

<img src="{{ 'images/v2.1/raw-status-endpoints.png' | relative_url }}" alt="Raw Status Endpoints" style="border:1px solid #eee;max-width:100%" />

### 노드 상태 명령

[`cockroach node status`](view-node-details.html) 명령은 각 노드의 상태에 대해 메트릭을 제공합니다.

- `--ranges` 표시에서, 비가동률과 낮은 복제 기능을 포함하여세부적인 범위와 복제본의 세부 정보를 얻습니다.
- `--stats` 표시에서, 디스크 사용량의 세부 정보를 얻습니다.
- `--decommission` 표시에서, [node decommissioning](remove-nodes.html) 프로세스의 디테일에 대해서 얻습니다.
- `--all` 표시에서, 위의 모든 것을 얻습니다.

## 경고할 이벤트

Active monitoring helps you spot problems early, but it is also essential to create alerting rules that promptly send notifications when there are events that require investigation or intervention. This section identifies the most important events to create alerting rules for, with the [Prometheus Endpoint](#prometheus-endpoint) metrics to use for detecting the events.

{{site.data.alerts.callout_success}}If you use Prometheus for monitoring, you can also use our pre-defined <a href="https://github.com/cockroachdb/cockroach/blob/master/monitoring/rules/alerts.rules.yml">alerting rules</a> with Alertmanager. See <a href="monitor-cockroachdb-with-prometheus.html">Monitor CockroachDB with Prometheus</a> for guidance.{{site.data.alerts.end}}

### Node is down

- **Rule:** Send an alert when a node has been down for 5 minutes or more.

- **How to detect:** If a node is down, its `_status/vars` endpoint will return a `Connection refused` error. Otherwise, the `liveness_livenodes` metric will be the total number of live nodes in the cluster.

### Node is restarting too frequently

- **Rule:** Send an alert if a node has restarted more than 5 times in 10 minutes.

- **How to detect:** Calculate this using the number of times the `sys_uptime` metric in the node's `_status/vars` output was reset back to zero. The `sys_uptime` metric gives you the length of time, in seconds, that the `cockroach` process has been running.

### Node is running low on disk space

- **Rule:** Send an alert when a node has less than 15% of free space remaining.

- **How to detect:** Divide the `capacity` metric by the `capacity_available` metric in the node's `_status/vars` output.

### Node is not executing SQL

- **Rule:** Send an alert when a node is not executing SQL despite having connections.

- **How to detect:** The `sql_conns` metric in the node's `_status/vars` output will be greater than `0` while the `sql_query_count` metric will be `0`. You can also break this down by statement type using `sql_select_count`, `sql_insert_count`, `sql_update_count`, and `sql_delete_count`.

### CA certificate expires soon

- **Rule:** Send an alert when the CA certificate on a node will expire in less than a year.

- **How to detect:** Calculate this using the `security_certificate_expiration_ca` metric in the node's `_status/vars` output.

### Node certificate expires soon

- **Rule:** Send an alert when a node's certificate will expire in less than a year.

- **How to detect:** Calculate this using the `security_certificate_expiration_node` metric in the node's `_status/vars` output.

## See also

- [Production Checklist](recommended-production-settings.html)
- [Manual Deployment](manual-deployment.html)
- [Orchestrated Deployment](orchestration.html)
- [Test Deployment](deploy-a-test-cluster.html)
- [Local Deployment](start-a-local-cluster.html)
