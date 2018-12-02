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
2. [가상 네트워크 생성](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-create-vnet-arm-pportal) that uses your **Resource Group**.
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
    - **Application support**:

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

- Run at least 3 nodes to [ensure survivability](recommended-production-settings.html#cluster-topology).

- Use storage-optimized [Ls-series](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes-storage) VMs with [Premium Storage](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/premium-storage) or local SSD storage with a Linux filesystem such as `ext4` (not the Windows `ntfs` filesystem). For example, Cockroach Labs has used `Standard_L4s` VMs (4 vCPUs and 32 GiB of RAM per VM) for internal testing.

    - If you choose local SSD storage, on reboot, the VM can come back with the `ntfs` filesystem. Be sure your automation monitors for this and reformats the disk to the Linux filesystem you chose initially.

- **Do not** use ["burstable" B-series](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/b-series-burstable) VMs, which limit the load on a single core. Also, Cockroach Labs has experienced data corruption issues on A-series VMs and irregular disk performance on D-series VMs, so we recommend avoiding those as well.

- When creating the VMs, make sure to select the **Resource Group**, **Virtual Network**, and **Network Security Group** you created.

For more details, see [Hardware Recommendations](recommended-production-settings.html#hardware) and [Cluster Topology](recommended-production-settings.html#cluster-topology).

## 단계 3. 클락 동기화

{% include {{ page.version.version }}/prod-deployment/synchronize-clocks.md %}

## 단계 4. 로드 밸런싱 설정

Each CockroachDB node is an equally suitable SQL gateway to your cluster, but to ensure client performance and reliability, it's important to use load balancing:

- **Performance:** Load balancers spread client traffic across nodes. This prevents any one node from being overwhelmed by requests and improves overall cluster performance (queries per second).

- **Reliability:** Load balancers decouple client health from the health of a single CockroachDB node. In cases where a node fails, the load balancer redirects client traffic to available nodes.

Microsoft Azure offers fully-managed load balancing to distribute traffic between instances.

1.  [Add Azure load balancing](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview). Be sure to:
  - Set forwarding rules to route TCP traffic from the load balancer's port **26257** to port **26257** on the nodes.
  - Configure health checks to use HTTP port **8080** and path `/health?ready=1`. This [health endpoint](monitoring-and-alerting.html#health-ready-1) ensures that load balancers do not direct traffic to nodes that are live but not ready to receive requests.

2.  Note the provisioned **IP Address** for the load balancer. You'll use this later to test load balancing and to connect your application to the cluster.

{{site.data.alerts.callout_info}}If you would prefer to use HAProxy instead of Azure's managed load balancing, see the <a href="deploy-cockroachdb-on-premises.html">On-Premises</a> tutorial for guidance.{{site.data.alerts.end}}

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
