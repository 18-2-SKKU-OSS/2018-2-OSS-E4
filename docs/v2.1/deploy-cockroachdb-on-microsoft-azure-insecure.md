---
title: Deploy CockroachDB on Microsoft Azure (Insecure)
summary: Learn how to deploy CockroachDB on Microsoft Azure.
toc: true
toc_not_nested: true
ssh-link: https://docs.microsoft.com/en-us/azure/virtual-machines/linux/mac-create-ssh-keys
---

<div class="filters filters-big clea rfix">
  <a href="deploy-cockroachdb-on-microsoft-azure.html"><button class="filter-button">Secure</button>
  <button class="filter-button current"><strong>Insecure</strong></button></a>
</div>

이 페이지에서는 클라이언트 트래픽을 분산하기 위한 Azure의 관리된 로드 밸런싱 서비스를 사용하여, 불안정한 다중 노드 CockroachDB 클러스터를 Microsoft Azure에 배포하는 방법을 보여 줍니다.

{{site.data.alerts.callout_danger}}만약 프로덕션에서 CockroachDB를 사용할 계획을 가지고 있다면, 대신 보안 클러스터를 사용하는 것을 추천합니다. 설명서를 보려면 위의 <strong>Secure</strong>을 선택하십시오.{{site.data.alerts.end}}


## 요구사항들

{% include {{ page.version.version }}/prod-deployment/insecure-requirements.md %}

## 추천사항들

{% include {{ page.version.version }}/prod-deployment/insecure-recommendations.md %}

## 1 단계. 네트워크를 구성하십시오.

CockroachDB는 두 개의 포트에서 TCP 통신을 필요로 합니다:

- **26257** (`tcp:26257`) 노드간 통신 (i.e., working as a cluster), 어플리케이션을 로드밸런서에 연결, 로드 밸런서에서 노드로 노선을 연결하기 위함
- **8080** (`tcp:8080`) 관리 UI 노출을 위함

Azure에서 이것을 가능하게 하기 위해 리소스 그룹, 가상 네트워크, 네트워크 보안 그룹을 생성해야 합니다.

