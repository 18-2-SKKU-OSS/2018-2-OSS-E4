---
title: Deploy CockroachDB on Digital Ocean (Insecure)
summary: Learn how to deploy CockroachDB on Digital Ocean.
toc: true
toc_not_nested: true
ssh-link: https://www.digitalocean.com/community/tutorials/how-to-connect-to-your-droplet-with-ssh
---

<div class="filters filters-big clearfix">
  <a href="deploy-cockroachdb-on-digital-ocean.html"><button class="filter-button">Secure</button>
  <button class="filter-button current"><strong>Insecure</strong></button></a>
</div>

이 페이지에서는 클라이언트 트래픽을 분산시키는 Digital Ocean의 관리 로드 밸런싱 서비스를 사용하여 Digital Ocean에 안전하지 않은 다중 노드 CockroachDB 클러스터를 배포하는 방법을 보여줍니다.

{{site.data.alerts.callout_danger}}프로덕션에서 CockroachDB를 사용하려는 경우, 보안 클러스터를 대신 사용하는 것이 좋습니다. 지침을 보려면 위의 <strong>Secure</strong>을 선택하십시오.{{site.data.alerts.end}}


## 요구 사항

{% include {{ page.version.version }}/prod-deployment/insecure-requirements.md %}

## 권장 사항

{% include {{ page.version.version }}/prod-deployment/insecure-recommendations.md %}

- 모든 CockroachDB 노드와 클라이언트가 단일 영역에서 Droplet으로 실행될 경우, [개인 네트워킹](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-use-digitalocean-private-networking) 사용을 고려해보십시오.

## 1단계. Droplet 생성

클러스터에 넣으려는 각 노드에 대해 [Droplets을 생성](https://www.digitalocean.com/community/tutorials/how-to-create-your-first-digitalocean-droplet)하십시오. 클러스터에 대해 샘플 워크로드를 실행하려면, 해당 워크로드에 별도의 Droplet을 생성하십시오.

- 최소한 3개의 노드를 실행하여 [존속성 보장](recommended-production-settings.html#cluster-topology).

- 최소한의 요구 사항을 밑도는 RAM 1GB만있는 표준 Droplets을 제외한 모든 [Droplets](https://www.digitalocean.com/pricing/)을 사용하십시오. 모든 Digital Ocean Droplet은 SSD 저장소를 사용합니다.

자세한 내용은 [하드웨어 권장 사항](recommended-production-settings.html#hardware) 및 [클러스터 토폴로지](recommended-production-settings.html#cluster-topology)를 참조하십시오.

## 2단계. 클럭 동기화

{% include {{ page.version.version }}/prod-deployment/synchronize-clocks.md %}

## 3단계. 로드 밸런싱 설정

각 CockroachDB 노드는 클러스터에 똑같이 적합한 SQL 게이트웨이이지만, 클라이언트 성능과 안정성을 보장하려면, 로드 밸런싱을 사용하는 것이 중요합니다:

- **성능:** 로드 밸런서는 클라이언트 트래픽을 노드에 분산시킵니다. 이렇게 하면 모든 노드가 요청에 압도당하는 것을 방지하고 전체 클러스터 성능 (초당 쿼리 수)을 향상시킬 수 있습니다.

- **신뢰성:** 로드 밸런서는 클라이언트 상태를 단일 CockroachDB 노드의 상태와 분리합니다. 노드가 실패할 경우, 로드 밸런서는 클라이언트 트래픽을 사용 가능한 노드로 재전송합니다.

Digital Ocean은 Droplet 사이에서 트래픽을 분산시키기 위해 완벽하게 관리되는 로드 밸런서를 제공합니다.


1. [Digital Ocean 로드 밸런서 생성](https://www.digitalocean.com/community/tutorials/an-introduction-to-digitalocean-load-balancers)하십시오. 다음 사항에 유의하십시오:
	- 로드 밸런서의 포트 **26257**에서 포트 **26257**까지의 TCP 트래픽을 노드 Droplets에 전송하도록 전달 규칙을 설정합니다.
	- HTTP 포트 **8080** 및 `/health?ready=1` 경로를 사용하도록 상태 검사를 구성하십시오. 이 [상태 엔드포인트](monitoring-and-alerting.html#health-ready-1)는 로드 밸런서가 실제 상태이지만 요청을 수신할 준비가 되지 않은 노드로 트래픽을 보내지 않도록 합니다.	
2. 로드 밸런서에 대해 제공된 **IP Address**에 유의하십시오. 나중에 로드 밸런싱을 테스트하고 애플리케이션을 클러스터에 연결하는 데 사용합니다.

{{site.data.alerts.callout_info}}Digital Ocean의 관리되는 로드 밸런싱 대신 HAProxy를 사용하려면, <a href="deploy-cockroachdb-on-premises-insecure.html">On-Premises</a> 튜토리얼을 참조하십시오.{{site.data.alerts.end}}

## 4단계. 네트워크 구성

각 Droplet마다 방화벽을 설정하여 다음 두 포트에서 TCP 통신을 허용하십시오:

- **26257** (`tcp:26257`) 노드 간 통신 (즉, 클러스터로 작업), 애플리케이션이 로드 밸런서에 연결하고, 로드 밸런서에서 노드로 전송하기 위해
- **8080** (`tcp:8080`) Admin UI 공개를 위한

자세한 내용은 Digital Ocean's Guide를 사용하여 Droplet OS 기반의 방화벽 구성을 참조하십시오:


- 우분투와 데비안은 [`ufw`](https://www.digitalocean.com/community/tutorials/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server)를 사용할 수 있습니다.
- FreeBSD는 [`ipfw`](https://www.digitalocean.com/community/tutorials/recommended-steps-for-new-freebsd-10-1-servers)를 사용할 수 있습니다.
- Fedora는 [`iptables`](https://www.digitalocean.com/community/tutorials/initial-setup-of-a-fedora-22-server)을 사용할 수 있습니다.
- CoreOS는 [`iptables`](https://www.digitalocean.com/community/tutorials/how-to-secure-your-coreos-cluster-with-tls-ssl-and-firewall-rules)를 사용할 수 있습니다.
- CentOS는 [`firewalld`](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-using-firewalld-on-centos-7)를 사용할 수 있습니다.

## 5단계. 노드 시작

{% include {{ page.version.version }}/prod-deployment/insecure-start-nodes.md %}

## 6단계. 클러스터 초기화

{% include {{ page.version.version }}/prod-deployment/insecure-initialize-cluster.md %}

## 7단계. 클러스터 테스트

{% include {{ page.version.version }}/prod-deployment/insecure-test-cluster.md %}

## 8단계. 샘플 워크로드 실행

{% include {{ page.version.version }}/prod-deployment/insecure-test-load-balancing.md %}

## 9단계. 모니터링 및 경고 설정

{% include {{ page.version.version }}/prod-deployment/monitor-cluster.md %}

## 10단계. 클러스터 크기 조정

{% include {{ page.version.version }}/prod-deployment/insecure-scale-cluster.md %}

## 11단계. 클러스터 사용

배포가 완료되었으므로, 다음 작업을 수행할 수 있습니다:

1. [데이터 모델 구현](sql-statements.html).
2. [사용자 생성](create-and-manage-users.html) 및 [권한 부여](grant.html).
3. [애플리케이션 연결](install-client-drivers.html). 애플리케이션을 CockroachDB 노드가 아닌 Digital Ocean 로드 밸런서에 연결하십시오.

## 더 보기

{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
