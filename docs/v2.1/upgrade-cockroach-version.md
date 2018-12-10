---
title: Upgrade to CockroachDB v2.1
summary: Learn how to upgrade your CockroachDB cluster to a new version.
toc: true
toc_not_nested: true
---

CockroachDB의 [다중-활성 가용성](multi-active-availability.html) 설계로 인해 CockroachDB 클러스터의 "롤링 업그레이드"를 수행할 수 있습니다. 
즉, 클러스터의 전반적인 상태 및 작업을 방해하지 않고 한 번에 하나씩 노드를 업그레이드 할 수 있습니다.

{{site.data.alerts.callout_info}}
이 페이지에서는 v2.0.x 또는 v2.1.x 시리즈의 모든 패치 릴리스에서 최신 v2.1 릴리스 ({{page.release_info.version}})로 업그레이드하는 방법을 보여줍니다. v2.0.x 시리즈에서 업그레이드하려면, [이 페이지의 v2.0 버전](https://www.cockroachlabs.com/docs/v2.0/upgrade-cockroach-version.html)을 참조하십시오.
{{site.data.alerts.end}}

## 1단계. 업그레이드 할 수 있는지 확인

업그레이드 할 때, 패치 릴리스를 건너 뛸 수는 있지만, **전체 릴리스를 건너 뛸 수는 없습니다**. 따라서, v1.1.x에서 v2.1로 업그레이드하는 경우:

1. 처음으로, [v2.0으로 업그레이드](../v2.0/upgrade-cockroach-version.html)하십시오. [완료 단계](../v2.0/upgrade-cockroach-version.html#finalize-the-upgrade)를 포함하여 모든 단계를 완료하십시오. 

2. 그런 다음 이 페이지로 돌아가서 v2.1로 두 번째 롤링 업그레이드를 수행하십시오.

v2.0.x 또는 v2.1.x 패치 릴리스에서 업그레이드하는 경우, 중간 릴리스를 거칠 필요가 없습니다; 2단계를 계속하십시오.

## 2단계. 업그레이드 준비

업그레이드를 시작하기 전에, 다음 단계를 완료하십시오.

1. 클러스터가 [로드 밸런서](recommended-production-settings.html#load-balancing) 뒤에 있는지 확인하거나, 클라이언트가 여러 노드와 통신하도록 구성하십시오. 어플리케이션이 단일 노드와 통신하는 경우, 해당 노드를 중지하여 CockroachDB 바이너리를 업그레이드하면 어플리케이션이 실패하게됩니다.

2. [Admin UI](admin-ui-access-and-navigate.html)를 사용하여 클러스터의 전반적인 상태를 확인하십시오. **클러스터 개요**에서:
    - **노드 상태**에서, 활성상태여야 하는 모든 노드가 표시되어야 합니다. 노드가 예기치 않게 의심 또는 비활성 상태로 나열되는 경우, 노드가 오프라인 상태인 이유를 확인하고 업그레이드를 시작하기 전에 노드를 다시 시작하거나 [해제](remove-nodes.html)하십시오. 클러스터에 사용 불능 노드와 폐기되지 않은 노드가 있는 경우, 자동 또는 수동으로 업그레이드를 완료 할 수 없습니다.
    - **복제 상태**에서, 복제-부족 및 사용 불가능 범위가 0인지 확인하십시오. 그렇지 않으면, 롤링 업그레이드를 수행하면 범위에서 복제본의 대부분이 손실되고 클러스터를 사용할 수 없게 될 위험이 커집니다. 따라서, 업그레이드를 시작하기 전에 복제 그리고/또는 사용할 수 없는 범위의 원인을 확인하고 해결하는 것이 중요합니다.
    - **노드 리스트**에서:
        - 모든 노드가 동일한 버전인지 확인하십시오. 노드가 뒤에 있는 경우, 클러스터의 현재 버전으로 먼저 업그레이드 한 다음, 이 프로세스를 다시 시작하십시오.
        - 용량과 메모리 사용이 각 노드에 대해 합리적인지 확인하십시오. 새 버전이 워크로드에 더 많은 리소스를 사용하는 경우 노드는 약간의 증가를 허용할 수 있어야 합니다. 또한 **Metrics > Dashboard: Hardware**로 이동하여 클러스터에서 CPU 비율이 적절한지 확인하십시오. 이러한 메트릭 중 여유 공간이 충분하지 않은 경우, 업그레이드를 시작하기 전에 클러스터에 [노드 추가](start-a-node.html)를 고려하십시오.

3. 클러스터의 노드에 대해 [`cockroach debug zip`](debug-zip.html) 명령을 실행하여 클러스터의 현재 상태를 캡처하십시오. 업그레이드가 계획대로 진행되지 않으면, 캡쳐된 세부 정보가 도움이 되며 Cockroach Labs가 문제를 해결하는 데 도움이 됩니다.

4. [클러스터 백업](backup-and-restore.html)하십시오. 업그레이드가 계획대로 진행되지 않으면, 데이터를 사용하여 클러스터를 이전 상태로 복원할 수 있습니다.

## 3단계. 업그레이드가 어떻게 완료될지 결정 

{{site.data.alerts.callout_info}}
이 단계는 v2.0.x에서 v2.1로 업그레이드할 때만 관련이 있습니다. v2.1.x 시리즈에서 업그레이드하려면, 이 단계를 건너 뜁니다.
{{site.data.alerts.end}}

기본적으로, 모든 노드에서 새 버전을 실행하면, 업그레이드 프로세스가 **자동-종료**됩니다. 이를 통해 v2.1에서 소개된 특정 성능 향상 및 버그 수정이 가능합니다. 그러나, 종료 후, 더 이상 v2.0으로 다운 그레이드 할 수 없습니다. 치명적인 오류나 손상이 발생할 경우, 이전 바이너리를 사용하여 새 클러스터를 시작한 다음, 업그레이드를 수행하기 전에 생성 된 백업 중 하나에서 복원하는 것이 유일한 옵션입니다.

업그레이드를 완료하기 전에 업그레이드된 클러스터의 안정성과 성능을 모니터링 할 수 있도록 자동-종료를 비활성화하는 것이 좋지만, 5단계의 수동 완료를 포함하여 이후의 모든 지침을 따라야 합니다:

1. 아직 업그레이드하지 않았다면, [v2.0으로 업그레이드](../v2.0/upgrade-cockroach-version.html)하십시오. 아래에 언급된 `cluster.preserve_downgrade_option` 설정은 v2.0.3부터 사용 가능합니다.

2. 클러스터의 노드에 대해 [`cockroach sql`](use-the-built-in-sql-client.html) 쉘을 시작하십시오.

3. `cluster.preserve_downgrade_option` [클러스터 세팅](cluster-settings.html)을 설정하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SET CLUSTER SETTING cluster.preserve_downgrade_option = '2.0';
    ~~~
    
    이 설정은 현재 클러스터 버전으로만 설정할 수 있습니다.

## 4단계. 롤링 업그레이드 수행

클러스터의 각 노드에 대해, 다음 단계를 완료하십시오.

{{site.data.alerts.callout_success}}
이러한 단계를 수동으로 수행하는 대신 수행하는 스크립트를 만드는 것이 좋습니다.
{{site.data.alerts.end}}

{{site.data.alerts.callout_danger}}
한 번에 하나의 노드만 업그레이드하고, 노드가 다음 노드를 업그레이드하기 위해 클러스터에 다시 참여한 후 최소 1분을 기다리십시오. 둘 이상의 노드를 동시에 업그레이드하면, 범위에서 복제본의 대부분이 손실되어 클러스터를 사용할 수 없게 될 위험이 커집니다.
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
    보안 클러스터의 Admin UI에 접근하려면, [비밀번호가 있는 사용자를 생성](create-user.html#create-a-user-with-a-password)하십시오. 그런 다음 브라우저를 열고 `https://<any node's external IP address>:8080`으로 가십시오. Admin UI에 접근하면, 사용자 이름과 비밀번호를 입력해야 하는 로그인 화면이 표시됩니다.
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

위의 3단계에서 자동-완성 기능을 사용하지 않도록 설정한 경우, 업그레이드에 익숙해질 필요가 있는 한 (일반적으로 하루 이상) 클러스터의 안정성 및 성능을 모니터링하십시오. 이 시간 동안 업그레이드를 롤백하기로 결정한 경우, 이전 바이너리로 롤링 재시작 절차를 반복하십시오.

새 버전에 만족하면, 자동-완성을 다시-사용하십시오:

1. 클러스터의 노드에 대해 [`cockroach sql`](use-the-built-in-sql-client.html) 쉘을 시작하십시오.
2. 자동-종료 기능 다시-사용:

    {% include copy-clipboard.html %}
    ~~~ sql
    > RESET CLUSTER SETTING cluster.preserve_downgrade_option;
    ~~~

## 6단계. 문제 해결

업그레이드가 완료되면 (수동 또는 자동으로), 이전 릴리스로 다운그레이드 할 수 없습니다. 문제가 발생하면, 다음과 같이 하는 것이 좋습니다:

1. 클러스터의 모든 노드에 대해 [`cockroach debug zip`] 명령을 실행하여 클러스터의 상태를 캡처하십시오.
2. [Cockroach Labs의 지원](support-resources.html)을 받아, 디버그 zip을 공유하십시오.

치명적인 오류나 손상이 발생할 경우, 이전 바이너리를 사용하여 새 클러스터를 시작한 다음, 업그레이드를 수행하기 전에 생성된 백업 중 하나에서 복원하는 것이 유일한 옵션입니다.

## 더 보기

- [노드 세부사항 보기](view-node-details.html)
- [디버그 정보 수집](debug-zip.html)
- [버전 세부사항 보기](view-version-details.html)
- [최신 버전의 출시 정보](../releases/{{page.release_info.version}}.html)
