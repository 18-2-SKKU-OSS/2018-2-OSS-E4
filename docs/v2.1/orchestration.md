---
title: Orchestration
summary: Learn how to run CockroachDB with popular open-source orchestration systems.
toc: false
---
 
오케스트레이션 시스템은 컨테이너식 어플리케이션의 배포, 확장 및 관리를 자동화합니다. CockroachDB의 [자동화된 샤딩](frequently-asked-questions.html#how-does-cockroachdb-scale) 및 [결함 허용](frequently-asked-questions.html#how-does-cockroachdb-survive-failures)과 결합되어, 오퍼레이터 오버헤드를 거의 없앨 수 있습니다.

널리 사용되는 오픈소스 오케스트레이션 시스템과 함께 CockroachDB를 실행하려면 다음 가이드를 사용하십시오:

- [Kubernetes 배포](orchestrate-cockroachdb-with-kubernetes.html)
- [Kubernetes 성능 최적화](kubernetes-performance.html)
- [Docker 배포](orchestrate-cockroachdb-with-docker-swarm.html)
- [Mesosphere DC/OS 배포](orchestrate-cockroachdb-with-mesosphere-insecure.html)

{{site.data.alerts.callout_success}}CockroachDB를 처음 시작하는 경우, <a href="orchestrate-a-local-cluster-with-kubernetes-insecure.html"> 로컬 클러스터를 오케스트레이트</a>하여 데이터베이스의 기본 사항을 배우고 싶을 수 있습니다.{{site.data.alerts.end}}

## 

- [프로덕션 체크리스트](recommended-production-settings.html)
- [수동 배포](manual-deployment.html)
- [모니터링 및 경고](monitoring-and-alerting.html)
- [배포 테스트](deploy-a-test-cluster.html)
- [로컬 배포](start-a-local-cluster.html)
