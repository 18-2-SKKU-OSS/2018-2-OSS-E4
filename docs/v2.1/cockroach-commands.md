---
title: Cockroach Commands
summary: Learn the commands for configuring, starting, and managing a CockroachDB cluster.
toc: true
---

이 페이지는 특정 플래그 대신 사용할 수 있는 환경 변수들 뿐만 아니라 CockroachDB 클러스터 설정, 시작, 관리를 위한 `cockroach` 명령어들을 소개합니다. 

쉘에서 `cockroach help`을 실행하면 비슷한 지침을 얻을 수 있습니다.


## Commands

Command | Usage
--------|----
[`cockroach start`](start-a-node.html) | 노드를 시작합니다.
[`cockroach init`](initialize-a-cluster.html) | 클러스터를 초기화합니다.
[`cockroach cert`](create-security-certificates.html) | CA, 노드 및 클라이언트 인증서를 만듭니다. 
[`cockroach quit`](stop-a-node.html) | 일시적으로 노드를 중지하거나 노드를 영구적으로 제거합니다.
[`cockroach sql`](use-the-built-in-sql-client.html) | 내장된 SQL 클라이언트를 사용합니다.
[`cockroach user`](create-and-manage-users.html) | 사용자를 얻고, 설정하고, 나열하고, 제거합니다.
[`cockroach zone`](configure-replication-zones.html) | **사용되지 않음** 특정 데이터 세트에 대한 복제본의 수와 위치를 구성하려면 [`ALTER ... CONFIGURE ZONE`](configure-zone.html)와 [`SHOW ZONE CONFIGURATIONS`](show-zone-configurations.html)를 사용합니다.
[`cockroach node`](view-node-details.html) | 노드 ID를 나열하고, 상태를 표시하고, 제거할 노드를 제거하거나, 노드를 다시 제안합니다.
[`cockroach dump`](sql-dump.html) | 테이블과 모든 행을 다시 생성하는 데 필요한 SQL 문을 출력하여 테이블을 백업합니다.
[`cockroach demo`](cockroach-demo.html) | 임시 메모리 내 단일 노드 CockroachDB 클러스터를 시작하고 여기에 대화식 SQL 쉘을 엽니다.
[`cockroach gen`](generate-cockroachdb-resources.html) | 실행중인 클러스터에 대한 manpages, bash 완료 파일, 예제 SQL 데이터 또는 HAProxy 구성 파일을 생성합니다.
[`cockroach version`](view-version-details.html) | CockroachDB 버전 정보를 출력합니다.
[`cockroach debug zip`](debug-zip.html) | Cockroach Labs가 클러스터 문제를 해결하는 데 도움이되는 `.zip` 파일을 생성합니다.

## 환경 변수들

`--port`와`--user`와 같은 많은 일반적인 `cockroach` 플래그들에 대해 명령을 실행할 때마다 수동으로 플래그를 전달하는 대신 한 번 환경 변수를 설정할 수 있습니다.

- 환경 변수를 지원하는 플래그를 찾으려면 각 [command](#commands)에 대한 문서를 참조하십시오.
- CockroachDB와 환경 변수들의 현재 구성을 출력하려면 `env`를 실행하십시오.
- 노드가 [startup](start-a-node.html)에서 환경 변수를 사용하면 변수 이름이 노드의 로그에 인쇄됩니다. 그러나 변수 값은 그렇지 않습니다.

CockroachDB는 명령 플래그, 환경 변수 및 기본값을 다음과 같이 우선 순위를 지정합니다.

1. 명령에 플래그가 설정되어 있으면 CockroachDB가 그것을 사용합니다.
2. 명령에 플래그가 설정되어 있지 않으면 CockroachDB는 해당하는 환경 변수를 사용합니다.
3. 플래그와 환경 변수가 모두 설정되지 않은 경우 CockroachDB는 플래그에 대해 기본값을 사용합니다.
5. 플래그 기본값이 없는 경우 CockroachDB는 오류를 발생시킵니다.

자세한 내용은 [Client Connection Parameters](connection-parameters.html)를 참조하십시오.