1. [리소스 그룹 생성](https://azure.microsoft.com/en-us/updates/create-empty-resource-groups/).

2. [가상 네트워크 생성](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-create-vnet-arm-pportal) **리소스 그룹**.을 사용 합니다.

3. [네트워크 보안 그룹 생성](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-create-nsg-arm-pportal) **리소스 그룹**을 사용하고, **인바운드** 규칙을 추가합니다:
    - **관리 UI 지원**:

         분야 | 추천 값 
        -------|-------------------
         이름 | **cockroachadmin** 
         소스 | **IP 주소** 
         소스 IP 주소/CIDR 범위 | 지역 네트워크의 IP 범위 
         포트 범위 소스 | * 
         목적지 | **어떤 것이든** 
         목적지 포트 범위 | **8080** 
         프로토콜 | **TCP** 
         행동 | **허락** 
         우선순위 | 어떤 수 > 1000 
    - **어플리케이션 지원**:

        {{site.data.alerts.callout_success}} 어플리케이션이 동일한 Azure 가상 네트워크에서도 호스팅 되는 경우, 어플리케이션이 로드 밸런서와 통신하는데 방화벽 규칙을 만들 필요가 없습니다.{{site.data.alerts.end}}

         분야 | 추천 값 
        -------|-------------------
         이름 | **cockroachapp** 
         소스 | **IP 주소**
         소스 IP 주소/CIDR 범위 | 로컬 네트워크의 IP 범위 
         포트 범위 소스 | * 
         목적지 | **어떤 것이든** 
         목적지 포트 범위 | **26257** 
         프로토콜 | **TCP** 
         행동 | **허락** 
         우선순위 | 어떤 수 > 1000


## 2 단계. VMs를 만드십시오.

[리눅스 VMs 생성](https://docs.microsoft.com/en-us/azure/virtual-machines/virtual-machines-linux-quick-create-portal) 클러스터에 포함될 각 노드에 대한 설명을 위해서 합니다. 클러스터 대신 샘플 워크로드를 실행할 계횓인 경우, 그 워크로드에 대한 분리된 VM을 생성하십시오.

- 최소한 3개의 노드를 실행하여 [생존성을 보장](recommended-production-settings.html#cluster-topology)하십시오.

- [Ls-series](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes-storage) [프리미엄 저장소](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/premium-storage)를 가지는 VMs 또는 `ext4` (윈도우의 `ntfs` 파일시스템이 아님)과 같은 리눅스 파일 시스템을 가지는 지역 SSD 저장소와 같은 최적화된 저장소를 사용하십시오. 예를 들어, Cockroach Labs 내부 테스트 용으로 `Standard_L4s` VMs (4 vCPUs and 32 GiB of RAM per VM)을 사용합니다.

    - 로컬 SSD 저장소를 선택한 경우, 재부팅 시, VM이 `ntfs` 과 함께 돌아올 수 있습니다. 처음에 선택한 리눅스 파일 시스템으로 디스크를 재구성하는지 확인하십시오.

- 단일 코어에 대한 로드를 제한하는 ["burstable" B-series](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/b-series-burstable) VMs을 **사용하지 마십시오**. 또한, Cockroach Labs는 A-series VMs에서 일어나는 데이터 손상 문제와 D-series VMs에서 발생하는 불규칙한 디스크 성능을 경험했으므로, 이러한 문제를 피하는 것을 추천합니다.

- VMs을 생성할 때, 당신이 생성한 **리소스 그룹**, **가상 네트워크**, 그리고 **네트워크 보안 그룹** 을 선택하십시오.

더 자세한 설명을 위해, [하드웨어 추천](recommended-production-settings.html#hardware)와 [클러스터 토폴로지](recommended-production-settings.html#cluster-topology)를 보십시오.

## 3 단계. 클락을 동기화 하십시오.

{% include {{ page.version.version }}/prod-deployment/synchronize-clocks.md %}

## 4 단계. 로드밸런싱을 설정하십시오.

각각의 CockroachDB 노드는 모든 클러스터에 동일하게 적절한 SQL 게이트웨이이지만, 고객에게 성능과 신뢰성을 보장하기 위해, 로드 밸런싱을 사용하는 것은 중요합니다:

- **성능:** 노드 밸런서가 노드 전체에 클라이언트 트래픽을 분산합니다. 따라서 한 개의 노드가 리퀘스트에 의해 압도되지 않고 전체 클러스터 성능이 향상됩니다 (초당 쿼리).

- **신뢰성:** 로드 밸런서는 클라이언트 상태를 단일 CockroachDB 노드의 상태와 분리합니다. 노드가 실패한 케이스에, 로드 밸런서는 클라이언트 트래픽을 사용 가능한 노드로 다시 보냅니다.

Microsoft Azure는 인스턴스 간에 트래픽을 분산하기 위해 완전히 관리되는 로드 밸런싱을 제공합니다.

1. [Azure 로드 밸런싱 추가](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview):
	- 로드 밸런서의 포트 **26257** 에서 노드에 있는 포트 **26257** 로 TCP 트래픽을 라우팅하는 포워딩 규칙을 설정합니다 .
	- HTTP 포트 **8080** 과 경로 `/health?ready=1`을 사용하는 상태 체크을 설정합니다. [상태 엔드포인트](monitoring-and-alerting.html#health-ready-1) 로드 밸런서가 활성 상태이지만 요청을 수신할 준비가 되지 않은 노드로 트래픽을 전송하지 않도록 합니다.

2. 로드 밸런서에 의해 공급된 **IP 주소** 를 기록합니다. 나중에 이 기능을 사용하여 로드 밸런싱을 테스트하고 응용 프로그램을 클러스터에 연결할 수 있습니다.

{{site.data.alerts.callout_info}}Azure의 관리된 로드 밸런싱 대신에 HAProxy을 사용하는 것을 선호한다면, 가이드 라인을 위해 <a href="deploy-cockroachdb-on-premises-insecure.html">On-Premises</a> 을 보십시오.{{site.data.alerts.end}}

## 5 단계. 노드들을 시작하십시오.

{% include {{ page.version.version }}/prod-deployment/insecure-start-nodes.md %}

## 6 단계. 클러스터를 초기화 하십시오.

{% include {{ page.version.version }}/prod-deployment/insecure-initialize-cluster.md %}

## 7 단계. 클러스터를 테스트 하십시오.

{% include {{ page.version.version }}/prod-deployment/insecure-test-cluster.md %}

## 8 단계. 샘플 작업을 실행하십시오.

{% include {{ page.version.version }}/prod-deployment/insecure-test-load-balancing.md %}

## 9 단계. 모니터링과 경고를 설정하십시오.

{% include {{ page.version.version }}/prod-deployment/monitor-cluster.md %}

## 10 단계. 클러스터를 확장하십시오.

{% include {{ page.version.version }}/prod-deployment/insecure-scale-cluster.md %}

## 11 단계. 클러스터를 사용하십시오.

이제 배포가 되는 중이므로, 다음을 수행할 수 있습니다:

1. [데이터 모델 실행](sql-statements.html).
2. [사용자 만들기](create-and-manage-users.html)과 [grant them privileges](grant.html).
3. [응용 프로그램 연결](install-client-drivers.html). 응용프로그램을 CockroachDB 노드가 아니라 Azure 로드 밸런서에 연결해야 합니다.

## 또 다른 참고문헌들

{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
