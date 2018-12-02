---
title: Deploy CockroachDB on Microsoft Azure
summary: Learn how to deploy CockroachDB on Microsoft Azure.
toc: true
toc_not_nested: true
ssh-link: https://docs.microsoft.com/en-us/azure/virtual-machines/linux/mac-create-ssh-keys
---

<div class="filters filters-big clearfix">
  <button class="filter-button current"><strong>Secure</strong></button>
  <a href="deploy-cockroachdb-on-microsoft-azure-insecure.html"><button class="filter-button">Insecure</button></a>
</div>

이 페이지는 클라이언트 트래픽을 배포하는 Azure의 관리된 로드 밸런싱 서비스를 사용하여 CockroachDB 클러스터의 안전한 다중 노드를 Microsoft Azure에 배포하는 방법을 보여줍니다. 

만약 TLS 암호화를 사용한 네트워크 커뮤니케이션을 신경쓰지 않고, 오직 CockroachDB를 테스트해 보고 싶기만 하다면, 보안되지 않는 클러스를 대신 사용할 수 있습니다. 설명서를 보려면 위의 **Insecure** 을 선택하십시오.


## 요구사항

{% include {{ page.version.version }}/prod-deployment/secure-requirements.md %}

## 추천사항

{% include {{ page.version.version }}/prod-deployment/secure-recommendations.md %}

## 단계 1. 네트워크 구성

CockroachDB는 두 개의 포트에서 TCP 통신을 필요로 합니다:

- **26257** (`tcp:26257`) 노드간 통신 (i.e., working as a cluster), 어플리케이션을 로드밸런서에 연결, 로드 밸런서에서 노드로 노선을 연결하기 위함
- **8080** (`tcp:8080`) 관리 UI 노출을 위함

Azure에서 이것을 가능하게 하기 위해 리소스 그룹, 가상 네트워크, 네트워크 보안 그룹을 생성해야 합니다.

