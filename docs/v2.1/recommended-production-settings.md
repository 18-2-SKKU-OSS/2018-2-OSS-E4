---
title: Production Checklist
summary: Recommended settings for production deployments.
toc: true
---

이 페이지는 CockroachDB 프로덕션 배포에 대한 중요한 권장 사항을 제공합니다.

## 클러스터 위상 배치

### 용어

클러스터의 위상 배치를 올바르게 계획하려면, CockroachDB 관련 기본 용어 몇 가지를 검토하는 것이 중요합니다:

용어    | 정의
-------|------------
**클러 스터**  | 하나 이상의 데이터베이스가 포함된 단일 논리 어플리케이션으로 작동하는 CockroachDB 배포
**노드** | CockroachDB를 실행하는 개별 머신. 많은 노드가 클러스터를 작성하기 위해 결합합니다.
**범위** | CockroachDB는 모든 사용자 데이터와 거의 모든 시스템 데이터를 키-값 쌍으로 구성된 거대한 정렬 맵에 저장합니다. 이 키스페이스는 키스페이스의 연속적인 덩어리인 "범위"로 구분되어, 모든 키를 항상 단일 범위에서 찾을 수 있습니다.
**복제본** | CockroachDB는 각 범위를 복제하고(기본적으로 3회) 각 복제본을 다른 노드에 저장합니다.
**레인지 리스** | 각 범위에 대해, 복제본 중 하나가 "레인지 리스"를 보유합니다. "리스 홀더"라고 하는 이 복제본은 범위에 대한 모든 읽기 및 쓰기 요청을 수신하고 조정합니다.

### 기본 위상 배치 권장 사항

- 별도의 머신에서 각 노드를 실행하십시오. CockroachDB는 노드를 통해 복제하므로, 머신이 실패할 경우 머신당 둘 이상의 노드를 실행하면 데이터 손실의 위험이 증가합니다. 마찬가지로, 머신이 여러 개의 디스크나 SSD를 가지고 있다면, 디스크당 하나의 노드가 아닌 `--store` 플래그가 여러개인 노드 하나를 실행하십시오. 스토어에 대한 자세한 내용은, [노드 시작](start-a-node.html)을 참조하십시오.

