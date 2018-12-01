---
title: Deploy CockroachDB On-Premises (Insecure)
summary: Learn how to manually deploy an insecure, multi-node CockroachDB cluster on multiple machines.
toc: true
ssh-link: https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2
redirect_from: manual-deployment-insecure.html

---

<div class="filters filters-big clearfix">
  <a href="deploy-cockroachdb-on-premises.html"><button class="filter-button">Secure</button></a>
  <a href="deploy-cockroachdb-on-premises-insecure.html"><button class="filter-button current"><strong>Insecure</strong></button></a>
</div>

이 튜토리얼에서는 [HAProxy](http://www.haproxy.org/) 로드 밸런서를 사용하여 클라이언트 트래픽을 분산하여 안전하지 않은 다중 노드 CockroachDB 클러스터를 여러 시스템에 수동으로 배포하는 방법을 보여줍니다.

{{site.data.alerts.callout_danger}}CockroachDB를 프로덕션에 사용하려는 경우, 보안 클러스터를 사용하는 것이 좋습니다. 지침을 보려면 위의 <strong>Secure</strong>을 선택하십시오.{{site.data.alerts.end}}

## 요구 사항

{% include {{ page.version.version }}/prod-deployment/insecure-requirements.md %}

## 권장 사항

{% include {{ page.version.version }}/prod-deployment/insecure-recommendations.md %}

## 1단계. 클럭 동기화

{% include {{ page.version.version }}/prod-deployment/synchronize-clocks.md %}

## 2단계. 노드 시작

{% include {{ page.version.version }}/prod-deployment/insecure-start-nodes.md %}

## 3단계. 클러스터 초기화

{% include {{ page.version.version }}/prod-deployment/insecure-initialize-cluster.md %}

## 4단계. 클러스터 테스트

{% include {{ page.version.version }}/prod-deployment/insecure-test-cluster.md %}

## 5단계. HAProxy로드 밸런서 설정

각 CockroachDB 노드는 클러스터에 똑같이 적합한 SQL 게이트웨이이지만, 클라이언트 성능과 안정성을 보장하려면 로드 밸런싱을 사용하는 것이 중요합니다:

- **성능:** 로드 밸런서는 클라이언트 트래픽을 노드에 분산시킵니다. 이렇게 하면 모든 노드가 요청에 압도당하는 것을 방지하고 전체 클러스터 성능 (초당 쿼리 수)을 향상시킬 수 있습니다.

- **신뢰성:** 로드 밸런서는 클라이언트 상태를 단일 CockroachDB 노드의 상태와 분리합니다. 노드가 실패 할 경우 로드 밸런서는 클라이언트 트래픽을 사용 가능한 노드로 다시 전송합니다.{{site.data.alerts.callout_success}} 단일 로드 밸런서를 사용하는 경우 클라이언트 연결은 노드 장애에 복원력이 있지만 로드 밸런서 자체는 장애 지점입니다. 따라서 클라이언트를 위한 로드 밸런서를 선택하려면 부동 IP 또는 DNS와 같은 메커니즘을 사용하여 다중 로드 밸런싱 인스턴스를 사용하여 로드 밸런싱을 복원하고 복원력을 높이는 것이 가장 좋습니다. {{site.data.alerts.end}}

[HAProxy](http://www.haproxy.org/)는 가장 인기 있는 오픈 소스 TCP 로드 밸런서 중 하나이며, CockroachDB에는 실행 중인 클러스터에서 작동하도록 미리 설정된 구성 파일을 생성하기 위한 빌트인 명령이 포함되어 있으므로, 여기에 해당 툴이 있습니다.

1. 당신이 HAProxy를 실행하고 싶은 머신에 SSH.

2. HAProxy 설치:

    {% include copy-clipboard.html %}
	~~~ shell
	$ apt-get install haproxy
	~~~

3. Linux의 경우 [CockroachDB 아카이브](https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz)를 다운로드하고, 바이너리를 추출하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget -qO- https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz \
    | tar  xvz
    ~~~

4. 바이너리를 `PATH`에 복사하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cp -i cockroach-{{ page.release_info.version }}.linux-amd64/cockroach /usr/local/bin
    ~~~

	권한 오류가 발생하면, 명령 앞에 `sudo`를 추가하십시오.

5. CockroachDB 노드의 주소를 지정하여 [`cockroach gen haproxy`](generate-cockroachdb-resources.html) 명령을 실행하십시오:

    {% include copy-clipboard.html %}
	  ~~~ shell
	  $ cockroach gen haproxy --insecure \
	  --host=<address of any node> \
	  --port=26257 \
	  ~~~

      {% include {{ page.version.version }}/misc/haproxy.md %}

6. `haproxy.cfg` 파일을 가리키는 `-f` 플래그로 HAProxy를 시작하십시오:

    {% include copy-clipboard.html %}
	~~~ shell
	$ haproxy -f haproxy.cfg
	~~~

7. 실행하려는 HAProxy의 추가 인스턴스마다 이 단계를 반복하십시오.

## 6단계. 샘플 워크로드 실행

{% include {{ page.version.version }}/prod-deployment/insecure-test-load-balancing.md %}

## 7단계. 모니터링 및 경고 설정

{% include {{ page.version.version }}/prod-deployment/monitor-cluster.md %}

## 8단계. 클러스터 크기 조정

{% include {{ page.version.version }}/prod-deployment/insecure-scale-cluster.md %}

## 9단계. 클러스터 사용

{% include {{ page.version.version }}/prod-deployment/use-cluster.md %}

## 더 보기

{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