1. [리소스 그룹 생성](https://azure.microsoft.com/en-us/updates/create-empty-resource-groups/).
2. [가상 네트워크 생성](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-create-vnet-arm-pportal) **리소스 그룹**을 이용해서 하십시오.
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

        {{site.data.alerts.callout_success}}어플리케이션이 동일한 Azure 가상 네트워크에서도 호스팅 되는 경우, 어플리케이션이 로드 밸런서와 통신하는데 방화벽 규칙을 만들 필요가 없습니다.{{site.data.alerts.end}}

         분야 | 추천 값 
        -------|-------------------
         이름 | **cockroachapp** 
         소스 | **IP 주소** 
         소스 IP 주소/CIDR 범위 | 지역 네트워크의 IP 범위 
         포트 범위 소스 | * 
         목적지 | **어떤 것이든** 
         목적지 포트 범위 | **26257** 
         프로토콜 | **TCP** 
         행동 | **허락** 
         우선순위 | 어떤 수 > 1000 

## 단계 2. VMs 이 인증서 생성

[Linux VMs 생성](https://docs.microsoft.com/en-us/azure/virtual-machines/virtual-machines-linux-quick-create-portal) for each node you plan to have in your cluster. If you plan to run a sample workload against the cluster, create a separate VM for that workload.

- 적어도 3개의 노드를 실행하여 [생존성을 보장](recommended-production-settings.html#cluster-topology)하십시오.

- [Ls-series](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes-storage) [프리미엄 저장소](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/premium-storage)를 가지는 VMs 또는 `ext4` (윈도우의 `ntfs` 파일시스템이 아님)과 같은 리눅스 파일 시스템을 가지는 지역 SSD 저장소와 같은 최적화된 저장소를 사용하십시오. 예를 들어, Cockroach Labs 내부 테스트 용으로 `Standard_L4s` VMs (4 vCPUs and 32 GiB of RAM per VM)을 사용합니다.

    - 로컬 SSD 저장소를 선택한 경우, 재부팅 시, VM이 `ntfs` 과 함께 돌아올 수 있습니다. 처음에 선택한 리눅스 파일 시스템으로 디스크를 재구성하는지 확인하십시오.

- 단일 코어에 대한 로드를 제한하는 ["burstable" B-series](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/b-series-burstable) VMs을 **사용하지 마십시오**. 또한, Cockroach Labs는 A-series VMs에서 일어나는 데이터 손상 문제와 D-series VMs에서 발생하는 불규칙한 디스크 성능을 경험했으므로, 이러한 문제를 피하는 것을 추천합니다.

- VMs을 생성할 때, 당신이 생성한 **리소스 그룹**, **가상 네트워크**, 그리고 **네트워크 보안 그룹** 을 선택하십시오.

더 자세한 설명을 위해, [하드웨어 추천](recommended-production-settings.html#hardware)와 [클러스터 토폴로지](recommended-production-settings.html#cluster-topology)를 보십시오.

## 단계 3. 클락 동기화

{% include {{ page.version.version }}/prod-deployment/synchronize-clocks.md %}

## 단계 4. 로드 밸런싱 설정

각각의 CockroachDB 노드는 모든 클러스터에 동일하게 적절한 SQL 게이트웨이이지만, 고객에게 성능과 신뢰성을 보장하기 위해, 로드 밸런싱을 사용하는 것은 중요합니다:

- **성능:** 노드 밸런서가 노드 전체에 클라이언트 트래픽을 분산합니다. 따라서 한 개의 노드가 리퀘스트에 의해 압도되지 않고 전체 클러스터 성능이 향상됩니다 (초당 쿼리).

- **신뢰성:** 로드 밸런서는 클라이언트 상태를 단일 CockroachDB 노드의 상태와 분리합니다. 노드가 실패한 케이스에, 로드 밸런서는 클라이언트 트래픽을 사용 가능한 노드로 다시 보냅니다.

Microsoft Azure는 인스턴스 간에 트래픽을 분산하기 위해 완전히 관리되는 로드 밸런싱을 제공합니다.

1.  [Azure 로드 밸런싱 추가](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview):
  - 로드 밸런서의 포트 **26257** 에서 노드에 있는 포트 **26257** 로 TCP 트래픽을 라우팅하는 포워딩 규칙을 설정합니다 .
	- HTTP 포트 **8080** 과 경로 `/health?ready=1`을 사용하는 상태 체크을 설정합니다. [상태 엔드포인트](monitoring-and-alerting.html#health-ready-1) 로드 밸런서가 활성 상태이지만 요청을 수신할 준비가 되지 않은 노드로 트래픽을 전송하지 않도록 합니다.

2. 로드 밸런서에 의해 공급된 **IP 주소** 를 기록합니다. 나중에 이 기능을 사용하여 로드 밸런싱을 테스트하고 응용 프로그램을 클러스터에 연결할 수 있습니다.

{{site.data.alerts.callout_info}}Azure의 관리된 로드 밸런싱 대신에 HAProxy을 사용하는 것을 선호한다면, 가이드 라인을 위해 <a href="deploy-cockroachdb-on-premises-insecure.html">On-Premises</a> 을 보십시오.{{site.data.alerts.end}}

## 단계 5. 인증서 생성

{% include {{ page.version.version }}/prod-deployment/secure-generate-certificates.md %}

## 단계 6. 노드 시작

{% include {{ page.version.version }}/prod-deployment/secure-start-nodes.md %}

## 단계 7. 클러스터 초기화

{% include {{ page.version.version }}/prod-deployment/secure-initialize-cluster.md %}

## 단계 8. 클러스터 테스트

{% include {{ page.version.version }}/prod-deployment/secure-test-cluster.md %}

## 단계 9. 샘플 워크로드 실행

{% include {{ page.version.version }}/prod-deployment/secure-test-load-balancing.md %}

## 단계 10. 모니터링 및 알림 설정

{% include {{ page.version.version }}/prod-deployment/monitor-cluster.md %}

## 단계 11. 클러스터 확장

{% include {{ page.version.version }}/prod-deployment/secure-scale-cluster.md %}

## 단계 12. 데이터베이스 사용

{% include {{ page.version.version }}/prod-deployment/use-cluster.md %}

## 또 다른 참고 사항

{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
