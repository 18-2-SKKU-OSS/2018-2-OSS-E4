---
title: Orchestrate CockroachDB in a Single Kubernetes Cluster (Insecure)
summary: How to orchestrate the deployment, management, and monitoring of an insecure 3-node CockroachDB cluster with Kubernetes.
toc: true
---

<div class="filters filters-big clearfix">
  <a href="orchestrate-cockroachdb-with-kubernetes.html"><button class="filter-button">Secure</button>
  <button class="filter-button current"><strong>Insecure</strong></button></a>
</div>

이 페이지에서는 [StatefulSet](http://kubernetes.io/) 기능을 사용하여 단일 [Kubernetes](http://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/) 클러스터에서 인시큐어 3 노드 CockroachDB 클러스터의 배포, 관리 및 모니터링을 조율하는 방법을 보여줍니다. 

 
대신 다른 지역의 여러 Kubernetes 클러스터에 배포하려면, [Kubernetes 다중 클러스터 배포](orchestrate-cockroachdb-with-kubernetes-multi-cluster.html)를 참고하십시오. 또한, Kubernetes에서 CockroachDB를 실행할 때 알아야 할 잠재적인 성능 병목 현상에 대한 자세한 내용과 성능 향상을 위해 배포를 최적화하는 방법에 대한 지침은 [Kubernetes에서의 CockroachDB 성능](kubernetes-performance.html)을 참조하십시오.

{{site.data.alerts.callout_danger}}프로덕션에서 CockroachDB를 사용하려는 경우, 보안 클러스터를 대신 사용하는 것이 좋습니다. 지침을 보려면 위의 <strong>Secure</strong>을 선택하십시오.{{site.data.alerts.end}}

## 시작하기 전에

시작하기 전에, Kubernetes 관련 용어 및 현재 제한 사항을 검토하는 것이 좋습니다.

### Kubernetes 용어

특징 | 설명
--------|------------
인스턴스 | 실제 또는 가상 머신. 이 튜토리얼에서는, GCE 또는 AWS 인스턴스를 만들고 그것들을 로컬 워크 스테이션의 단일 Kubernetes 클러스터에 참여시킵니다.
[포드](http://kubernetes.io/docs/user-guide/pods/) | 포드는 더커 (Docker) 컨테이너 중 하나의 그룹입니다. 이 튜토리얼에서는, 각 포드를 별도의 인스턴스에서 실행하고 단일 CockroachDB 노드를 실행하는 도커 컨테이너 하나를 포함합니다. 3 포드로 시작하여 4 포드로 늘어납니다.
[StatefulSet](http://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/) | StatefulSet은 stateful 단위로 취급되는 포드의 그룹입니다. 각 포드는 구별 가능한 네트워크 아이덴티티를 가지며 항상 재시작시 동일한 영구 저장 장치에 바인드됩니다. StatefulSet은 버전 1.5에서 베타 버전에 도달 한 후 Kubernetes 버전 1.9에서 안정적인 것으로 간주됩니다.
[영구 볼륨](http://kubernetes.io/docs/user-guide/persistent-volumes/) | 영구 볼륨은 포드에 마운트된 네트워크 저장장치 (GCE의 영구 디스크, AWS의 탄성 블록 저장소)입니다. 영구 볼륨의 수명은 이를 사용하는 포드의 수명에서 분리되어, 각 CockroachDB 노드가 재시작시 동일한 저장 장치에 다시 바인드되도록 보장합니다.<br><br>이 튜토리얼에서는 동적 볼륨 공급을 사용할 수 있다고 가정합니다. 그런 경우가 아니라면, [영구 볼륨 주장](http://kubernetes.io/docs/user-guide/persistent-volumes/#persistentvolumeclaims)이 수동으로 생성되어야 합니다. 

### 제한 사항

{% include {{ page.version.version }}/orchestration/kubernetes-limitations.md %}

## 1단계. Kubernetes 시작

{% include {{ page.version.version }}/orchestration/start-kubernetes.md %}

## 2단계. CockroachDB 노드 시작

{% include {{ page.version.version }}/orchestration/start-cluster.md %}

## 3단계. 클러스터 초기화

{% include {{ page.version.version }}/orchestration/initialize-cluster-insecure.md %}

## 4단계. 빌트인 SQL 클라이언트 사용

{% include {{ page.version.version }}/orchestration/test-cluster-insecure.md %}

## 5단계. Admin UI 접근

{% include {{ page.version.version }}/orchestration/monitor-cluster.md %}

## 6단계. 노드 오류 시뮬레이션

{% include {{ page.version.version }}/orchestration/kubernetes-simulate-failure.md %}

## 7단계. 모니터링 및 경고 설정

{% include {{ page.version.version }}/orchestration/kubernetes-prometheus-alertmanager.md %}

## 8단계. 클러스터 유지 보수

### 클러스터 크기 조정

{% include {{ page.version.version }}/orchestration/kubernetes-scale-cluster.md %}

3. 네 번째 포드를 성공적으로 추가했는지 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    ~~~
    NAME                                   READY     STATUS    RESTARTS   AGE
    alertmanager-cockroachdb-0             2/2       Running   0          2m
    alertmanager-cockroachdb-1             2/2       Running   0          2m
    alertmanager-cockroachdb-2             2/2       Running   0          2m
    cockroachdb-0                          1/1       Running   0          9m
    cockroachdb-1                          1/1       Running   0          9m
    cockroachdb-2                          1/1       Running   0          7m
    cockroachdb-3                          0/1       Pending   0          5s
    prometheus-cockroachdb-0               3/3       Running   1          5m
    prometheus-operator-85dd478dbb-66lvb   1/1       Running   0          6m
    ~~~

### 클러스터 업그레이드

{% include {{ page.version.version }}/orchestration/kubernetes-upgrade-cluster.md %}

### 클러스터 중지

CockroachDB 클러스터를 종료하려면 다음을 수행하십시오:

1. 로그 및 원격 영구 볼륨을 포함하여 생성한 모든 자원을 삭제하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl delete pods,statefulsets,services,persistentvolumeclaims,persistentvolumes,poddisruptionbudget,jobs,rolebinding,clusterrolebinding,role,clusterrole,serviceaccount,alertmanager,prometheus,prometheusrule,serviceMonitor -l app=cockroachdb
    ~~~

    ~~~
    pod "cockroachdb-0" deleted
    pod "cockroachdb-1" deleted
    pod "cockroachdb-2" deleted
    pod "cockroachdb-3" deleted
    service "alertmanager-cockroachdb" deleted
    service "cockroachdb" deleted
    service "cockroachdb-public" deleted
    persistentvolumeclaim "datadir-cockroachdb-0" deleted
    persistentvolumeclaim "datadir-cockroachdb-1" deleted
    persistentvolumeclaim "datadir-cockroachdb-2" deleted
    persistentvolumeclaim "datadir-cockroachdb-3" deleted
    poddisruptionbudget "cockroachdb-budget" deleted
    job "cluster-init" deleted
    clusterrolebinding "prometheus" deleted
    clusterrole "prometheus" deleted
    serviceaccount "prometheus" deleted
    alertmanager "cockroachdb" deleted
    prometheus "cockroachdb" deleted
    prometheusrule "prometheus-cockroachdb-rules" deleted
    servicemonitor "cockroachdb" deleted
    ~~~

2. Kubernetes 중지:

{% include {{ page.version.version }}/orchestration/stop-kubernetes.md %}

## 더 보기

- [Kubernetes 다중 클러스터 배포](orchestrate-cockroachdb-with-kubernetes-multi-cluster.html)
- [Kubernetes 성능 가이드](kubernetes-performance.html)
{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
