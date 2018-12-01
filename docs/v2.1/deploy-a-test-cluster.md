---
title: Deploy a Test Cluster
summary: Use CockroachDB's CloudFormation template to deploy a Kubernetes-orchestrated test cluster on AWS.
toc: true
---

이 페이지에서는 CockroachDB의 [AWS CloudFormation](https://aws.amazon.com/cloudformation/) 템플릿을 사용하여 설치를 간소화하고, 라이언트 작업 부하의 배포, 유지 관리 및 로드 균형 조정을 자동화하는 [Kubernetes](https://kubernetes.io/)를 사용하여 안전하지 않은 다중 노드 CockroachDB 클러스터를 테스트하는 가장 쉬운 방법을 보여줍니다.

<!-- {{site.data.alerts.callout_success}}이 튜토리얼에서는 <a href="https://github.com/cockroachdb/cockroach/wiki/Roadmap"> 로드맵 </a>의 시험판 기능을 평가할 수 있는 CockroachDB v2.0 alpha 바이너리를 제공합니다. 최신 안정 버전을 테스트하려면, 이 페이지의 <a href="../v1.1/deploy-a-test-cluster.html"> v1.1 버전 </a>을 사용하십시오.{{site.data.alerts.end}} -->


## 시작하기 전에

시작하기 전에 몇 가지 제한 사항과 요구 사항을 검토하는 것이 중요합니다.

### 제한 사항

{{site.data.alerts.callout_danger}}<a href="https://github.com/cockroachdb/cockroachdb-cloudformation"> CockroachDB AWS CloudFormation 템플릿 </a>은 프로덕션 용도가 아닌 테스트 용으로 설계되었습니다.{{site.data.alerts.end}}

- 클러스터를 최대 15개의 노드까지 확장할 수 있습니다.

- 배치를위한 AWS 영역은 구성 가능하지만, 클러스터는 해당 영역 내의 단일 AWS 가용성 영역에서 실행됩니다. 최소 3개의 노드를 배치하는 한 노드 장애로부터 쉽게 복구되고 복구되지만, 가용성 영역 장애가 발생해도 복구되지 않습니다.
    - 프로덕션 복원력의 경우, 단일 영역 또는 3개 이상의 영역에 3개 이상의 가용성 영역을 확장하는 것이 좋습니다.

- 클러스터가 완전히 불안정하므로 다음과 같은 위험이 발생합니다:
    - 네트워크 암호화 또는 인증이 없으므로 기밀성이 없습니다.
    - 클러스터는 특정 범위의 IP 주소로 클라이언트 액세스를 제한하는 옵션이 있지만, 기본적으로 모든 클라이언트에 열려 있습니다.
    - 모든 사용자, `root`조차도 패스워드를 제공하지 않고 로그인할 수 있습니다.
    - `root`로 연결하는 모든 사용자는 클러스터의 모든 데이터를 읽거나 쓸 수 있습니다.

### 요구 사항

- [AWS 계정](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-sign-up-for-aws.html)이 있어야 합니다.
- 클러스터가 배포된 AWS 영역에 [SSH 액세스](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)가 있어야 합니다.

## 1단계. CockroachDB 시작

1. [CockroachDB CloudFormation 템플릿 시작](http://amzn.to/2CZjJLZ).

2. CloudFormation UI에서 클러스터 설정을 검토하고 사용자 정의하십시오. 대부분의 기본값은 시나리오 테스트에 충분합니다. 그러나 **SSH Key**를 선택하여 나중에 Kubernetes 마스터 노드에 연결하고 **CockroachDB Version**을 **v2.0** 옵션으로 설정할 수 있다는 것이 중요합니다. 

    또한 다음 작업을 수행 할 수도 있습니다:
    - 클러스터가 실행될 **AWS region**을 변경하십시오. 기본 지역은 **US West**입니다. 일부 지역에서는 일부 인스턴스 유형을 사용하지 못할 수도 있습니다.
    - **IP Address Whitelist**를 추가하여 CockroachDB Admin UI에 대한 사용자 액세스 및 클러스터에 대한 SQL 클라이언트 액세스를 제한하십시오. 기본적으로 모든 위치에 액세스할 수 있습니다.
    - 초기 **Cluster Size**를 늘리십시오. 기본값은 **3** 노드입니다.

3. **Load Generators** 섹션에서, 클러스터에 대해 실행하려는 **Workload** 유형을 선택하십시오.

4. 클러스터를 시작할 준비가되면, **Create**을 클릭하십시오.

    개시 과정은 일반적으로 10분에서 15분 정도 소요됩니다. CloudFormation UI에서 `CREATE_COMPLETE`상태를 확인하면, 클러스터를 테스트 할 수 있습니다.

    {{site.data.alerts.callout_info}}실행 프로세스가 시간 초과되거나 실패하면, <a href="https://docs.aws.amazon.com/general/latest/gr/aws_service_limits.html"> AWS 서비스 한도</a>로 실행 중일 수 있습니다. <a href=" https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-monitor-stack.html" data-proofer-ignore> 이벤트 기록</a>에서 오류를 볼 수 있습니다.{{site.data.alerts.end}}

## 2단계. 클러스터 테스트

1. 아직 설치하지 않은 경우, 로컬 머신에 [CockroachDB를 설치](install-cockroachdb.html)하십시오. 

2. CloudFormation UI의 **Outputs** 섹션에서 **Connection String**을 확인하십시오.

3. 터미널에서 `--url` 플래그로 **Connection String**을 사용하여 `cockroach` 바이너리에 내장된 [SQL 쉘](use-built-in-sql-client.html)을 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql \
    --insecure \
    --url="postgresql://root@Cockroach-ApiLoadB-LVZZ3VVHMIDA-1266691548.us-west-2.elb.amazonaws.com:26257?sslmode=disable"
    ~~~

4. 몇 가지 기본 [CockroachDB SQL 문](learn-cockroachdb-sql.html)을 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE bank;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO bank.accounts VALUES (1, 1000.50);
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > SELECT * FROM bank.accounts;
    ~~~

    ~~~
    +----+---------+
    | id | balance |
    +----+---------+
    |  1 |  1000.5 |
    +----+---------+
    (1 row)
    ~~~

{{site.data.alerts.callout_success}}로컬 컴퓨터의 <code>cockroach</code> 바이너리를 사용하면, 다른 클라이언트 <a href="cockroach-commands.html"><code>cockroach</code> commands</a>을 동일한 방식으로 실행할 수 있습니다.{{site.data.alerts.end}}

## 3단계. 클러스터 모니터링

클러스터의 [Admin UI](admin-ui-overview.html)를 사용하여 워크로드 및 전체 클러스터 동작을 모니터링 할 수 있습니다.

1. CloudFormation UI의 **Outputs** 섹션에서 **Web UI** 링크를 클릭하십시오. 그런 다음 왼쪽 탐색 메뉴에서 **Metrics**를 클릭하십시오.

2. **Overview** 대시 보드에서, **SQL Queries** 그래프 위로 마우스를 가져가면 로드 생성기에서 오는 읽기 및 쓰기 비율을 볼 수 있습니다.

    <img src="{{ 'images/v2.1/cloudformation_admin_ui_sql_queries.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

3. 아래로 스크롤하여 **Replicas per Node** 그래프 위로 마우스를 가져 가면 CockroachDB가 자동으로 데이터를 백그라운드에서 복제하는 방법을 확인할 수 있습니다.

    <img src="{{ 'images/v2.1/cloudformation_admin_ui_replicas.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

4. [Admin UI](admin-ui-overview.html)의 다른 영역을 탐색하십시오.

5. [프로덕션 모니터링 및 경고](monitoring-and-alerting.html)에 대해 자세히 알아보십시오.

## 4단계. 노드 오류 시뮬레이션

Kubernetes는 클러스터가 초기 구성 중에 지정한 노드의 수를 항상 유지하도록 합니다 (기본적으로 3개). 노드에 장애가 발생하면, Kubernetes는 동일한 네트워크 ID 및 영구 저장 장치를 사용하여 자동으로 다른 노드를 생성합니다.

이 작업을 보려면:

1. CloudFormation UI의 **Outputs** 섹션에서 **SSHProxyCommand**를 확인하십시오.

2. 새 터미널에서, SSH에 대한 **SSHProxyCommand**를 Kubernetes 마스터 노드로 실행하십시오. `.pem` 파일의 위치를 가리키도록 `SSH_KEY` 환경 변수 정의를 업데이트하십시오.

3. CockroachDB 노드에 매핑되는 Kubernetes 포드를 나열하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    ~~~
    NAME            READY     STATUS    RESTARTS   AGE
    cockroachdb-0   1/1       Running   0          1h
    cockroachdb-1   1/1       Running   0          1h
    cockroachdb-2   1/1       Running   0          1h
    ~~~

3. CockroachDB 노드 중 하나를 종료하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl delete pod cockroachdb-2
    ~~~

    ~~~
    pod "cockroachdb-2" deleted
    ~~~

4. Admin UI에서, **Cluster Overview** 패널에 한 노드가 **Suspect**로 표시 될 수 있습니다. Kubernetes가 노드를 자동으로 다시 시작하면, 노드가 다시 건강해지는 모습을 관찰하십시오.

    또한 **Runtime** 대시 보드를 선택하고 **Live Node Count** 그래프에서 노드를 다시 시작할 수 있습니다.

    <img src="{{ 'images/v2.1/cloudformation_admin_ui_live_node_count.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## 5단계. 클러스터 중지

CloudFormation UI에서 **Other Actions** > **Delete Stack**를 선택하십시오. 이것은 클러스터에 연결된 모든 AWS 자원을 삭제할 때 필수적입니다. 이러한 자원을 삭제하지 않으면 AWS에서 계속해서 비용을 청구합니다.

## 더 보기

{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
