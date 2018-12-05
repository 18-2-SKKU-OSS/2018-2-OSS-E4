---
title: Orchestrate CockroachDB with Mesosphere DC/OS (Insecure)
summary: How to orchestrate the deployment and management of an insecure 3-node CockroachDB cluster with Mesosphere DC/OS.
toc: true
---

이 페이지에서는 [Mesosphere DC/OS](https://mesosphere.com/)를 사용하여 인시큐어한 3-노드 CockroachDB 클러스터의 배포 및 관리를 조정하는 방법을 보여줍니다.

{{site.data.alerts.callout_danger}}프로덕션 데이터의 경우 <strong>인시큐어한</strong> 클러스터를 배포하는 것은 권장되지 않습니다. 이 페이지는 Mesosphere DC/OS를 사용하여 시큐어 클러스터를 조정할 수 있게 되면 업데이트될 것입니다.{{site.data.alerts.end}}

## 시작하기 전에

시작하기 전에 몇 가지 현재 요구 사항과 제한 사항을 검토해야 합니다.

### 요구 사항

- 클러스터에는 최소 3개의 프라이빗 노드가 있어야 합니다.
- Enterprise DC/OS를 사용하는 경우 CockroachDB를 설치하기 전에 [서비스 계정 프로비저닝](https://docs.mesosphere.com/1.9/security/ent/service-auth/custom-service-auth/)이 필요할 수 있습니다. `superuser` 권한을 가진 사람만이 이러한 서비스 계정을 만들 수 있습니다.

    보안 모드 | 서비스 계정
    --------------|----------------
    `strict` | 필수
    `permissive` | 선택
    `disabled` | 필요 없음

### 제한 사항

DC/OS의 CockroachDB는 다음과 같은 제한 사항을 제외하고는 다른 환경과 동일하게 작동합니다:

-`cockroachdb` DC/OS 서비스는 DC/OS 버전 1.9 및 1.10에서만 테스트되었습니다.
- 시큐어 모드로 실행하는 것은 현재 지원되지 않습니다.
- 멀티 데이터 센터 클러스터 실행은 현재 지원되지 않습니다.
- 노드 제거는 현재 지원되지 않습니다.
- 초기 배포 후에 볼륨 유형이나 볼륨 크기 요구 사항이 변경되지 않을 수 있습니다.
- 랙(rack) 배치 및 인식은 현재 지원되지 않습니다.

## 1단계. DC/OS 설치 및 시작

시작 및 실행하는 가장 빠른 방법은 [AWS CloudFormation의 오픈 소스 DC/OS 템플릿](https://docs.mesosphere.com/1.10/installing/oss/cloud/aws/basic/)을 사용하는 것입니다. 그러나 공식 문서에서 다른 오픈 소스 DC/OS 또는 엔터프라이즈 DC/OS 설치 방법에 대한 세부 정보를 찾을 수 있습니다:

- [오픈 소스 DC/OS](https://docs.mesosphere.com/1.10/installing/oss/)
- [엔터프라이즈 DC/OS](https://docs.mesosphere.com/1.10/installing/ent/)

AWS CloudFormation을 사용할 때 시작 과정은 일반적으로 10-15 분이 걸립니다. CloudFormation UI에서 `CREATE_COMPLETE` 상태를 확인한 후 [DC/OS를 실행](https://docs.mesosphere.com/1.10/installing/oss/cloud/aws/basic/#launch-dcos)하고 [DC/OS CLI를 설치](https://docs.mesosphere.com/1.10/cli/install/)하십시오.

## 2단계. CockroachDB 시작

1. 기본 CockroachDB 설정을 검토하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ dcos package describe --config cockroachdb
    ~~~

    [기본 CockroachDB 설정](https://github.com/cockroachdb/dcos-cockroachdb-service#node-settings)은 합리적인 기본값을 가진 3-노드 CockroachDB 클러스터를 생성하지만 배포 환경에 따라 다른 설정이 필요할 수 있습니다. 배포 환경을 사용자 정의하려면 위 명령의 출력을 기반으로 `cockroach.json` 파일을 작성하십시오. [권장 배포 환경](recommended-production-settings.html)을 참고하십시오.

2. CockroachDB 클러스터를 DC/OS 서비스로 시작하십시오.

    - 기본 설정을 사용하는 경우 다음을 실행하십시오:

        {% include copy-clipboard.html %}
        ~~~ shell
        $ dcos package install cockroachdb
        ~~~
    - 사용자 정의`cockroach.json` 설정을 생성했다면 다음을 실행하십시오:

        {% include copy-clipboard.html %}
        ~~~ shell
        $ dcos package install cockroachdb --options=cockroach.json
        ~~~

3. DC/OS UI의 **서비스** 탭에서 클러스터 배포를 모니터링하십시오.

{{site.data.alerts.callout_info}}<a href="https://docs.mesosphere.com/1.10/deploying-services/install/#installing-a-service-using-the-gui">DC/OS UI에서도 CockroachDB를 설치</a>할 수 있습니다.{{site.data.alerts.end}}

## 3단계. 클러스터 테스트

1. CockroachDB 클러스터의 엔드 포인트를 찾으십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ dcos cockroachdb endpoints pg
    ~~~

    ~~~
    {
      "address": [
        "10.0.0.212:26257",
        "10.0.2.57:26257",
        "10.0.3.81:26257"
      ],
      "dns": [
        "cockroachdb-0-node-init.cockroachdb.autoip.dcos.thisdcos.directory:26257",
        "cockroachdb-1-node-join.cockroachdb.autoip.dcos.thisdcos.directory:26257",
        "cockroachdb-2-node-join.cockroachdb.autoip.dcos.thisdcos.directory:26257"
      ],
      "vip": "pg.cockroachdb.l4lb.thisdcos.directory:26257"
    }
    ~~~

    리턴된 엔드 포인트는 다음을 포함합니다.
     - DC/OS 클러스터 내에서 이동하는 인스턴스들 뒤에 오는 각 인스턴스의 `.mesos` 호스트 이름.
     - `.mesos` 호스트 이름을 확인할 수 없는 경우 각 인스턴스의 직접 IP 주소.
     - 임의의 인스턴스에 접근하기 위한 HA 지원 호스트 이름인 `vip` 주소. 다음 단계의 일부에서 `vip` 주소를 사용할 것입니다.

    일반적으로, `.mesos` 엔드 포인트들은 동일한 DC/OS 클러스터에서만 작동합니다. 클러스터 밖에서는 직접 IP를 사용하거나 CockroachDB 인스턴스의 프론트 엔드 역할을 하는 프록시 서비스를 설정할 수 있습니다. 개발 및 테스트 목적으로 [DC/OS 터널](https://docs.mesosphere.com/1.10/developing-services/tunnel/)을 사용하여 클러스터 외부의 서비스에 접근할 수 있지만 이 옵션은 배포용으로 적합하지 않습니다. 자세한 내용은 아래 [클러스터 모니터링](#step-4-monitor-the-cluster)을 참조하십시오.

2. [SSH를 DC/OS 마스터 노드에 연결](https://docs.mesosphere.com/1.10/administering-clusters/sshcluster/)하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ dcos node ssh --master-proxy --leader
    ~~~

3. 임시 컨테이너를 시작하고 `vip` 엔드 포인트를 `--host`로 사용하여 [빌트인 SQL 쉘](built-in-sql-client.html)을 여십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ docker run -it {{ page.release_info.docker_image }}:{{ page.release_info.version }}  sql --insecure --host=pg.cockroachdb.l4lb.thisdcos.directory
    ~~~

    ~~~
    # Welcome to the cockroach SQL interface.
    # All statements must be terminated by a semicolon.
    # To exit: CTRL + D.
    root@pg.cockroachdb.l4lb.thisdcos.directory:26257/>
    ~~~

4. 몇 가지 기본 [CockroachDB SQL 명령문](learn-cockroachdb-sql.html)을 실행하십시오.

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

5. SQL 쉘을 종료하고 임시 포드(pod)를 삭제하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > \q
    ~~~

## 4단계. 클러스터 모니터링

클러스터의 [Admin UI](explore-the-admin-ui.html)에 접근하려면 [DC/OS 터널을 사용하여 HTTP 프록시를 실행](https://docs.mesosphere.com/1.10/developing-services/tunnel/#using-dcos-tunnel)할 수 있습니다:

1. DC/OS 터널 패키지를 설치하십시오:

     ~~~ shell
     $ dcos package install tunnel-cli --cli
     ~~~

2. DC/OS 터널을 시작하십시오:

    ~~~ shell
    $ sudo dcos tunnel http
    ~~~

3. 브라우저에서 <a href="http.cockroachdb.l4lb.thisdcos.directory.mydcos.directory" data-proofer-ignore>http://http.cockroachdb.l4lb.thisdcos.directory.mydcos.directory</a>로 이동하십시오.

## 5단계. 클러스터 확장

기본 `cockroachdb` 서비스는 3-노드 CockroachDB 클러스터를 생성합니다. 스케줄러 프로세스를 업데이트함으로써 서비스가 시작된 후에도 노드를 클러스터에 추가할 수 있습니다:

1. DC/OS UI에서 **서비스** 표로 이동하십시오.
2. **cockroachdb** 서비스를 선택하십시오.
3. 오른쪽 상단에서 **편집**을 선택하십시오.
4. **환경**을 선택하십시오.
5. `NODE_COUNT` 변수를 원하는 CockroachDB 노드 수와 일치하도록 업데이트하십시오.
6. **검토 및 실행**을 클릭한 다음 **서비스 실행**을 클릭하십시오.

스케줄러 프로세스가 새로운 설정으로 다시 시작되고 감지된 변경 사항의 유효성을 검사합니다. 노드가 클러스터에 성공적으로 추가되었는지 확인하려면 Admin UI로 돌아가서 **Node List**를 보고 새 노드를 확인하십시오.

또는 [SSH를 DC/OS 마스터 노드에 연결](https://docs.mesosphere.com/1.10/administering-clusters/sshcluster/)한 다음, `vip` 엔드 포인트를 `--host`로 다시 사용하여 임시 컨테이너에서 [`cockroach node status`](view-node-details.html) 명령을 실행할 수 있습니다:

{% include copy-clipboard.html %}
~~~ shell
$ dcos node ssh --master-proxy --leader
~~~

{% include copy-clipboard.html %}
~~~ shell
$ docker run -it {{ page.release_info.docker_image }}:{{ page.release_info.version }} node status --all --insecure --host=pg.cockroachdb.l4lb.thisdcos.directory
~~~

~~~
+----+--------------------------------------------------------------------------+----------------------+---------------------+---------------------+------------+-----------+-------------+--------------+--------------+------------------+-----------------------+--------+--------------------+-----------------------+
| id |                                 address                                  | build                |     updated_at      |     started_at      | live_bytes | key_bytes | value_bytes | intent_bytes | system_bytes | replicas_leaders | replicas_leaseholders | ranges | ranges_unavailable | ranges_underreplicated |
+----+--------------------------------------------------------------------------+----------------------+---------------------+---------------------+------------+-----------+-------------+--------------+--------------+------------------+-----------------------+--------+--------------------+-----------------------+
|  1 | cockroachdb-0-node-init.cockroachdb.autoip.dcos.thisdcos.directory:26257 | {{ page.release_info.version }} | 2017-12-11 20:59:12 | 2017-12-11 19:14:42 |   41183973 |      1769 |    41187432 |            0 |         6018 |                2 |                     2 |      2 |                  0 |                      0 |
|  2 | cockroachdb-1-node-join.cockroachdb.autoip.dcos.thisdcos.directory:26257 | {{ page.release_info.version }} | 2017-12-11 20:59:12 | 2017-12-11 19:14:52 |     115448 |     71037 |      209282 |            0 |         6218 |                4 |                     4 |      4 |                  0 |                      0 |
|  3 | cockroachdb-2-node-join.cockroachdb.autoip.dcos.thisdcos.directory:26257 | {{ page.release_info.version }} | 2017-12-11 20:59:03 | 2017-12-11 19:14:53 |     120325 |     72652 |      217422 |            0 |         6732 |                4 |                     3 |      4 |                  0 |                      0 |
|  4 | cockroachdb-3-node-join.cockroachdb.autoip.dcos.thisdcos.directory:26257 | {{ page.release_info.version }} | 2017-12-11 20:59:03 | 2017-12-11 20:21:43 |   41248030 |     79147 |    41338632 |            0 |         6569 |                1 |                     1 |      1 |                  0 |                      0 |
|  5 | cockroachdb-4-node-join.cockroachdb.autoip.dcos.thisdcos.directory:26257 | {{ page.release_info.version }} | 2017-12-11 20:59:04 | 2017-12-11 20:56:54 |   41211967 |     30550 |    41181417 |            0 |         6854 |                1 |                     1 |      1 |                  0 |                      0 |
+----+--------------------------------------------------------------------------+--------+---------------------+---------------------+------------+-----------+-------------+--------------+--------------+------------------+-----------------------+--------+--------------------+-------------+-----------------------+
(5 rows)
~~~

## 6단계. 클러스터 유지 관리

관련 유지 관리 작업을 선택하십시오:

- [구성 업테이트](#update-configurations)
- [노드 재시작](#restart-a-node)
- [노드 교체](#replace-a-node)
- [문제 해결(Troubleshoot) (로그 액세스)](#troubleshoot-access-logs)
- [백업 및 복원](#backup-and-restore)

### 구성 업데이트

노드를 추가하는 것 외에도 노드에 대한 [CPU 및 메모리 요구 사항](https://github.com/cockroachdb/dcos-cockroachdb-service#resizing-a-node)을 변경하고 [위치 제한 사항](https://github.com/cockroachdb/dcos-cockroachdb-service#resizing-a-node), [기타 서비스 설정](https://github.com/cockroachdb/dcos-cockroachdb-service#service-settings)을 변경할 수 있습니다. 이전 단계의 지침을 따르되 수행할 업데이트에 대한 환경 변수를 변경하십시오.

- 변경 후에는 스케줄러가 재시작되어 감지된 변경사항을 서비스에 한 번에 한 노드씩 자동으로 배치합니다. 예를 들어, 주어진 변경 사항은 먼저 `cockroachdb-0`에 적용된 다음 `cockroachdb-1` 등에 적용됩니다.
- 노드는 순서의 다음 노드에 지정된 변경 사항을 적용하기 전에 기본 서비스가 정상(healthy) 상태인지 확인하기 위해 "준비 검사(Readiness check)"로 구성됩니다. 그러나 이러한 기본적인 점검은 절대 안전한 것은 아니며 주어진 구성 변경이 서비스 동작에 부정적인 영향을 미치지 않도록 적절한 주의를 기울여야 합니다. 

### 노드 재시작

현재 영구 볼륨 데이터로 현재 위치에 유지하면서 노드를 재시작할 수 있습니다. 시스템 프로세스 재시작과 유사하다고 생각될 수 있지만 이는 영구 볼륨에 없는 데이터도 삭제합니다.

1. 재시작하려는 노드의 포드 이름을 가져오십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ dcos cockroachdb pod list
    ~~~~

    ~~~~
    [
      "cockroachdb-0",
      "cockroachdb-1",
      "cockroachdb-2",
      "cockroachdb-3",
      "cockroachdb-4",
      "metrics-0"
    ]
    ~~~~

2. 관련 포드를 재시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ dcos cockroachdb pods restart cockroachdb-<NUM>
    ~~~

### 노드 교체

노드를 새 시스템으로 이동하고 이전 시스템에서 영구 볼륨을 삭제하여 새 시스템에서 재구성 할 수 있습니다. 노드는 자동으로 이동하지 않으므로, 예를 들어 시스템이 오프라인되기 전이나 시스템이 이미 오프라인 상태일 때 이 단계를 수행해야 합니다.

1. 재시작하려는 노드의 포드 이름을 가져오십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ dcos cockroachdb pod list
    ~~~~

    ~~~~
    [
      "cockroachdb-0",
      "cockroachdb-1",
      "cockroachdb-2",
      "cockroachdb-3",
      "cockroachdb-4",
      "metrics-0"
    ]
    ~~~~

2. DC/OS 클러스터의 새 위치에서 포드를 중지했다가 다시 시작하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ dcos cockroachdb pods replace cockroachdb-<NUM>
    ~~~

### 문제 해결(Troubleshoot) (로그 액세스)

스케줄러 및 서비스(예: CockroachDB)에 대한 로그는 DC/OS 웹 인터페이스에서 볼 수 있습니다.

- 스케줄러 로그는 노드가 시작되지 않는 이유를 결정하는 데 유용합니다(이는 스케줄러의 권한하에 있습니다).
- 노드 로그는 서비스 자체(예: CockroachDB)의 문제를 조사하는 데 유용합니다.

모든 경우에 로그는 일반적으로 stdout 및/또는 stderr 라는 파일에 파이프됩니다.

주어진 노드의 로그를 보려면:

1. DC/OS UI에서 **서비스** 표로 이동하십시오.
2. **cockroachdb** 서비스를 선택하십시오.
3. 서비스 작업 목록에서 검사할 작업을 선택합니다. 스케줄러는 이 서비스의 이름을 따서 명명되고, 노드는 `cockroachdb-0-node-init` 또는 `cockroachdb-#-node-join`입니다.
4. 작업 세부 정보에서 **로그** 탭으로 이동하십시오.

### 백업 및 복원

`cockroachdb` DC/OS 서비스는 CockroachDB의 오픈 소스 [`cockroach dump`](sql-dump.html) 명령을 사용하여 데이터베이스별로 데이터를 S3 버킷에 백업하고 해당 백업에서 데이터를 복원할 수 있는 쉬운 방법을 제공합니다. S3이 아닌 다른 데이터 저장소를 사용하는 것은 아직 지원되지 않습니다.

{{site.data.alerts.callout_success}}S3 이외의 데이터 소스를 백업/복원해야 하거나 데이터베이스가 매우 커서 <a href="backup.html">더 빠른 백업</a>, <a href="backup.html#incremental-backups">증분 백업(incremental backups)</a> 또는 <a href="restore.html">더 빠른 분산 복원 프로세스</a>가 필요한 경우 Cockroach Labs에 <a href="https://www.cockroachlabs.com/pricing/">엔터프라이즈 라이센스</a>에 대해 문의하십시오.{{site.data.alerts.end}}

#### 백업

데이터베이스의 테이블을 백업하려면 다음 명령을 실행하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ dcos cockroachdb backup [<flags>] <database> <s3-bucket>
~~~

다음 선택 플래그를 사용하여 S3와의 통신을 구성할 수 있습니다:

플래그 | 설명
-----|------------
`--aws-access-key` | AWS 액세스 키
`--aws-secret-key` | AWS 시크릿 키
`--s3-dir` | AWS S3 목표 경로
`--s3-backup-dir` | s3-dir 내의 목표 경로
`--region` | AWS 지역

기본적으로 AWS 액세스 및 시크릿 키는 각각 `AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY` 환경 변수를 통해 사용자 환경에서 가져올 수 있습니다. 이러한 환경 변수를 정의하거나 백업이 작동할 플래그를 지정해야 합니다.

백업을 수행할 수 있는 충분한 디스크 공간을 노드에 제공하는지 확인하십시오. 백업은 S3에 업로드되기 전에 디스크에 저장되며 현재 테이블에 있는 데이터만큼의 공간을 차지하므로 전체 키 공간을 한 번에 자유롭게 백업하려면 총 사용 가능한 공간의 절반이 필요합니다.

#### 복원

클러스터 데이터를 복원하려면 다음 명령을 실행하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ dcos cockroachdb restore [<flags>] <database> <s3-bucket> <s3-backup-dir>
~~~

CLI 명령에 다음과 같은 선택 플래그를 사용하여 S3와의 통신을 구성 할 수 있습니다:

플래그 | 설명
-----|------------
`--aws-access-key` | AWS 액세스 키
`--aws-secret-key` | AWS 시크릿 키
`--s3-dir` | AWS S3 목표 경로
`--s3-backup-dir` | s3-dir 내의 목표 경로
`--region` | AWS 지역

기본적으로 AWS 액세스 및  키는 각각 `AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY` 환경 변수를 통해 사용자 환경에서 가져올 수 있습니다. 이러한 환경 변수를 정의하거나 백업이 작동할 플래그를 지정해야 합니다.

## 7단계. 클러스터 중지

CockroachDB 클러스터를 종료하려면 다음을 수행하십시오:

1. `cockroachdb` 서비스를 제거하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ MY_SERVICE_NAME=cockroachdb
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ dcos package uninstall --app-id=$MY_SERVICE_NAME $MY_SERVICE_NAME
    ~~~

2. DC/OS 버전 1.9를 사용하는 경우 다음 명령으로 프레임워크 클리너 스크립트 [`janitor.py`]를 사용하여 나머지 예약된 리소스를 정리하십시오. 이 단계는 DC / OS 1.10에서는 필요하지 않습니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ dcos node ssh --master-proxy --leader "docker run mesosphere/janitor /janitor.py \
    -r $MY_SERVICE_NAME-role \
    -p $MY_SERVICE_NAME-principal \
    -z dcos-service-$MY_SERVICE_NAME"
    ~~~

3. DC/OS를 제거하십시오. AWS CloudFormation을 사용한 경우 [AWS EC2에서 DC/OS 제거](https://docs.mesosphere.com/1.10/installing/oss/cloud/aws/removeaws/)를 참조하십시오.

## 더 보기

{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
