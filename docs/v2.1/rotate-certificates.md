---
title: Rotate Security Certificates
summary: Rotate the security certificates of a secure CockroachDB cluster by creating and reloading new certificates.
toc: true
---

CockroachDB를 사용하면 노드를 다시 시작하지 않고도 보안 인증서를 교대할 수 있습니다.

{{site.data.alerts.callout_success}}보안 CockroachDB 클러스터에서 보안 인증서가 작동하는 방식에 대한 소개는 <a href="create-security-certificates.html">Create Security Certificates</a>를 참조하십시오.{{site.data.alerts.end}}


## When to rotate certificates

You may need to rotate the node, client, or CA certificates in the following scenarios:

- The node, client, or CA certificates are expiring soon.
- Your organization's compliance policy requires periodical certificate rotation.
- The key (for a node, client, or CA) is compromised.
- You need to modify the contents of a certificate, for example, to add another DNS name or the IP address of a load balancer through which a node can be reached. In this case, you would need to rotate only the node certificates.

## 인증서를 교대하는 상황

노드, 클라이언트 또는 CA 인증서를 교대가 필요할 수 있는 상황:

- 노드, 클라이언트 또는 CA 인증서가 곧 만료됨.
- 조직의 규정 준수 정책에 따라 정기적인 인증서 순환이 필요함.
- (노드, 클라이언트 또는 CA의 경우) 키가 손상됨
- 인증서의 내용을 수정할 때. 예를 들면, 노드에 도달할 수 있는 다른 DNS 이름이나 로드 밸런서의 IP 주소를 추가할 때(이 경우 노드 인증서만 교대하면 된다)

### 클라이언트 인증서 교대

1. 새로운 인증서와 그에 해당하는 키를 생성합니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-client <username>
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

2. 원하는 메소드를 사용하여 새로운 클라이언트와 해당 키를 클라이언트에 업로드 합니다.

3. 클라이언트가 새 클라이언트 인증서를 사용하도록 하십시오.

    이 단계는 애플리케이션별로 다르며 클라이언트를 재시작해야 할 수도 있습니다.

### 노드 인증서 교대

노드 인증서를 교대하기 위해, 새 노드 인증서와 해당 키를 생성하여 노드에 다시 로드합니다.

1. 새 노드 인증서와 그에 해당하는 키를 생성합니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-node \
    <node hostname> \
    <node other hostname> \
    <node yet another hostname> \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key \
    --overwrite
    ~~~

    기존 인증서 및 키와 동일한 디렉터리에 새 인증서와 키를 생성해야 하므로  `--overwrite`플래그를 사용하여 기존 파일을 덮어쓰십시오. 또한 노드에 도달할 수 있는 모든 주소를 지정하십시오.

2. 노드 인증서 및 키를 노드에 업로드하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ scp certs/node.crt \
    certs/node.key \
    <username>@<node address>:~/certs
    ~~~

3. `cockroach` 프로세스에 `SIGHUP` 신호를 전송하여 노드를 다시 시작하지 않고 노드 인증서를 다시 로드하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    pkill -SIGHUP -x cockroach
    ~~~

    `SIGHUP` 신호는 프로세스를 실행하는 동일한 사용자가 전송해야 합니다(예: `cockroach` 프로세스가 사용자 `root`로 실행 중인 경우 `sudo`로 실행).

4. 관리 UI의 **Local Node Certificates** 페이지를 사용하여 인증서 회전에 성공했는지 확인하십시오: `https://<address of node with new certs>:8080/#/reports/certificates/local`.

    노드 인증서 세부 정보로 스크롤하고 **Valid Until** 필드에 새 인증서 만료 시간이 표시되는지 확인하십시오.

### CA 인증서 교대

CA 인증서를 교대하려면 새 CA 인증서와 이전 CA 인증서가 포함된 결합 CA 인증서를 생성한 다음 노드와 클라이언트에서 새 결합된 CA 인증서를 다시 로드하십시오. 모든 노드와 클라이언트가 결합된 CA 인증서를 가지고 나면 새 CA 인증서로 서명된 새 노드와 클라이언트 인증서를 생성하고 노드와 클라이언트에서도 해당 인증서를 다시 로드하십시오.

