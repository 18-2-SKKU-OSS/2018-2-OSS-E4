---
title: Upgrade to CockroachDB v2.1
summary: Learn how to upgrade your CockroachDB cluster to a new version.
toc: true
toc_not_nested: true
---

CockroachDB의 [다중-활성 가용성](multi-active-availability.html) 설계로 인해 CockroachDB 클러스터의 "롤링 업그레이드"를 수행할 수 있습니다. 
즉, 클러스터의 전반적인 상태 및 작업을 방해하지 않고 한 번에 하나씩 노드를 업그레이드 할 수 있습니다.

{{site.data.alerts.callout_info}}
This page shows you how to upgrade to the latest v2.1 release ({{page.release_info.version}}) from v2.0.x, or from any patch release in the v2.1.x series. To upgrade within the v2.0.x series, see [the v2.0 version of this page](https://www.cockroachlabs.com/docs/v2.0/upgrade-cockroach-version.html).
{{site.data.alerts.end}}

## 1단계. 업그레이드 할 수 있는지 확인. 

업그레이드 할 때, 패치 릴리스를 건너 뛸 수는 있지만, **전체 릴리스를 건너 뛸 수는 없습니다**. 따라서, v1.1.x에서 v2.1로 업그레이드하는 경우:

1. 처음으로, [v2.0으로 업그레이드](../v2.0/upgrade-cockroach-version.html)하십시오. Be sure to complete all the steps, include the [finalization step](../v2.0/upgrade-cockroach-version.html#finalize-the-upgrade). 모든 단계를 완료하고, [완료 단계]를 포함하십시오. 

2. 그런 다음 이 페이지로 돌아가서 v2.1로 두 번째 롤링 업그레이드를 수행하십시오.

v2.0.x 또는 v2.1.x 패치 릴리스에서 업그레이드하는 경우, 중간 릴리스를 거칠 필요가 없습니다; 2단계를 계속하십시오.

## 2단계. 업그레이드 준비

업그레이드를 시작하기 전에, 다음 단계를 완료하십시오.

1. Make sure your cluster is behind a [load balancer](recommended-production-settings.html#load-balancing), or your clients are configured to talk to multiple nodes. If your application communicates with a single node, stopping that node to upgrade its CockroachDB binary will cause your application to fail.

2. Verify the overall health of your cluster using the [Admin UI](admin-ui-access-and-navigate.html). On the **Cluster Overview**:
    - Under **Node Status**, make sure all nodes that should be live are listed as such. If any nodes are unexpectedly listed as suspect or dead, identify why the nodes are offline and either restart them or [decommission](remove-nodes.html) them before beginning your upgrade. If there are dead and non-decommissioned nodes in your cluster, it will not be possible to finalize the upgrade (either automatically or manually).
    - Under **Replication Status**, make sure there are 0 under-replicated and unavailable ranges. Otherwise, performing a rolling upgrade increases the risk that ranges will lose a majority of their replicas and cause cluster unavailability. Therefore, it's important to identify and resolve the cause of range under-replication and/or unavailability before beginning your upgrade.
    - In the **Node List**:
        - Make sure all nodes are on the same version. If any nodes are behind, upgrade them to the cluster's current version first, and then start this process over.
        - Make sure capacity and memory usage are reasonable for each node. Nodes must be able to tolerate some increase in case the new version uses more resources for your workload. Also go to **Metrics > Dashboard: Hardware** and make sure CPU percent is reasonable across the cluster. If there's not enough headroom on any of these metrics, consider [adding nodes](start-a-node.html) to your cluster before beginning your upgrade.

3. Capture the cluster's current state by running the [`cockroach debug zip`](debug-zip.html) command against any node in the cluster. If the upgrade does not go according to plan, the captured details will help you and Cockroach Labs troubleshoot issues.

4. [클러스터 백업](backup-and-restore.html)하십시오. 업그레이드가 계획대로 진행되지 않으면, 데이터를 사용하여 클러스터를 이전 상태로 복원할 수 있습니다.

## 3단계. 업그레이드가 어떻게 완료될지 결정. 

{{site.data.alerts.callout_info}}
This step is relevant only when upgrading from v2.0.x to v2.1. For upgrades within the v2.1.x series, skip this step.
{{site.data.alerts.end}}

By default, after all nodes are running the new version, the upgrade process will be **auto-finalized**. This will enable certain performance improvements and bug fixes introduced in v2.1. After finalization, however, it will no longer be possible to perform a downgrade to v2.0. In the event of a catastrophic failure or corruption, the only option will be to start a new cluster using the old binary and then restore from one of the backups created prior to performing the upgrade.

We recommend disabling auto-finalization so you can monitor the stability and performance of the upgraded cluster before finalizing the upgrade, but note that you will need to follow all of the subsequent directions, including the manual finalization in step 5:

1. [Upgrade to v2.0](../v2.0/upgrade-cockroach-version.html), if you haven't already. The `cluster.preserve_downgrade_option` setting mentioned below is available only as of v2.0.3.

2. Start the [`cockroach sql`](use-the-built-in-sql-client.html) shell against any node in the cluster.

3. Set the `cluster.preserve_downgrade_option` [cluster setting](cluster-settings.html):

    {% include copy-clipboard.html %}
    ~~~ sql
    > SET CLUSTER SETTING cluster.preserve_downgrade_option = '2.0';
    ~~~

    It is only possible to set this setting to the current cluster version.

## 4단계. 롤링 업그레이드 수행

For each node in your cluster, complete the following steps.

{{site.data.alerts.callout_success}}
We recommend creating scripts to perform these steps instead of performing them manually.
{{site.data.alerts.end}}

{{site.data.alerts.callout_danger}}
Upgrade only one node at a time, and wait at least one minute after a node rejoins the cluster to upgrade the next node. Simultaneously upgrading more than one node increases the risk that ranges will lose a majority of their replicas and cause cluster unavailability.
{{site.data.alerts.end}}

1. 노드에 연결하십시오.

2. `cockroach` 프로세스를 중단하십시오.

    `systemd`와 같은 프로세스 관리자가 없다면, 다음 명령을 사용하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ pkill cockroach
    ~~~
    
    프로세스 관리자로 `systemd`를 사용하고 있다면, 이 명령을 사용하여 `systemd`를 재시작하지 않고 노드를 정지시키십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ systemctl stop <systemd config filename>
    ~~~

    그런 다음 프로세스가 중지되었는지 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ps aux | grep cockroach
    ~~~
    
    다른 방법으로, `server drained and shutdown completed` 메시지에 대한 노드 로그를 검사할 수 있습니다.

3. 사용할 CockroachDB 바이너리를 다운로드하여 설치하십시오:

    <div class="filters clearfix">
      <button style="width: 15%" class="filter-button" data-scope="mac">Mac</button>
      <button style="width: 15%" class="filter-button" data-scope="linux">Linux</button>
    </div>
    <p></p>

    <div class="filter-content" markdown="1" data-scope="mac">
    {% include copy-clipboard.html %}
    ~~~ shell
    # Get the CockroachDB tarball:
    $ curl -O https://binaries.cockroachdb.com/cockroach-{{page.release_info.version}}.darwin-10.9-amd64.tgz
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    # Extract the binary:
    $ tar xfz cockroach-{{page.release_info.version}}.darwin-10.9-amd64.tgz
    ~~~
    </div>

    <div class="filter-content" markdown="1" data-scope="linux">
    {% include copy-clipboard.html %}
    ~~~ shell
    # Get the CockroachDB tarball:
    $ wget https://binaries.cockroachdb.com/cockroach-{{page.release_info.version}}.linux-amd64.tgz
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    # Extract the binary:
    $ tar xfz cockroach-{{page.release_info.version}}.linux-amd64.tgz
    ~~~
    </div>

4. `$PATH`에서 `cockroach`를 사용하면, 오래된 `cockroach` 바이너리의 이름을 바꾼 다음, 새 파일을 그 자리로 옮깁니다:

    <div class="filters clearfix">
      <button style="width: 15%" class="filter-button" data-scope="mac">Mac</button>
      <button style="width: 15%" class="filter-button" data-scope="linux">Linux</button>
    </div>
    <p></p>

    <div class="filter-content" markdown="1" data-scope="mac">
    {% include copy-clipboard.html %}
    ~~~ shell
    $ i="$(which cockroach)"; mv "$i" "$i"_old
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cp -i cockroach-{{page.release_info.version}}.darwin-10.9-amd64/cockroach /usr/local/bin/cockroach
    ~~~
    </div>

    <div class="filter-content" markdown="1" data-scope="linux">
    {% include copy-clipboard.html %}
    ~~~ shell
    $ i="$(which cockroach)"; mv "$i" "$i"_old
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cp -i cockroach-{{page.release_info.version}}.linux-amd64/cockroach /usr/local/bin/cockroach
    ~~~
    </div>

5. 노드를 시작하여 클러스터에 다시 가입시키십시오.

    `systemd`와 같은 프로세스 관리자가 없다면, 처음에 노드를 시작하는 데 사용했던 [`cockroach start`](start-a-node.html) 명령을 다시 실행하십시오, 예를 들어:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --certs-dir=certs \
    --advertise-addr=<node address> \
    --join=<node1 address>,<node2 address>,<node3 address>
    ~~~
    
    프로세스 관리자로 `systemd`를 사용한다면, 이 명령을 실행하여 노드를 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ systemctl start <systemd config filename>
    ~~~

6. 노드가 출력을 통해 `stdout`으로 또는 [Admin UI](admin-ui-access-and-navigate.html)를 통해 클러스터에 다시 참여했는지 확인하십시오.

    {{site.data.alerts.callout_info}}
    To access the Admin UI for a secure cluster, [create a user with a password](create-user.html#create-a-user-with-a-password). Then open a browser and go to `https://<any node's external IP address>:8080`. On accessing the Admin UI, you will see a Login screen, where you will need to enter your username and password.
    {{site.data.alerts.end}}

7. `$PATH`에서 `cockroach`를 사용하면, 오래된 바이너리를 제거할 수 있습니다:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ rm /usr/local/bin/cockroach_old
    ~~~
    
    버전이 지정된 바이너리를 서버에 남겨두면, 아무것도 할 필요가 없습니다.

8. 노드가 클러스터에 다시 가입한 후 1 분 이상 기다린 후, 다음 노드에 대해 이 단계를 반복하십시오.

## 5단계. 업그레이드 완료

{{site.data.alerts.callout_info}}
이 단계는 v2.0.x에서 v2.1로 업그레이드할 때만 관련이 있습니다. v2.1.x 시리즈에서 업그레이드하려면, 이 단계를 건너 뜁니다.
{{site.data.alerts.end}}

If you disabled auto-finalization in step 3 above, monitor the stability and performance of your cluster for as long as you require to feel comfortable with the upgrade (generally at least a day). If during this time you decide to roll back the upgrade, repeat the rolling restart procedure with the old binary.

새 버전에 만족하면, 자동-완성을 다시-사용하십시오:

1. Start the [`cockroach sql`](use-the-built-in-sql-client.html) shell against any node in the cluster.
2. Re-enable auto-finalization:

    {% include copy-clipboard.html %}
    ~~~ sql
    > RESET CLUSTER SETTING cluster.preserve_downgrade_option;
    ~~~

## 6단계. 문제 해결

업그레이드가 완료되면 (수동 또는 자동으로), 이전 릴리스로 다운그레이드 할 수 없습니다. 문제가 발생하면, 다음과 같이 하는 것이 좋습니다:

1. 클러스터의 모든 노드에 대해 [`cockroach debug zip`] 명령을 실행하여 클러스터의 상태를 캡처하십시오.
2. [Reach out for support](support-resources.html) from Cockroach Labs, sharing your debug zip.

치명적인 오류나 손상이 발생할 경우, 이전 바이너리를 사용하여 새 클러스터를 시작한 다음, 업그레이드를 수행하기 전에 생성된 백업 중 하나에서 복원하는 것이 유일한 옵션입니다.

## 더 보기

- [노드 세부사항 보기](view-node-details.html)
- [디버그 정보 수집](debug-zip.html)
- [버전 세부사항 보기](view-version-details.html)
- [최신 버전의 출시 정보](../releases/{{page.release_info.version}}.html)
