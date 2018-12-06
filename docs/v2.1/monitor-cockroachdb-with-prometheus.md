---
title: Monitor CockroachDB with Prometheus
summary: Use Prometheus to monitor CockroachDB.
toc: true
---

클러스터의 각 노드에 대한 자세한 시계열 메트릭을 생성합니다. 이 페이지에서는 시계열 데이터를 저장, 집계 및 쿼리하는 오픈 소스 도구인 [프로메테우스](https://prometheus.io/)에 이러한 메트릭을 가져 오는 방법을 보여줍니다. 또한 유연한 데이터 시각화 및 알림을 위해, [그라파나](https://grafana.com/) 및 [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/)를 프로메테우스에 연결하는 방법을 보여줍니다.

{{site.data.alerts.callout_success}}기타 모니터링 옵션에 대한 자세한 내용은, <a href="monitoring-and-alerting.html"> 모니터링 및 경고</a>를 참조하십시오.{{site.data.alerts.end}}


## 시작하기 전에

- [로컬](start-a-local-cluster.html) 또는 [프로덕션 환경](manual-deployment.html)에서 CockroachDB 클러스터를 이미 시작했는지 확인하십시오.

- 이 튜토리얼에서 사용된 모든 파일은 CockroachDB 저장소의 [`monitoring`](https://github.com/cockroachdb/cockroach/tree/master/monitoring) 디렉토리에 있습니다.

## 1단계. 프로메테우스 설치

1. 사용중인 OS용 [2.x 프로메테우스 타르볼](https://prometheus.io/download/)을 다운로드하십시오.

2. 바이너리를 추출하여 `PATH`에 추가하십시오. 따라서 모든 쉘에서 프로메테우스를 쉽게 시작할 수 있습니다.

3. 프로메테우스가 성공적으로 설치되었는지 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ prometheus --version
    ~~~

    ~~~
    prometheus, version 2.2.1 (branch: HEAD, revision: bc6058c81272a8d938c05e75607371284236aadc)
      build user:       root@149e5b3f0829
      build date:       20180314-14:21:40
      go version:       go1.10
    ~~~

## 2단계. 프로메테우스 구성 

1. CockroachDB의 스타터 [프로메테우스 구성 파일](https://github.com/cockroachdb/cockroach/blob/master/monitoring/prometheus.yml) 다운로드하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/prometheus.yml \
    -O prometheus.yml
    ~~~

    구성 파일을 검사하면, 10초마다 인시큐어 단일 로컬 노드의 시계열 메트릭을 스크래핑하도록 설정되었음을 알 수 있습니다:
    
    - `scrape_interval: 10s` 스크래핑 간격을 정의합니다. 
    - `metrics_path: '/_status/vars'` 시계열 메트릭을 스크래핑하는 프로메테우스 관련 CockroachDB 엔드 포인트를 정의합니다.
    - `scheme: 'http'` 스크래핑되는 클러스터가 인시큐어하다고 명시합니다.
    - `targets: ['localhost:8080']` 시계열 메트릭을 수집할 Cockroach 노드의 호스트이름과 `http-port`를 명시합니다.

2. 배포 시나리오에 맞게 구성 파일을 편집합니다:

    시나리오 | 구성 변경
    ---------|--------------
    다중 노드 로컬 클러스터 | 각 추가 노드에 대해, `'localhost:<http-port>'`를 포함하도록 `targets` 필드를 확장하십시오.
    프로덕션 클러스터 | 클러스터의 각 노드에 대해, `'<hostname>:<http-port>'`를 포함하도록 `targets` 필드를 변경하십시오. 또한, 네트워크 구성이 지정된 포트에서 TCP 통신을 허용하는지 확인하십시오.
    시큐어 클러스터 | `scheme: 'https'`를 주석 처리하고 `scheme: 'http'`를 주석 처리하십시오.

4. `rules` 디렉토리를 생성하고 CockroachDB를 위한 [어그레게이션 규칙](https://github.com/cockroachdb/cockroach/blob/master/monitoring/rules/aggregation.rules.yml)과 [경고 규칙](https://github.com/cockroachdb/cockroach/blob/master/monitoring/rules/alerts.rules.yml)을 다운로드하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ mkdir rules
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cd rules
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget -P rules https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/rules/aggregation.rules.yml
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget -P rules https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/rules/alerts.rules.yml
    ~~~

## 3단계. 프로메테우스 시작

1. 구성 파일을 가리키는 `--config.file` 플래그로 프로메테우스 서버를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ prometheus --config.file=prometheus.yml
    ~~~

    ~~~
    INFO[0000] Starting prometheus (version=1.4.1, branch=master, revision=2a89e8733f240d3cd57a6520b52c36ac4744ce12)  source=main.go:77
    INFO[0000] Build context (go=go1.7.3, user=root@e685d23d8809, date=20161128-10:02:41)  source=main.go:78
    INFO[0000] Loading configuration file prometheus.yml     source=main.go:250
    INFO[0000] Loading series map and head chunks...         source=storage.go:354
    INFO[0000] 0 series loaded.                              source=storage.go:359
    INFO[0000] Listening on :9090                            source=web.go:248
    INFO[0000] Starting target manager...                    source=targetmanager.go:63
    ~~~

2. 프로메테우스 UI를 사용하여 CockroachDB 시계열 메트릭을 쿼리, 집계 및 그래프로 표시할 수 있는 `http://<hostname of machine running prometheus>:9090`를 브라우저로 지정하십시오.  
  - Prometheus는 자동으로 CockroachDB 시계열 메트릭을 완성하지만, 설명이 포함된 전체 목록을 보려면 `http://<hostname of a CockroachDB node>:8080/_status/vars`를 브라우저로 지정하십시오.
  - 프로메테우스 UI 사용에 대한 자세한 내용은, [공식 문서](https://prometheus.io/docs/introduction/getting_started/)를 보십시오. 

## 4단계. Alertmanager를 사용하여 알림 보내기

액티브 모니터링은 문제를 일찍 발견하는 데 도움이 되지만, 조사 또는 개입이 필요한 이벤트가 있을 때 알림을 보내는 것이 필수적입니다. 2단계에서, CockroachDB의 스타터 [경고 규칙](https://github.com/cockroachdb/cockroach/blob/master/monitoring/rules/alerts.rules.yml)을 이미 다운로드했습니다. 이제, [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/)를 다운로드, 구성 및 시작하십시오.

1. 사용중인 OS에 맞는 [최신 Alertmanager 타르볼](https://prometheus.io/download/#alertmanager)을 다운로드하십시오.

2. 바이너리를 추출하여 `PATH`에 추가하십시오. 이렇게 하면 모든 쉘에서 Alertmanager를 쉽게 시작할 수 있습니다.

3. Alertmanager가 성공적으로 설치되었는지 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ alertmanager --version
    ~~~

    ~~~
    alertmanager, version 0.15.0-rc.1 (branch: HEAD, revision: acb111e812530bec1ac6d908bc14725793e07cf3)
      build user:       root@f278953f13ef
      build date:       20180323-13:07:06
      go version:       go1.10
    ~~~

4. 바이너리 `simple.yml`과 함께 제공되는 [Alertmanager 구성 파일을 편집](https://prometheus.io/docs/alerting/configuration/)하여 알림에 대하여 원하는 수신자를 지정합니다.

5. 구성 파일을 가리키는 `--config.file` 플래그로 Alertmanager 서버를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ alertmanager --config.file=simple.yml
    ~~~

6. Alertmanager UI를 사용하여 [경보 해지](https://prometheus.io/docs/alerting/alertmanager/#silences)에 대한 규칙을 정의할 수 있는 `http://<hostname of machine running alertmanager>:9093`를 브라우저를 지정하십시오. 


## 5단계. 그라파나의 메트릭 시각화

프로메테우스는 그래프 메트릭을 제공하지만, [그라파나](https://grafana.com/)는 프로메테우스와 쉽게 통합되는 훨씬 강력한 시각화 도구입니다.

1. [OS에 그라파나를 설치 및 시작](https://grafana.com/grafana/download)하십시오.

2. `http://<hostname of machine running grafana>:3000`를 브라우저로 지정하고, 그라파나 UI에 기본 사용자이름/암호인 `admin/admin`으로 로그인하거나 자신의 계정을 생성합니다. 

3. [프로메테우스를 데이터 소스로 추가](http://docs.grafana.org/datasources/prometheus/)하고, 다음과 같이 데이터 소스를 구성하십시오:

    필드 | 정의
    ------|-----------
    이름 | 프로메테우스
    기본 | 트루
    타입 | 프로메테우스
    Url | `http://<hostname of machine running prometheus>:9090`
    접근 | 다이렉트

4. CockroachDB의 스타터 [대시 보드](https://github.com/cockroachdb/cockroach/tree/master/monitoring/grafana-dashboards)를 다운로드하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    # runtime dashboard: node status, including uptime, memory, and cpu.
    $ wget https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/grafana-dashboards/runtime.json
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    # storage dashboard: storage availability.
    $ wget https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/grafana-dashboards/storage.json
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    # sql dashboard: sql queries/transactions.
    $ wget https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/grafana-dashboards/sql.json
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    # replicas dashboard: replica information and operations.
    $ wget https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/grafana-dashboards/replicas.json
    ~~~

5. [그라파나에 대시 보드 추가](http://docs.grafana.org/reference/export_import/#importing-a-dashboard).

## 더 보기

- [모니터링 및 경고](monitoring-and-alerting.html)
