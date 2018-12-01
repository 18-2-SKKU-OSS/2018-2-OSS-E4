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

- **26257** (`tcp:26257`) for inter-node communication (i.e., working as a cluster), for applications to connect to the load balancer, and for routing from the load balancer to nodes
- **8080** (`tcp:8080`) for exposing your Admin UI

To enable this in Azure, you must create a Resource Group, Virtual Network, and Network Security Group.

1. [리소스 그룹 생성](https://azure.microsoft.com/en-us/updates/create-empty-resource-groups/).

2. [가상 네트워크 생성](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-create-vnet-arm-pportal) that uses your **Resource Group**.

3. [네트워크 보안 그룹 생성](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-create-nsg-arm-pportal) that uses your **리소스 그룹**, and then add the following **inbound** rules to it:
    - **관리 UI 지원**:

         분야 | 추천 값 
        -------|-------------------
         이름 | **cockroachadmin** 
         소스 | **IP Addresses** 
         Source IP addresses/CIDR ranges | Your local network’s IP ranges 
         Source port ranges | * 
         Destination | **Any** 
         Destination port range | **8080** 
         프로토콜 | **TCP** 
         Action | **Allow** 
         우선순위 | Any value > 1000 
    - **Application support**:

        {{site.data.alerts.callout_success}}If your application is also hosted on the same Azure     Virtual Network, you will not need to create a firewall rule for your application to communicate     with your load balancer.{{site.data.alerts.end}}

         분야 | 추천 값 
        -------|-------------------
         이름 | **cockroachapp** 
         소스 | **IP Addresses**
         Source IP addresses/CIDR ranges | Your local network’s IP ranges 
         Source port ranges | * 
         Destination | **Any** 
         Destination port range | **26257** 
         Protocol | **TCP** 
         Action | **Allow** 
         Priority | Any value > 1000 


## 2 단계. VMs를 만드십시오.

[리눅스 VMs 생성](https://docs.microsoft.com/en-us/azure/virtual-machines/virtual-machines-linux-quick-create-portal) for each node you plan to have in your cluster. If you plan to run a sample workload against the cluster, create a separate VM for that workload.

- Run at least 3 nodes to [생존 보장](recommended-production-settings.html#cluster-topology).

- Use storage-optimized [Ls-series](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes-storage) VMs with [Premium Storage](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/premium-storage) or local SSD storage with a Linux filesystem such as `ext4` (not the Windows `ntfs` filesystem). For example, Cockroach Labs has used `Standard_L4s` VMs (4 vCPUs and 32 GiB of RAM per VM) for internal testing.

    - If you choose local SSD storage, on reboot, the VM can come back with the `ntfs` filesystem. Be sure your automation monitors for this and reformats the disk to the Linux filesystem you chose initially.

- **Do not** use ["burstable" B-series](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/b-series-burstable) VMs, which limit the load on a single core. Also, Cockroach Labs has experienced data corruption issues on A-series VMs and irregular disk performance on D-series VMs, so we recommend avoiding those as well.

- When creating the VMs, make sure to select the **Resource Group**, **Virtual Network**, and **Network Security Group** you created.

For more details, see [Hardware Recommendations](recommended-production-settings.html#hardware) and [Cluster Topology](recommended-production-settings.html#cluster-topology).

## 3 단계. 클락을 동기화 하십시오.

{% include {{ page.version.version }}/prod-deployment/synchronize-clocks.md %}

## 4 단계. 로드밸런싱을 설정하십시오.

Each CockroachDB node is an equally suitable SQL gateway to your cluster, but to ensure client performance and reliability, it's important to use load balancing:

- **성능:** Load balancers spread client traffic across nodes. This prevents any one node from being overwhelmed by requests and improves overall cluster performance (queries per second).

- **신뢰성:** Load balancers decouple client health from the health of a single CockroachDB node. In cases where a node fails, the load balancer redirects client traffic to available nodes.

Microsoft Azure offers fully-managed load balancing to distribute traffic between instances.

1. [Add Azure load balancing](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview). Be sure to:
	- Set forwarding rules to route TCP traffic from the load balancer's port **26257** to port **26257** on the nodes.
	- Configure health checks to use HTTP port **8080** and path `/health?ready=1`. This [health endpoint](monitoring-and-alerting.html#health-ready-1) ensures that load balancers do not direct traffic to nodes that are live but not ready to receive requests.

2. Note the provisioned **IP Address** for the load balancer. You'll use this later to test load balancing and to connect your application to the cluster.

{{site.data.alerts.callout_info}}If you would prefer to use HAProxy instead of Azure's managed load balancing, see the <a href="deploy-cockroachdb-on-premises-insecure.html">On-Premises</a> tutorial for guidance.{{site.data.alerts.end}}

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