- 단일 데이터 센터에 배포할 때:
    - 임의의 1 노드의 오류를 허용하려면, [기본 3 방향 복제 요소](configure-replication-zones.html#view-the-default-replication-zone)가 있는 최소 3개 노드를 사용하십시오. 이 경우, 1 노드가 실패하면, 각 범위는 3개의 복제본 중 2개, 즉 다수를 보유합니다.
    
    - 2개의 동시 노드 장애를 허용하려면, 최소한 5개의 노드를 사용하고, [기본 복제 요소를 증가](configure-replication-zones.html#edit-the-default-replication-zone)시켜 5로 설정하고, [중요한 내부 데이터에 대한 복제 요소를 증가](configure-replication-zones.html#create-a-a-a-system-range)시켜 5로 설정하십시오. 이 경우 2개의 노드가 동시에 실패하는 경우, 각 범위는 5개의 복제본 중 3개, 즉 대다수를 보유합니다.

- 하나 이상의 지역에 있는 여러 데이터 센터에 배포할 때:
    - 1개의 전체 데이터 센터의 오류를 허용하려면, 최소한 3개의 데이터 센터를 사용하고 각 노드에 `--locality`를 설정하여 데이터 센터 전체에 데이터를 고르게 분산시킵니다. (자세한 내용은 다음 문단을 참조하십시오). 이 경우, 1개의 데이터 센터가 오프라인 상태가 되면, 나머지 2개의 데이터 센터는 대다수의 복제본을 유지합니다.
    - 각 노드를 시작할 때는, [`--locality`](start-a-node.html#locality) 플래그를 사용하여 노드의 위치를 설명합니다 (예를 들어, `--locality=region=west,datacenter=us-west-1`). 키-값 쌍은 가장 적게 포함하여 주문해야 하며, 키-값 쌍의 키와 순서는 모든 노드에서 동일해야 합니다.
        - CockroachDB는 각 데이터 조각의 복제본을 우선 순위를 결정하는 순서와 함께 가능한 한 다양한 지역 집합으로 분산시킵니다. 그러나 지역성은 다양한 방식으로 [복제 영역](configure-replication-zones.html#replication-constraints)을 사용하여 데이터 복제본의 위치에 영향을 주는 데 사용될 수 있습니다. 
        - 노드간에 대기 시간이 긴 경우, CockroachDB는 지역성을 사용하여 현재 워크로드에보다 가까운 레인지 리스를 이동시키고, 네트워크 왕복을 줄이며 읽기 성능을 향상시킵니다 ([팔로우 워크로드](demo-follow-the-workload.html)라고도 함). 
        그러나 3개 이상의 데이터 센터에 배포할 경우, 모든 데이터가 "팔로우 워크로드"의 이점을 얻도록 하려면, 데이터 센터의 총 수와 일치하도록 [복제 요소를 증가](configure-replication-zones.html#edit-the-default-replication-zone)시켜야 합니다.
        - 지역성은 [테이블 파티셔닝](partitioning.html) 및 [**노드 맵**](enable-node-map.html) 엔터프라이즈 기능을 사용하기 위한 전제 조건입니다.      

- 5개 이상의 노드로 이루어진 클러스터를 실행하는 경우, 사용자 데이터에 대해 그렇게 하지 않더라도, [중요한 내부 데이터의 복제 요소를 증가](configure-replication-zones.html#create-a-a-system-rangeforasystem-range)시켜 5로 설정하는 것이 가장 안전합니다. 전체 클러스터를 계속 사용하려면, 이 내부 데이터의 범위에 항상 복제본의 대부분을 보유해야 합니다.

{{site.data.alerts.callout_success}}
CockroachDB의 결함 허용 및 자동 복구 기능에 대한 추가 컨텍스트를 보려면, [이 트레이닝](training/fault-tolerance-and-automated-repair.html)을 참조하십시오.
{{site.data.alerts.end}}

## 하드웨어

### 기본 하드웨어 권장사항

- 노드에는 워크로드를 처리할 수있는 충분한 CPU, RAM, 네트워크 및 저장 장치 용량이 있어야 합니다. 프로덕션 환경에 배포하기 전에 하드웨어 설정을 테스트하고 조정하는 것이 중요합니다.

- 최소한 각 노드에는 **2GB RAM과 전체 코어**가 있어야 합니다. 더 많은 데이터, 복잡한 워크로드, 높은 동시성 및 빠른 성능을 위해서는 추가 자원이 필요합니다.
    {{site.data.alerts.callout_danger}}
    단일 코어의 로드를 제한하는 "버스트 가능" 또는 "공유 코어" 가상 머신을 피하십시오.
    {{site.data.alerts.end}}

- 최고의 성능을 위해:
    - HDD보다 SSD를 사용하십시오.
    - 보다 크고 강력한 노드를 사용하십시오. RAM을 추가하는 것보다 더 많은 CPU를 추가하는 것이 일반적으로 더 유용합니다.

- 최고의 탄력성을 위해:
    - 적은 수의 대형 노드 대신 많은 수의 작은 노드를 사용하십시오. 데이터가 더 많은 노드로 확산될 때 실패한 노드에서 복구하는 것이 더 빠릅니다.
    - [영역 구성](configure-replication-zones.html)을 사용하여 복제 요소를 3 (기본값)에서 5로 늘립니다. 이는 로컬 디스크가 실패의 위험이 더 높기 때문에, 종종 커버 아래에 복제되는 클라우드 공급자의 네트워크 연결 디스크가 아닌 로컬 디스크를 사용하는 경우에 특히 좋습니다. [전체 클러스터](configure-replication-zones.html#edit-the-default-replication-zone) 또는 특정 [데이터베이스](configure-replication-zones.html#create-a-replication-zone-for-a-database), [테이블](configure-replication-zones.html#create-a-replication-zone-for-a-table) 또는 [행](configure-replication-zones.html#create-a-replication-zone-for-a-table-or-secondary-index-partition)에 대해 이 작업을 수행할 수 있습니다 (enterprise-only).
        {{site.data.alerts.callout_danger}}
        {% include {{page.version.version}}/known-limitations/system-range-replication.md %}
        {{site.data.alerts.end}}

### 클라우드 관련 권장사항

Cockroach Labs는 자체 내부 테스트를 기반으로 다음과 같은 클라우드 특정 구성을 권장합니다. 여기서 권장하지 않는 구성을 사용하기 전에 철저히 테스트해야 합니다.

#### AWS

- `m`(범용), `c`(연산 최적화) 또는 `i`(스토리지 최적화)[인스턴스](https://aws.amazon.com/ec2/instance-types/)를 사용하십시오. 예를 들어, Cockroach Labs는 내부 테스트를 위해 `m3.large` 인스턴스 (인스턴스당 2개의 vCPU 및 7.5 GiB)를 사용했습니다.
- 단일 코어의 로드를 제한하는 ["버스터가능한" `t2` 인스턴스](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/t2-instances.html)를 사용하지 **마십시오**.
- [공급된 IOPS SSD-backed (io1) EBS volumes](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html#EBSVolumeTypes_piops)이나 [SSD 인스턴스 저장소 볼륨](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ssd-instance-store.html)을 사용하십시오.

#### Azure

- 스토리지에 최적화된 [Ls-시리즈](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes-storage) VMs을 사용하십시오. 예를 들어, Cockroach Labs는 내부 테스트를 위해 `Standard_L4s` VMs (VM당 4개의 vCPU 및 32 GiB의 RAM)을 사용했습니다.
- [프리미엄 스토리지](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/premium-storage) 또는 로컬 SSD 스토리지를 `ext4` (Windows ' NTFS 파일 시스템이 아님)와 같은 Linux 파일 시스템과 함께 사용하십시오. [프리미엄 스토리지 디스크의 크기는 IOPS에 영향을 미친다](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/premium-storage#premium-storage-disk-limits)는 것에 주의하십시오.
- 재부팅할 때 로컬 SSD 스토리지를 선택하면, VM이 `ntfs` 파일 시스템으로 돌아올 수 있습니다. 자동화가 이를 모니터하고 처음에 선택한 Linux 파일 시스템으로 디스크를 다시 포맷하십시오.
- 단일 코어에 대한 로드를 제한하는 ["버스트 가능한" B 시리즈](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/b-series-burstable) VMs를 사용하지 **마십시오**. 또한, Cockroach Labs는 A 시리즈 VM에서의 데이터 손상 문제와 D 시리즈 VM에서의 불규칙한 디스크 성능을 경험했으므로 피해야 합니다.

#### 디지털 오션

- 최소 요구 사항보다 낮은 1GB RAM만 사용하는 표준 Droplets을 제외한 모든 [droplets](https://www.digitalocean.com/pricing/)을 사용하십시오. 모든 Digital Ocean Droplets은 SSD 저장소를 사용합니다.

#### GCE

- `n1-standard` 또는 `n1-highcpu` [미리 정의된 VMs](https://cloud.google.com/compute/pricing#predefined_machine_types), 또는 [커스텀 VMs](https://cloud.google.com/compute/pricing#custommachinetypepricing)을 사용합니다. 예를 들어, Cockroach Labs는 내부 테스트에 사용자 지정 VMs(8 vCPU 및 VM당 RAM 16GiB)을 사용했습니다.
- 단일 코어의 로드를 제한하는 `f1` 또는`g1` [공유 코어 머신](https://cloud.google.com/compute/docs/machine-types#sharedcore)을 사용하지 **마십시오**.
- [로컬 SSD](https://cloud.google.com/compute/docs/disks/#localssds) 또는 [SSD 영구 디스크](https://cloud.google.com/compute/docs/disks/#pdspecs)를 사용하십시오. [SSD 영구 디스크의 IOPS는 디스크 크기와 컴퓨터의 CPU 수에 의존](https://cloud.google.com/compute/docs/disks/performance#optimizessdperformance)한다는 것에 주의하십시오.

## 보안

인시큐어 클러스터에는 심각한 위험이 있습니다:

- 클러스터는 노드의 IP 주소에 액세스할 수있는 모든 클라이언트에 열려 있습니다.
- 모든 사용자, `root`조차도 패스워드를 제공하지 않고 로그인할 수 있습니다.
- `root`로 연결하는 모든 사용자는 클러스터의 모든 데이터를 읽거나 쓸 수 있습니다.
- 네트워크 암호화 또는 인증이 없으므로, 기밀성이 없습니다.

따라서 프로덕션 환경에서 CockroachDB를 배포하려면, TLS 인증서를 사용하여 노드 및 클라이언트의 ID를 인증하고 노드와 클라이언트 간의 기내 데이터를 암호화하는 것이 좋습니다. 빌트인 [`cockroach cert` 명령](create-security-certificates.html) 또는 [`openssl` 명령](create-security-certificates-openssl.html)을 사용하여 배포용 보안 인증서를 생성할 수 있습니다. 어떤 옵션을 선택하든, 다음 파일이 필요합니다:

- 다른 모든 인증서에 서명하는 데 사용되는 인증 기관(CA) 인증서 및 키.
- 배포에 포함된 각 노드에 대해 공통 이름 `node`를 가진 별도의 인증서와 키.
- 공통 이름은 사용자 이름으로 설정되는 노드에 연결할 각 클라이언트 및 사용자에 대한 별도의 인증서 및 키. 기본 사용자는 `root`입니다.

    또는 CockroachDB는 [비밀번호 인증](create-and-manage-users.html#사용자인증)을 지원합니다. 일반적으로 대신 클라이언트 인증서를 사용하는 것이 좋습니다.

## 네트워킹

### 네트워킹 플래그

[노드를 시작](start-a-node.html)할 때에는, 두 개의 기본 플래그가 사용되어 네트워크 연결을 제어합니다.

- `--listen-addr`는 다른 노드와 클라이언트로부터의 연결을 받아들일 주소를 결정합니다.
- `--advertise-addr`는 다른 노드에게 사용할 주소를 결정합니다.

효과는 이 두 플래그를 조합하여 사용하는 방법에 따라 다릅니다:

| | 명시되지 않은 `--listen-addr` | 명시된 `--listen-addr` |
|-|----------------------------|------------------------|
| **명시되지 않은`--advertise-addr`** | 노드는 포트 `26257`의 모든 IP 주소를 청취하고 정식 호스트 이름을 다른 노드에 알려줍니다. | 노드는 `--listen-addr`에 지정된 IP 주소/호스트 이름과 `--advertise-addr`에 명시된 포트를 듣고 `--advertise-addr`에 지정된 값을 다른 노드에 알려줍니다.
| **명시된`--advertise-addr`** | 노드는 포트 `27257`의 모든 IP 주소를 청취하고 정식 호스트 이름을 다른 노드에 알려줍니다. **대부분의 경우 권장됨** | 노드는 `--listen-addr`에 지정된 IP 주소/호스트 이름과 `--advertise-addr`에 명시된 포트를 듣고 `--advertise-addr`에 지정된 값을 다른 노드에 알려줍니다. `--advertise-addr` 포트 번호가 `--listen-addr`에서 사용된 것과 다른 경우, 포트 포워딩이 필요합니다.

{{site.data.alerts.callout_success}}
호스트 이름을 사용할 때, 그들이 올바르게 해석되는지 확인하십시오(예를 들어, DNS나 `etc/hosts`). 특히 `--advertise-addr`가 지정되지 않은 경우, `--advertise-addr` 또는 `--listen-addr`를 통해 다른 노드에 알려지는 값에 주의하십시오.
{{site.data.alerts.end}}

### 단일 네트워크상의 클러스터

단일 네트워크에서 클러스터를 실행하는 경우, 네트워크가 개인용인지 여부에 따라 설정이 달라집니다. 개인 네트워크에서 머신은 주소가 네트워크에만 국한되어, 공용 인터넷에 액세스 할 수 없습니다. 이 주소를 사용하는 것이 더 안전하며 일반적으로 공용 주소보다 대기 시간이 짧습니다.

프라이벳? | 권장 설정
---------|------------------
O | `--listen-addr`가 사설 IP 주소로 설정된 각 노드를 시작하고 `--advertise-addr`을 지정하지 마십시오. 
X | 각 노드를`--advertise-addr`가 노드에 전송하는 안정된 공용 IP 주소로 설정된 각 노드를 시작하고 `--listen-addr`을 지정하지 마십시오. 이렇게 하면 다른 노드가 보급된 특정 IP 주소를 사용하게 되지만, 로드 밸런서/클라이언트는 노드로 전송하는 모든 주소를 사용할 수 있습니다.<br><br>로드 밸런서/클라이언트가 네트워크 외부에 있는 경우, 외부 트래픽이 클러스터에 도달할 수 있도록 방화벽을 구성하십시오.

### 여러 네트워크에 걸친 클러스터

여러 네트워크에서 클러스터를 실행하는 경우, 노드가 네트워크에서 서로 연결할 수 있는지 여부에 따라 설정이 달라집니다.

네트워크를 통해 노드에 도달할 수 있는가?| 권장 설정
--------------------------------------|------------------
O | 이는 모든 네트워크가 동일한 클라우드에 있는 경우 일반적입니다. 이 경우, 위의 관련 [단일 네트워크 설정](#cluster-on-a-single-network)을 사용하십시오.
X | 이것은 네트워크가 다른 클라우드에 있을 경우 일반적입니다. 이 경우, [VPN](https://en.wikipedia.org/wiki/Virtual_private_network), [VPC](https://en.wikipedia.org/wiki/Virtual_private_cloud), [NAT](https://en.wikipedia.org/wiki/Network_address_translation) 또는 네트워크에서 통합 전송을 제공하는 또 다른 솔루션을 설정하십시오. 그런 다음, `--advertise-addr`을 다른 네트워크에서 접근할 수 있는 주소로 설정된 각 노드를 시작하고, `--listen-addr`을 지정하지 마십시오. 이렇게 하면, 다른 노드가 광고된 특정 IP 주소를 사용하게 됩니다. 그러나 로드 밸런서/클라이언트는 노드로 전송되는 모든 주소를 사용할 수 있습니다. <br><br><span class="version-tag">New in v2.1:</span>또한, 개인 또는 로컬 주소에서 네트워크의 다른 노드로부터 노드에 도달 할 수 있으면, 해당 주소로 [`--locality-advertise-addr`](start-a-node.html#networking)을 설정하십시오. 이렇게 하면 성능을 향상시키기 위해 같은 네트워크 내의 노드에 개인 또는 로컬 주소를 선호하게 됩니다. 이 기능을 사용하려면, 각 노드가 [`--locality`](start-a-node.html#locality) 플래그로 시작되어야 합니다 자세한 내용은, 이 [예시](start-a-node.html#start-a-multi-node-cluster-across-private-networks)를 참조하십시오.

## 로드 밸런싱

각 CockroachDB 노드는 클러스터에 똑같이 적합한 SQL 게이트웨이이지만, 클라이언트 성능과 안정성을 보장하려면 로드 밸런싱을 사용하는 것이 중요합니다:

- **성능:** 로드 밸런서는 클라이언트 트래픽을 노드에 분산시킵니다. 이렇게 하면 모든 노드가 요청에 압도당하는 것을 방지하고 전체 클러스터 성능(초당 쿼리 수)을 향상시킬 수 있습니다.

- **신뢰성:** 로드 밸런서는 클라이언트 상태를 단일 CockroachDB 노드의 상태와 분리합니다. 트래픽이 실패한 노드 또는 요청을 수신할 준비가 되지 않은 노드로 전달되지 않도록 하려면, 로드 밸런서는 [CockroachDB 준비 상태 검사](monitoring-and-alerting.html#health-ready-1)를 사용해야 합니다.

    {{site.data.alerts.callout_success}}
     단일 로드 밸런서의 경우, 클라이언트 연결은 노드 장애에 대해 복원력이 있지만, 로드 밸런서 자체는 실패 지점입니다. 따라서, 클라이언트에 대한 로드 밸런서를 선택하는 유동 IP 또는 DNS와 같은 메커니즘과 함께 여러 로드 밸런싱 인스턴스를 사용하여, 로드 밸런싱을 복원하는 것이 가장 좋습니다.
    {{site.data.alerts.end}}

로드 밸런싱에 대한 지침은 배포 환경에 대한 튜토리얼을 참조하십시오:

환경 | 주요 접근 방식
------------|---------------------
[On-Premises](deploy-cockroachdb-on-premises.html#step-6-set-up-haproxy-load-balancers) | HAProxy 사용.
[AWS](deploy-cockroachdb-on-aws.html#step-4-set-up-load-balancing) | Amazon의 관리 로드 밸런싱 서비스를 사용.
[Azure](deploy-cockroachdb-on-microsoft-azure.html#step-4-set-up-load-balancing) | Azure의 관리 로드 밸런싱 서비스를 사용.
[Digital Ocean](deploy-cockroachdb-on-digital-ocean.html#step-3-set-up-load-balancing) | 디지털 오션의 관리 로드 밸런싱 서비스를 사용.
[GCE](deploy-cockroachdb-on-google-cloud-platform.html#step-4-set-up-tcp-proxy-load-balancing) | GCE의 관리 TCP 프록시 로드 밸런싱 서비스를 사용.

## 모니터링 및 경고

{% include {{ page.version.version }}/prod-deployment/monitor-cluster.md %}

## 클럭 동기화

{% include {{ page.version.version }}/faq/clock-synchronization-effects.html %}

## 캐시 및 SQL 메모리 크기

기본적으로 각 노드의 캐시 크기와 임시 SQL 메모리 크기는 각각 `128MiB`입니다. 이러한 기본값은 사용자가 단일 컴퓨터에서 여러 CockroachDB 노드를 실행할 가능성이 있는 개발 및 테스트를 용이하게 하기 위해 선택되었습니다. 그러나 호스트당 하나의 노드가 있는 프로덕션 클러스터를 실행할 때는, 이 크기들을 늘리는 것이 좋습니다.

- 노드의 **캐시 크기**를 늘리면 노드의 읽기 성능이 향상됩니다.
- 노드의 **SQL 메모리 크기**를 늘리면, `ORDER BY`,`GROUP BY`,`DISTINCT`, 조인 및 창 함수를 사용할 때 행의 메모리 내 처리를 위한 노드의 용량뿐만 아니라 허용되는 동시 클라이언트 연결 수를 늘릴 수 있습니다 (`128MiB` 기본값은 최대 6200개의 동시 연결을 허용합니다).


노드의 캐시 크기와 SQL 메모리 크기를 수동으로 늘리려면, [`--cache`](start-a-node.html#플래그)와 [`--max-sql-memory`](start-a-node.html#플래그)플래그를 사용하여 노드를 시작하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start --cache=.25 --max-sql-memory=.25 <other start flags>
~~~

{{site.data.alerts.callout_danger}}
`--cache`와`--max-sql-memory`를 기계의 전체 RAM의 75% 이상을 합친 값으로 설정하지 마십시오. 이렇게 하면 메모리 관련 오류의 위험이 증가합니다.
{{site.data.alerts.end}}

## 파일 설명자 제한

CockroachDB는 종종 기본적으로 사용할 수 있는 것보다 많은 수의 열린 파일을 사용할 수 있습니다. 따라서 다음 권장 사항에 유의하십시오.

각 CockroachDB 노드에 대하여:

- **최소**에서 파일 설명자 제한은 1956 (스토어당 1700 네트워킹의 경우 +256)이어야 합니다. 한계가 이 임계 값보다 낮으면 노드가 시작되지 않습니다. 
- 파일 설명자 제한을 무제한으로 설정하는 것이 **좋습니다**. 그렇지 않은 경우, 권장되는 제한은 최소 15000입니다 (상점당 10000 + 네트워킹용 5000). 이 상한선은 성능을 보장하고 클러스터 성장을 수용합니다.
- 파일 설명자 제한이 권장 금액을 할당 할만큼 충분히 높지 않으면, CockroachDB는 저장소당 10000개를 할당하고 나머지는 네트워킹용으로 할당합니다. 이로 인해 네트워킹이 256보다 작아지는 경우, CockroachDB는 네트워킹을 위해 256을 할당하고 나머지는 상점간에 균등하게 나눕니다.

### 파일 설명자 제한 증가

<script>
$(document).ready(function(){

    //detect os and display corresponding tab by default
    if (navigator.appVersion.indexOf("Mac")!=-1) {
        $('#os-tabs').find('button').removeClass('current');
        $('#mac').addClass('current');
        toggleMac();
    }
    if (navigator.appVersion.indexOf("Linux")!=-1) {
        $('#os-tabs').find('button').removeClass('current');
        $('#linux').addClass('current');
        toggleLinux();
    }
    if (navigator.appVersion.indexOf("Win")!=-1) {
        $('#os-tabs').find('button').removeClass('current');
        $('#windows').addClass('current');
        toggleWindows();
    }

    var install_option = $('.install-option'),
        install_button = $('.install-button');

    install_button.on('click', function(e){
      e.preventDefault();
      var hash = $(this).prop("hash");

      install_button.removeClass('current');
      $(this).addClass('current');
      install_option.hide();
      $(hash).show();

    });

    //handle click event for os-tab buttons
    $('#os-tabs').on('click', 'button', function(){
        $('#os-tabs').find('button').removeClass('current');
        $(this).addClass('current');

        if($(this).is('#mac')){ toggleMac(); }
        if($(this).is('#linux')){ toggleLinux(); }
        if($(this).is('#windows')){ toggleWindows(); }
    });

    function toggleMac(){
        $(".mac-button:first").trigger('click');
        $("#macinstall").show();
        $("#linuxinstall").hide();
        $("#windowsinstall").hide();
    }

    function toggleLinux(){
        $(".linux-button:first").trigger('click');
        $("#linuxinstall").show();
        $("#macinstall").hide();
        $("#windowsinstall").hide();
    }

    function toggleWindows(){
        $("#windowsinstall").show();
        $("#macinstall").hide();
        $("#linuxinstall").hide();
    }
});
</script>

<div id="os-tabs" class="clearfix">
    <button id="mac" class="current" data-eventcategory="buttonClick-doc-os" data-eventaction="mac">Mac</button>
    <button id="linux" data-eventcategory="buttonClick-doc-os" data-eventaction="linux">Linux</button>
    <button id="windows" data-eventcategory="buttonClick-doc-os" data-eventaction="windows">Windows</button>
</div>

<section id="macinstall" markdown="1">

- [요세미티와 그 이후](#yosemite-and-later)
- [오래된 버전](#older-versions)

#### 요세미티와 그 이후

Linux에서 단일 프로세스에 대한 파일 설명자 제한을 조정하려면, PAM 사용자 제한을 활성화하고 [위](#file-descriptors-limit)에서 설명한 권장 사항으로 엄격한 한계를 설정하십시오. CockroachDB는 항상 엄격한 한계를 사용하므로, 부드러운 한계를 조정하는 것은 기술적으로 필수적이지는 않지만, 아래 단계에서 그렇게 합니다.

예를 들어, 3개의 스토어가 있는 노드의 경우, 엄격한 한계를 다음과 같이 최소 35000(상점당 10000개 및 네트워킹용 5000 개)으로 설정합니다:

1.  현재 한계를 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ launchctl limit maxfiles
    ~~~
    ~~~
    maxfiles    10240          10240
    ~~~

    마지막 두 열은 각각 부드러운, 엄격한 한계입니다. `unlimited`이 엄격한 한계로 나열되면, 단일 프로세스의 숨겨진 기본 제한은 실제로 10240입니다.

2.  `/Library/LaunchDaemons/limit.maxfiles.plist`를 생성하고 `ProgramArguments` 배열의 마지막 문자열을 35000으로 설정하여 다음 내용을 추가하십시오:

    ~~~ xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
      <plist version="1.0">
        <dict>
          <key>Label</key>
            <string>limit.maxfiles</string>
          <key>ProgramArguments</key>
            <array>
              <string>launchctl</string>
              <string>limit</string>
              <string>maxfiles</string>
              <string>35000</string>
              <string>35000</string>
            </array>
          <key>RunAtLoad</key>
            <true/>
          <key>ServiceIPC</key>
            <false/>
        </dict>
      </plist>
    ~~~

    plist 파일이 `root:wheel`에 의해 소유되고 권한 `-rw-r--r--`을 가지고 있는지 확인하십시오. 이러한 사용 권한은 기본적으로 적용됩니다.
    

3. 새 제한사항이 적용되도록 시스템을 다시 시작하십시오.

4. 현재 한계를 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ launchctl limit maxfiles
    ~~~
    ~~~
    maxfiles    35000          35000
    ~~~

#### 이전 버전

Linux에서 단일 프로세스에 대한 파일 설명자 제한을 조정하려면, PAM 사용자 제한을 활성화하고 [위](#file-descriptors-limit)에서 설명한 권장 사항으로 엄격한 한계를 설정하십시오. CockroachDB는 항상 엄격한 한계를 사용하므로, 부드러운 한계를 조정하는 것은 기술적으로 필수적이지는 않지만, 아래 단계에서 그렇게 합니다.

예를 들어, 3개의 스토어가 있는 노드의 경우, 엄격한 한계를 다음과 같이 최소 35000(상점당 10000개 및 네트워킹용 5000 개)으로 설정합니다.

1. 현재 한계를 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ launchctl limit maxfiles
    ~~~
    ~~~
    maxfiles    10240          10240
    ~~~

    마지막 두 열은 각각 부드러운, 엄격한 한계입니다. `unlimited`이 엄격한 한계로 나열되면, 단일 프로세스의 숨겨진 기본 제한은 실제로 10240입니다.


2.  `/etc/launchd.conf`를 편집(또는 생성)하고 마지막 값을 새로운 엄격한 한계로 설정한 다음과 같은 줄을 추가하십시오:

    ~~~
    limit maxfiles 35000 35000
    ~~~

3.  파일을 저장하고 시스템을 다시 시작하여 새 제한 사항을 적용하십시오.

4.  새 한계를 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ launchctl limit maxfiles
    ~~~
    ~~~
    maxfiles    35000          35000
    ~~~

</section>

<section id="linuxinstall" markdown="1">

- [프로세스 당 한도](#per-process-limit)
- [S시스템 전체 제한](#system-wide-limit)

#### 프로세스당 한도

Linux에서 단일 프로세스에 대한 파일 설명자 제한을 조정하려면, PAM 사용자 제한을 활성화하고 [위](#file-descriptors-limit)에서 설명한 권장 사항으로 엄격한 한계를 설정하십시오. CockroachDB는 항상 엄격한 한계를 사용하므로, 부드러운 한계를 조정하는 것은 기술적으로 필수적이지는 않지만, 아래 단계에서 그렇게 합니다.

예를 들어, 3개의 스토어가 있는 노드의 경우, 엄격한 한계를 다음과 같이 최소 35000(상점당 10000 개 및 네트워킹용 5000 개)으로 설정합니다:

1.  `/etc/pam.d/common-session`과`/etc/pam.d/common-session-noninteractive`에 다음 줄이 있는지 확인하십시오:

    ~~~ shell
    session    required   pam_limits.so
    ~~~

2.  `/etc/security/limits.conf`를 편집하고 다음 줄을 파일에 추가하십시오:

    ~~~ shell
    *              soft     nofile          35000
    *              hard     nofile          35000
    ~~~

    `*`는 CockroachDB 서버를 실행할 사용자 이름으로 바꿀 수 있습니다.

4.  파일을 저장하고 닫습니다.

5.  새 제한 사항이 적용되도록 시스템을 다시 시작하십시오.

6.  새 한계를 확인하십시오:

    ~~~ shell
    $ ulimit -a
    ~~~

또는, [Systemd](https://en.wikipedia.org/wiki/Systemd)를 사용하는 경우:

1.  열려있는 최대 파일 수를 구성하려면 서비스 정의를 편집하십시오:

    ~~~ ini
    [Service]
    ...
    LimitNOFILE=35000
    ~~~

2.  새로운 한도가 적용되도록 시스템을 다시 로드하십시오:

    ~~~ shell
    $ systemctl daemon-reload
    ~~~

#### 시스템 전체 제한

또한 Linux 시스템 전체에 대한 파일 설명자 제한이 위에 설명된 프로세스당 한도(예 : 최소 150000)보다 최소 10배 이상 높다는 것도 확인해야 합니다.

1. 시스템 전체 한계를 확인하십시오:

    ~~~ shell
    $ cat /proc/sys/fs/file-max
    ~~~

2. 필요하다면, `proc` 파일 시스템에서 시스템 전체 한계를 늘리십시오:

    ~~~ shell
    $ echo 150000 > /proc/sys/fs/file-max
    ~~~

</section>
<section id="windowsinstall" markdown="1">

CockroachDB는 아직 기본 Windows 바이너리를 제공하지 않습니다. 사용할 수 있게 되면, Windows에서 파일 설명자 제한을 조정하는 방법에 대한 설명서도 제공합니다.

</section>

#### 귀인

이 섹션의 "파일 설명자 제한"은 부분적으로 *오픈 파일 한도*에서 파생되었습니다. Riak LV 2.1.4 설명서에서, Creative Commons Attribution 3.0 Unported License에 사용됩니다.

## 오케스트레이션 / 쿠베르네스

Kubernetes에서 CockroachDB를 실행할 때, 다음과 같은 최소한의 사용자 지정을 수행하면, 더 나은 성능을 얻을 수 있습니다:

* [기존 HDD 대신 SSD](kubernetes-performance.html)를 사용.
* CPU 및 메모리 [자원 요청 및 제한](kubernetes-performance.html#resource-requests-and-limits)을 구성.

자세한 정보 및 추가 사용자 정의 제안 사항은 [Kubernetes에서 CockroachDB 성능](kubernetes-performance.html)의 전체 세부 안내서를 참조하십시오.
