---
title: Deploy CockroachDB on AWS EC2 (Insecure)
summary: Learn how to deploy CockroachDB on Amazon's AWS EC2 platform.
toc: true
toc_not_nested: true
ssh-link: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html
---

<div class="filters filters-big clearfix">
  <a href="deploy-cockroachdb-on-aws.html"><button class="filter-button">Secure</button>
  <button class="filter-button current"><strong>Insecure</strong></button></a>
</div>

이 페이지에서는 AWS의 관리 로드 밸런싱 서비스를 사용하여 클라이언트 트래픽을 분산하는 안전하지 않은 다중 노드 CockroachDB 클러스터를 Amazon의 AWS EC2 플랫폼에 수동으로 배포하는 방법을 보여줍니다.

{{site.data.alerts.callout_danger}} CockroachDB를 프로덕션에 사용하려는 경우, 보안 클러스터를 대신 사용하는 것이 좋습니다. 지침을 보려면 위의 <strong>Secure</strong>을 선택하십시오. {{site.data.alerts.end}}


## 요구 사항

{% include {{ page.version.version }}/prod-deployment/insecure-requirements.md %}

## 권장 사항

{% include {{ page.version.version }}/prod-deployment/insecure-recommendations.md %}

- CockroachDB를 실행하는 모든 인스턴스는 동일한 보안 그룹의 구성원이어야 합니다.

## 1단계. 네트워크 구성

CockroachDB는 두 포트에서 TCP 통신이 필요합니다:

- 노드 간 통신 (즉, 클러스터로 작동), 애플리케이션이 로드 밸런서에 연결하고 로드 밸런서에서 노드로 전송하기 위한 `26257`
- Admin UI 노출을 위한 `8080`

[보안 그룹의 인바운드 규칙](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html#adding-security-group-rule)을 사용하여 이러한 규칙을 만들 수 있습니다.

#### 노드 간 및 로드 밸런서 노드 간 통신

 필드 | 권장 값
-------|-------------------
 타입 | 사용자 지정 TCP 규칙
 프로토콜 | TCP
 포트 범위 | **26257**
 소스 | 보안 그룹의 이름 (예 : * sg-07ab277a *)

#### Admin UI

 필드 | 권장 값
-------|-------------------
 타입 | 사용자 지정 TCP 규칙
 프로토콜 | TCP
 포트 범위 | **8080**
 소스 | 네트워크의 IP 범위

#### 어플리케이션 데이터

 필드 | 권장 값
-------|-------------------
 타입 | 사용자 지정 TCP 규칙
 프로토콜 | TCP
 포트 범위 | **26257**
 소스 | 애플리케이션의 IP 범위

## 2단계. 인스턴스 생성

클러스터에 사용할 각 노드에 대해 [인스턴스를 생성](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/launching-instance.html)하십시오. 클러스터에 대해 샘플 워크로드를 실행하려면, 해당 워크로드에 대해 별도의 인스턴스를 작성하십시오.

- 적어도 3개의 노드를 실행하여 [생존성을 보장](recommended-production-settings.html)하십시오.

- SSD 지원 [EBS 볼륨](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html) 또는 [인스턴스 저장소 볼륨](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ssd-instance-store.html)와 함께, `m`(범용), `c` (연산 최적화) 또는 `i` (스토리지 최적화) [인스턴스](https://aws.amazon.com/ec2/instance-types/)를 사용하십시오. 예를 들어, Cockroach Labs는 내부 테스트를 위해 `m3.large` 인스턴스 (인스턴스 당 2개의 vCPU 및 7.5 GiB)를 사용했습니다.

- 단일 코어의 로드를 제한하는 [버스트가능한 `t2` ](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/t2-instances.html)를 사용하지 마십시오. 

자세한 내용은 [하드웨어 권장 사항](recommended-production-settings.html) 및 [클러스터 위상 배치](recommended-production-settings.html)를 참조하십시오.

## 3단계. 클럭 동기화

{% include {{ page.version.version }}/prod-deployment/synchronize-clocks.md %}

## 4단계. 로드 밸런싱 설정

각 CockroachDB 노드는 클러스터에 똑같이 적합한 SQL 게이트웨이지만, 클라이언트 성능과 안정성을 보장하려면 로드 밸런싱을 사용하는 것이 중요합니다:

- **성능:** 로드 밸런서는 클라이언트 트래픽을 노드에 분산시킵니다. 이렇게 하면 모든 노드가 요청에 압도당하는 것을 방지하고 전체 클러스터 성능(초당 쿼리 수)을 향상시킬 수 있습니다.

- **신뢰성:** 로드 밸런서는 클라이언트 상태를 단일 CockroachDB 노드의 상태와 분리합니다. 노드가 실패할 경우, 로드 밸런서는 클라이언트 트래픽을 사용 가능한 노드로 재전송합니다.

AWS는 완전히 관리되는 로드 밸런싱을 제공하여 인스턴스간에 트래픽을 분산시킵니다.

1. [AWS 로드 밸런싱 추가](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-increase-availability.html). 다음에 유의하십시오:
	- 로드 밸런서의 포트 **26257**에서 포트 **26257**까지의 TCP 트래픽을 노드에 전송하도록 전달 규칙을 설정합니다.
	- HTTP 포트 **8080** 및 `/health?ready=1` 경로를 사용하도록 상태 검사를 구성하십시오. 이 [상태 엔드 포인트](monitoring-and-alerting.html)는 로드 밸런서가 실시간 상태이지만 요청을 수신할 준비가되지 않은 노드로 트래픽을 보내지 않도록 합니다.
	
2. 로드 밸런서에 대해 제공된 **IP Address**에 유의하십시오. 나중에 로드 밸런싱을 테스트하고 어플리케이션을 클러스터에 연결하는 데 사용합니다.

{{site.data.alerts.callout_info}}AWS의 관리로드 밸런싱 대신 HAProxy를 사용하려면, <a href="deploy-cockroachdb-on-premises-insecure.html">On-Premises</a> 튜토리얼을 참조하십시오.{{site.data.alerts.end}}

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
3. [어플리케이션 연결](install-client-drivers.html). 애플리케이션을 CockroachDB 노드가 아닌 AWS 로드 밸런서에 연결하십시오.

## 더 보기

{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