자세한 내용은 [CockroachDB가 결합 CA 인증서를 생성하는 이유](rotate-certificates.html#why-cockroachdb-creates-a-combined-ca-certificate) 와 [CA 인증서를 사전에 교대하는 이유](rotate-certificates.html#why-to-rotate-ca-certificates-in-advance)를 참조하십시오.

1. 기존 CA 키 이름을 바꾸십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ mv  my-safe-directory/ca.key my-safe-directory/ca.old.key
    ~~~

2. `--overwrite` 플래그를 사용하여 새 CA 인증서와 키를 만들어 기존 CA 인증서를 덮어쓰십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-ca \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key \
    --overwrite
    ~~~

    이로 인해 [결합 CA 인증서](rotate-certificates.html#why-cockroachdb-creates-a-combined-ca-certificate)가 생성됩니다. 새 인증서와 이전 인증서를 포함하는 `ca.crt`.

    {{site.data.alerts.callout_danger}}CA 키는 <code> cockroach </code> 명령으로 자동 로드되지 않으므로, <code>--ca-key</code> 플래그로 식별되는 별도의 디렉토리에 생성되어야 합니다.{{site.data.alerts.end}}

2.각 노드에 새 CA 인증서를 업로드합니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ scp certs/ca.crt
    <username>@<node1 address>:~/certs
    ~~~

3. 원하는 방법을 사용하여 새 CA 인증서를 각 클라이언트에 업로드하십시오.

4. 각 노드에서 `cockroach` 프로세스에 `SIGHUP` 신호를 전송하여 노드를 다시 시작하지 않고 CA 인증서를 다시 로드하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    pkill -SIGHUP -x cockroach
    ~~~

    `SIGHUP` 신호는 프로세스를 실행하는 동일한 사용자가 전송해야 합니다(예: `cockroach` 프로세스가 사용자 `root`로 실행 중인 경우 `sudo`로 실행).

5.  각 클라이언트에 CA 인증서를 다시 로드하십시오.

     이 단계는 애플리케이션별로 다르며 클라이언트를 재시작해야 할 수도 있습니다.

6. 관리 UI의 **Local Node Certificates** 페이지에서 인증서 교대가 성공했는지 확인하십시오: `https://<address of node with new certs>:8080/#/reports/certificates/local`.

    기존 CA 인증서와 새 CA 인증서의 상세 내역이 표시되어야 합니다. 새 CA 인증서의 **Valid  Until** 필드에 새 인증서 만료 시간이 표시되는지 확인하십시오.

7. 모든 노드와 클라이언트가 새 CA 인증서를 가지고 있다고 확신하면, [노드 인증서 교대](#rotate-node-certificates) 및 [클라이언트 인증서 교대](#rotate-client-certificates)를 수행합니다.

## CockroachDB가 결합된 CA 인증서를 생성하는 이유

CA 인증서를 교대할 때, 노드는 certs 디렉토리를 다시 스캔된 후의 새 CA 인증서를 갖게 되며 클라이언트는 클라이언트가 재시작했던 때의 새 CA 인증서를 갖게 됩니다. 그러나 노드와 클라이언트 인증서가 새로운 인증서로 교대될 때까지 노드와 클라이언트 인증서는 여전히 이전 CA 인증서로 서명됩니다. 그러므로 노드와 클라이언트는 새로운 CA 인증서를 사용하여 서로의 신원을 확인할 수 없습니다.

우리는 이 문제를 해결하기 위해 여러 개의 CA 인증서가 동시에 활성화될 수 있다는 사실을 이용합니다. 다른 노드나 클라이언트의 신원을 확인하는 동안, 그들은 그들에게 업로드된 여러 개의 CA 인증서로 확인할 수 있다. 따라서 CockroachDB는 CA 인증서를 교대할 때 새 인증서만 생성하는 대신, 새 CA 인증서와 이전 CA 인증서를 포함하는 결합된 CA 인증서를 생성합니다. 노드와 클라이언트 인증서가 교대될 때와 마찬가지로, 결합 CA 인증서가 새 노드 및 클라이언트 인증서뿐 아니라 이전 노드 및 클라이언트 인증서를 확인할 수 있습니다.

## CA 인증서를 사전에 교대하는 이유

CA 인증서를 교대한 후 노드와 클라이언트 인증서를 회전할 때 노드와 클라이언트 인증서는 새 CA 인증서를 사용하여 서명됩니다. 노드는 노드의 인증서 디렉토리가 다시 스캔되는 즉시 새 노드와 CA 인증서를 사용한다. 그러나 클라이언트는 클라이언트를 재시작할 때만 새로운 CA 인증서 및 클라이언트 인증서를 사용합니다. 따라서 새로운 CA 인증서가 아직 없는 클라이언트에서 새로운 CA 인증서가 서명한 노드 인증서는 수락되지 않습니다. 모든 노드와 클라이언트에 최신 CA 인증서가 확실히 있도록 하려면, CA 인증서는 노드나 클라이언트 인증서와는 완전히 다른 스케줄(이상적으로 노드와 클라이언트 인증서를 변경하기 몇 개월 전)로 교대하십시오.

## 더 알아보기

- [보안 인증서 만들기](create-security-certificates.html)
- [수동 배포](manual-deployment.html)
- [조정된 배포](orchestration.html)
- [배포 테스트](deploy-a-test-cluster.html)
- [로컬 배포](secure-a-cluster.html)
- [ Cockroach 명령어](cockroach-commands.html)
