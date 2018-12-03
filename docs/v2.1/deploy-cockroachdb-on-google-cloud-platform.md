---
title: Deploy CockroachDB on Google Cloud Platform GCE
summary: Learn how to deploy CockroachDB on Google Cloud Platform's Compute Engine.
toc: true
toc_not_nested: true
ssh-link: https://cloud.google.com/compute/docs/instances/connecting-to-instance
---

<div class="filters filters-big clearfix">
  <button class="filter-button current"><strong>Secure</strong></button>
  <a href="deploy-cockroachdb-on-google-cloud-platform-insecure.html"><button class="filter-button">Insecure</button></a>
</div>

이 페이지에서는 Google의 TCP 프록시 로드 밸런싱 서비스를 사용하여 클라이언트 트래픽을 배포하는 Google Cloud Platform의 Compute Engine (GCE)에 안전하지 않은 다중 노드 CockroachDB 클러스터를 수동으로 배포하는 방법을 보여줍니다.

CockroachDB만 테스트 중이거나, TLS 암호화로 네트워크 통신을 보호하는 것과 관련이 없는 경우, 인시큐어 클러스터를 대신 사용할 수 있습니다. 지침을 보려면 **Insecure**을 선택하십시오.


## 요구 사항

{% include {{ page.version.version }}/prod-deployment/secure-requirements.md %}

## 권장 사항

{% include {{ page.version.version }}/prod-deployment/secure-recommendations.md %}

## 1단계. 네트워크 구성

CockroachDB는 두 포트에서 TCP 통신이 필요합니다

- **26257** (`tcp:26257`) 노드 간 통신을 위해 (즉, 클러스터로서 동작)
- **8080** (`tcp:8080`) Admin UI 노출을 위해

노드 간 통신은 GCE 인스턴스의 내부 IP 주소를 사용하여 기본적으로 작동하며, CockroachDB의 기본 포트인 `26257`에서 다른 인스턴스와 통신할 수 있습니다. 그러나, Admin UI를 노출하고 TCP 프록시로드 밸런서 및 상태 검사기의 트래픽을 인스턴스에 허용하려면, [프로젝트에 대한 방화벽 규칙 생성](https://cloud.google.com/compute/docs/vpc/firewalls)이 필요합니다.

### 방화벽 규칙 생성

방화벽 규칙을 만들 때, Google Cloud Platform의 **tag** 기능을 사용하는 것이 좋습니다. 이 기능을 사용하면 동일한 태그가 포함된 인스턴스에만 규칙을 적용하도록 지정할 수 있습니다.

#### Admin UI

  필드 | 권장 값 
-------|-------------------
 이름 | **cockroachadmin** 
 소스 필터 | IP 범위 
 소스 IP 범위 | 로컬 네트워크의 IP 범위
 허용된 프로토콜... | **tcp:8080** 
 타겟 태그 | **cockroachdb** 

#### 어플리케이션 데이터

어플리케이션은 CockroachDB 노드에 직접 연결하지 않습니다. 대신, GCE의 TCP 프록시 로드 밸런싱 서비스에 연결하여, 트래픽을 사용자에게 가장 가까운 인스턴스로 자동 전송합니다. 이 서비스는 Google Cloud 엣지에서 구현되므로, 로드 밸런서와 상태 검사기에서 사용자의 인스턴스로의 트래픽을 허용하는 방화벽 규칙을 생성해야 합니다. 이 내용은 [4단계](#step-4-set-up-tcp-proxy-load-balancing)에서 다룹니다.

{{site.data.alerts.callout_danger}}TCP 프록시 로드 밸런싱을 사용할 때는, 방화벽 규칙을 사용하여 로드 밸런서에 대한 접근을 제어 할 수 없습니다. 이러한 제어가 필요한 경우, <a href="https://cloud.google.com/compute/docs/load-balancing/network/"> 네트워크 TCP 로드 밸런싱</a>을 대신 사용해보십시오. 단, 지역 간에는 사용할 수 없습니다. 또한 HAProxy 로드 밸런서를 사용할 수도 있습니다 (지침은 <a href="deploy-cockroachdb-on-premises.html">On-Premises</a> 튜토리얼 참조).{{site.data.alerts.end}}


## 2단계. 인스턴스 생성

클러스터에 포함하려는 각 노드에 대한 [인스턴스를 생성](https://cloud.google.com/compute/docs/instances/create-start-instance)하십시오. 클러스터에 대해 샘플 워크로드를 실행하려면, 해당 워크로드에 대해 별도의 인스턴스를 생성하십시오.

- 적어도 3개의 노드를 실행하여 [survivability를 보장](recommended-production-settings.html#cluster-topology)하십시오.

- [로컬 SSDs](https://cloud.google.com/compute/docs/disks/#localssds) 및 [SSD 영구 디스크](https://cloud.google.com/compute/docs/disks/#pdspecs)와 함께, `n1-standard` 또는 `n1-highcpu` [미리 정의된 VMs](https://cloud.google.com/compute/pricing#predefined_machine_types) 및 [커스텀 VMs](https://cloud.google.com/compute/pricing#custommachinetypepricing)을 사용하십시오. 예를 들어, Cockroach Labs는 내부 테스트를 위해 커스텀 VMs (VM 당 8 개의 vCPU 및 16 GiB의 RAM)을 사용했습니다.

- 단일 코어의 로드를 제한하는 `f1` 또는 `g1` [공유된 코어 머신](https://cloud.google.com/compute/docs/machine-types#sharedcore)을 사용하지 **마십시오**.

- 방화벽 규칙에 태그를 사용한 경우, 인스턴스를 생성할 때, **Management, disk, networking, SSH keys**를 선택하십시오. 그런 다음 **Networking** 탭의 **Network tags** 필드에 **cockroachdb**를 입력하십시오.

자세한 내용은 [하드웨어 권장 사항](recommended-production-settings.html#hardware) 및 [클러스터 토폴로지](recommended-production-settings.html#cluster-topology)을 참조하십시오.

## 3단계. 클럭 동기화

{% include {{ page.version.version }}/prod-deployment/synchronize-clocks.md %}

## 4단계. TCP 프록시 로드 밸런싱 설정

각 CockroachDB 노드는 클러스터에 똑같이 적합한 SQL 게이트웨이지만, 클라이언트 성능과 안정성을 보장하려면, 로드 밸런싱을 사용하는 것이 중요합니다:

- **성능:** 로드 밸런서는 클라이언트 트래픽을 노드에 분산시킵니다. 이렇게 하면 모든 노드가 요청에 압도당하는 것을 방지하고, 전체 클러스터 성능 (초당 쿼리 수)을 향상시킬 수 있습니다.

- **신뢰성:** 로드 밸런서는 클라이언트 상태를 단일 CockroachDB 노드의 상태와 분리합니다. 노드가 실패 할 경우, 로드 밸런서는 클라이언트 트래픽을 사용 가능한 노드로 재전송합니다.

GCE는 완벽하게 관리되는 [TCP 프록시 로드 밸런싱](https://cloud.google.com/load-balancing/docs/tcp/)을 제공합니다. 이 서비스를 사용하면 전 세계 모든 사용자에게 단일 IP 주소를 사용하여 사용자에게 가장 가까운 인스턴스로 트래픽을 자동 전송할 수 있습니다.

{{site.data.alerts.callout_danger}}TCP 프록시 로드 밸런싱을 사용할 때는, 방화벽 규칙을 사용하여 로드 밸런서에 대한 접근을 제어 할 수 없습니다. 이러한 제어가 필요한 경우, <a href="https://cloud.google.com/compute/docs/load-balancing/network/"> 네트워크 TCP 로드 밸런싱</a>을 대신 사용해보십시오. 단, 지역 간에는 사용할 수 없습니다. 또한 HAProxy 로드 밸런서를 사용할 수도 있습니다 (<a href="deploy-cockroachdb-on-premises-insecure.html"> On-Premises</a> 튜토리얼을 참조).{{site.data.alerts.end}}

GCE의 TCP 프록시 로드 밸런싱 서비스를 사용하려면 다음과 같이 하십시오:

   1. 인스턴스를 실행 중인 각 영역에 대해, [고유한 인스턴스 그룹을 생성](https://cloud.google.com/compute/docs/instance-groups/creating-groups-of-unmanaged-instances)하십시오.
    - 로드 밸런서가 트래픽을 지시할 곳을 확실히 알려면, **Port name**으로 `tcp26257`을, **Port number**로`26257`을 사용하여 포트 이름 맵핑을 지정하십시오.
2. [각 인스턴스 그룹에 관련 인스턴스를 추가](https://cloud.google.com/compute/docs/instance-groups/creating-groups-of-unmanaged-instances#addinstances)하십시오.
3. [프록시 로드 밸런싱 구성](https://cloud.google.com/load-balancing/docs/tcp/setting-up-tcp#configure_load_balancer).
    - 백엔드 구성 중에, 상태 확인을 생성하고, **Protocol**을 `HTTP`로, **Port**를 `8080`으로, **Request path**를 `/health?ready=1` 경로로 설정하십시오. 이 [상태 엔드 포인트](monitoring-and-alerting.html#health-ready-1)는 로드 밸런서가 실시간 상태이지만 요청을 수신 할 준비가 되지 않은 노드로 트래픽을 보내지 않도록 합니다.
        - 수십 초 이상 유휴상태일 수 있는 수명이 긴 SQL 연결을 유지하려면, 이에 따라 백엔드 시간 초과 설정을 늘리십시오.
    - 프론트엔드 구성 중, 고정 IP 주소를 예약하고 포트를 선택하십시오. 이 주소/포트 조합은 모든 클라이언트 연결에 사용하므로, 이 주소/포트 조합을 기록하십시오.
4. 로드 밸런서 및 상태 검사기에서 사용자 인스턴스로의 트래픽을 허용하기 위해 [방화벽 규칙을 생성](https://cloud.google.com/load-balancing/docs/tcp/setting-up-tcp#config-hc-firewall)하십시오. 이는 TCP 프록시 로드 밸런싱이 Google Cloud의 엣지에 구현되어 있기 때문에 필요합니다.
    - **Source IP ranges**를 `130.211.0.0/22` 및 `35.191.0.0/16`으로 설정하고 **Target tags**를 `cockroachdb`로 설정하십시오. (링크된 지침에 명시된 값이 아님).

## 5단계. 인증서 생성

{% include {{ page.version.version }}/prod-deployment/secure-generate-certificates.md %}

## 6단계. 노드 시작

{% include {{ page.version.version }}/prod-deployment/secure-start-nodes.md %}

## 7단계. 클러스터 초기화

{% include {{ page.version.version }}/prod-deployment/secure-initialize-cluster.md %}

## 8단계. 클러스터 테스트

{% include {{ page.version.version }}/prod-deployment/secure-test-cluster.md %}

## 9단계. 샘플 워크로드 실행

{% include {{ page.version.version }}/prod-deployment/secure-test-load-balancing.md %}

## 10단계. 모니터링 및 경고 설정

{% include {{ page.version.version }}/prod-deployment/monitor-cluster.md %}

## 11단계. 클러스터 크기 조정

{% include {{ page.version.version }}/prod-deployment/secure-scale-cluster.md %}

## 12단계. 데이터베이스 사용

{% include {{ page.version.version }}/prod-deployment/use-cluster.md %}

## 더 보기

{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
