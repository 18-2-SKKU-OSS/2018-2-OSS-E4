---
title: Orchestrate CockroachDB with Docker Swarm
summary: How to orchestrate the deployment and management of a secure three-node CockroachDB cluster as a Docker swarm.
toc: true
---

<div class="filters filters-big clearfix">
  <button class="filter-button current"><strong>Secure</strong></button>
  <a href="orchestrate-cockroachdb-with-docker-swarm-insecure.html"><button class="filter-button">Insecure</button></a>
</div>

이 페이지에서는 안전하지 않은 세 노드 CockroachDB 클러스터의 배포, 관리를 [도커 엔진 스웜](https://docs.docker.com/engine/swarm/)로 바꾸는 것을 보여줍니다.

만약 오직 CockroachDB을 테스트하고만 싶은 경우, 또는 TLS 암호화를 통한 네트워크 통신 보호와 관련이 없는 경우, 대신 보안 클러스터를 사용하는 것을 추천합니다. 설명서를 보러면 위의 **Secure** 을 선택하십시오.


## 시작하기 전에

시작하기 전에, 용어를 복습하는 것은 도움이 됩니다:

특징 | 묘사
--------|------------
인스턴스 | 실제 또는 가상의 머신. 이 튜토리얼에서는, CockroachDB노드 당 3개씩을 사용하십시오.
[도커 엔진](https://docs.docker.com/engine/) | 이것은 컨테이너를 생성하고 실행하는 핵심 Docker 어플리케이션입니다. 이 튜토리얼에서는, 세 개의 예시 각각에 도커 엔진을 설치하고 시작하십시오.
[스웜](https://docs.docker.com/engine/swarm/key-concepts/#/swarm) | 스웜은 단일, 가상의 호스트로 된 도커 엔진 그룹입니다.
[스웜 노드](https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/) | 스웜의 각 멤버는 하나의 노드로 구성됩니다. 이 튜토리얼에서 각 인스턴스는 스웜 노드가 되며, 하나는 마스터 노드, 다른 두 개는 워커 노드 역할을 한다. 작업노드로 작업을 전달하는 마스터 노드에 서비스를 제출하십시오.
[서비스](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/) | 서비스는 스웜 노드에서 실행할 테스크의 정의입니다. 이 튜토리얼에서, 컨테이너 안에서 CockrochDB 노드를 시작하고 단일 클러스터에 결합하는 각각의 세 가지 서비스를 정의하십시오. 각 서비스는 확인 가능한 DNS 이름을 통해 재시작 시 안정적인 네트워크 ID를 보장합니다.
[시크릿](https://docs.docker.com/engine/swarm/secrets/) | 시크릿은 컨테이너가 런타임에 필요로 하는 민감한 데이터를 관리하기 위한 도커의 메커니즘입니다다. CockroachDB는 TLS 인증서를 사용하여 노드 간 및 클라이언트/노드 간 통신을 인증하고 암호화하므로, 인증서별로 암호를 생성하고 서비스에 암호를 사용하십시오.
[오버레이 네트워크](https://docs.docker.com/engine/userguide/networking/#/an-overlay-network-with-docker-engine-swarm-mode) | 오버레이 네트워크는 스웜 노드들 간의 통신을 가능하게 합니다. 이 튜토리얼에서, 오버레이 네트워크를 만들어 각 서비스에서 사용하십시오. 

## 단계 1. 인스턴스 생성

클러스터의 각 노드에 하나씩, 3개의 인스턴스를 생성하십시오.

- GCE별 지침, [GCE에 CockroachDB 배포](deploy-cockroachdb-on-google-cloud-platform-insecure.html)하는 부분의 2 단계 부분을 읽으십시오.
- AWS별 지침, [AWS에 CockroachDB 배포](deploy-cockroachdb-on-aws-insecure.html)하는 부분의 2 단계 부분을 읽으십시오.

이러한 포트에서 TCP 통신을 허용하도록 네트워크를 구성하십시오:

- 내부적 노드 의사소통(i.e., 클러스터로 일하는)과 어플리케이션과의 연결을 위한`26257`
- 관리자 UI 노출을 위한 `8080`

## 단계 2. 도커 엔진 설치

각각의 인스턴스에서:

1. [도커 엔진 설치하고 시작](https://docs.docker.com/engine/installation/).

2. 도커 데몬이 백그라운드에서 실행 중인지 확인:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker version
    ~~~

## 단계 3. 스웜 시작

1. 관리자 노드를 실행할 인스턴스에, [스웜을 설치](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/) 하십시오.

   다음 단계에서 사용할 명령이 포함되어 있으므로 `docker swarm init` 의 출력에 유의하십시오. 다음과 같이 보여야 합니다:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker swarm init --advertise-addr 10.142.0.2
    ~~~

    ~~~
    Swarm initialized: current node (414z67gr5cgfalm4uriu4qdtm) is now a manager
    To add a worker to this swarm, run the following command
       $ docker swarm join \
       --toke    SWMTKN-1-5vwxyi6zl3cc62lqlhi1jrweyspi8wblh2i3qa7kv277fgy74n-e5eg5c7ioxypjxlt3rpqorh15 \
       10.142.0.2:237
    To add a manager to this swarm, run 'docker swarm join-token manager' and follow th    instructions.
    ~~~

2. 다른 두 개의 인스턴스에서, 단계 1의 출력 명령인 `도커 스웜 합치기`를 실행함으로써 [스웜에 합쳐지는 워커 노드를 만드십시오](https://docs.docker.com/engine/swarm/swarm-tutorial/add-nodes/):

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker swarm join \
          --to    SWMTKN-1-5vwxyi6zl3cc62lqlhi1jrweyspi8wblh2i3qa7kv277fgy74n-e5eg5c7ioxypjxlt3rpqorh15 \
          10.142.0.2:2377
    ~~~

    ~~~
    This node joined a swarm as a worker.
    ~~~

3. 관리자 노드를 실행하는 인스턴스에서, 스웜이 실행 중인지 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker node ls
    ~~~

    ~~~
    ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
    414z67gr5cgfalm4uriu4qdtm *  instance-1  Ready   Active        Leader
    ae144s35dx1p1lcegh6bblyed    instance-2  Ready   Active
    aivjg2joxyvzvbksjsix27khy    instance-3  Ready   Active
    ~~~

## 단계 4. 오버레이 네트워크 생성

관리자 노드를 실행하는 인스턴스에서 스웜의 컨테이너가 서로 통신할 수 있도록 오버레이 네트워크를 생성하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ sudo docker network create --driver overlay --attachable cockroachdb
~~~

+`--attachable` 옵션은 도커에서 실행되는 스웜이 아닌 컨테이너가 네트워크상의 서비스에 접근할 수 있게 해 서비스를 상호작용적으로 사용하기 쉽게 해줍니다.

## 단계 5. 보안 리소스 생성

A secure CockroachDB cluster uses TLS certificates for encrypted inter-node and client/node authentication and communication. In this step, you'll install CockroachDB on the instance running your manager node, use the [`cockroach cert`](create-security-certificates.html) command to generate certificate authority (CA), node, and client certificate and key pairs, and use the [`docker secret create`](https://docs.docker.com/engine/reference/commandline/secret_create/) command to assign these files to Docker [secrets](https://docs.docker.com/engine/swarm/secrets/) for use by your Docker services.

1. 관리자 노드를 실행하는 인스턴스에서 최신 이진 파일을 사용하여 CockroachDB를 설치하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    # Get the latest CockroachDB tarball:
    $ wget https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    # Extract the binary:
    $ tar -xf cockroach-{{ page.release_info.version }}.linux-amd64.tgz  \
    --strip=1 cockroach-{{ page.release_info.version }}.linux-amd64/cockroach
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    # Move the binary:
    $ sudo mv cockroach /usr/local/bin
    ~~~

2. `certs` 디렉토리와 안전 디렉토리를 생성하여 CA 키를 보관하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ mkdir certs
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ mkdir my-safe-directory
    ~~~

3. CA 인증서 및 키 생성:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-ca \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ls certs
    ~~~

    ~~~
    ca.crt
    ~~~

4. [`docker secret create`](https://docs.docker.com/engine/reference/commandline/secret_create/) 명령을 사용하여 `ca.crt`를 위한 도커 시크릿을 생성합니다:

    {{site.data.alerts.callout_danger}} <code>ca.key</code> 파일을 어딘가에 안전하게 저장해두고 백업해둡니다; 손실된 경우 새 노드나 클라이언트를 클러스터에 추가할 수 없습니다.{{site.data.alerts.end}}

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker secret create ca-crt certs/ca.crt
    ~~~

    이 명령은 시크릿 (`ca-crt`) 에 이름을 할당하고 cockroach가 생성한 CA 인증서 파일의 위치를 식별합니다. 원하는 경우 다른 비밀 이름을 사용할 수 있지만 다음 단계에서 CockroachDB 노드를 시작할 때는 올바른 이름을 참조해야 합니다.

5. 첫 번째 노드의 인증서 및 키 생성:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-node \
    cockroachdb-1 \
    localhost \
    127.0.0.1 \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ls certs
    ~~~

    ~~~
    ca.crt
    node.crt
    node.key
    ~~~

    이 명령은 나중에 노드에 사용할 서비스 이름 (`cockroachdb-1`) 과 노드와 동일한 컨테이너에서 내장 SQL 셸 및 기타 CockroachDB 클라이언트 명령을 더 쉽게 실행할 수 있는 인증서/키 쌍을 로컬 주소로 발급합니다.

6. 첫 번째 노드의 인증서 및 키에 대한 Docker 암호 생성:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker secret create cockroachdb-1-crt certs/node.crt
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker secret create cockroachdb-1-key certs/node.key
    ~~~

    다시, 이러한 명령은 시크릿 (`cockroachdb-1-crt` 과 `cockroachdb-1-key`) 에 이름을 할당하고 바퀴벌레가 생성한 인증서와 키 파일의 위치를 식별합니다.
    
7.  `--overwrite` 플래그를 사용하여 첫 번째 노드에 대하여 생성된 파일을 대체하는 두 번째 노드에 대한 인증서와 키를 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-node --overwrite \
    cockroachdb-2 \
    localhost \
    127.0.0.1 \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ls certs
    ~~~

    ~~~
    ca.crt
    node.crt
    node.key
    ~~~

8. 두 번째 노드의 인증서 및 키에 대한 도커 암호 생성:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker secret create cockroachdb-2-crt certs/node.crt
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker secret create cockroachdb-2-key certs/node.key
    ~~~

9. `--overwrite` 플래그를 사용하여 두 번째 노드에 대해 생성된 파일을 다시 사용하여 세 번째 노드에 대한 인증서와 키를 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-node --overwrite \
    cockroachdb-3 \
    localhost \
    127.0.0.1 \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ls certs
    ~~~

    ~~~
    ca.crt
    node.crt
    node.key
    ~~~

10. 세 번째 노드의 인증서 및 키에 대한 Docker 암호 생성:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker secret create cockroachdb-3-crt certs/node.crt
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker secret create cockroachdb-3-key certs/node.key
    ~~~

11. `root` 사용자에 대한 클라리언트 인증서 및 키 생성:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-client \
    root \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

12. `root` 사용자의 인증서 및 키에 대한 도커 암호를 생성:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker secret create cockroachdb-root-crt certs/client.root.crt
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker secret create cockroachdb-root-key certs/client.root.key
    ~~~

## 단계 6. CockroachDB 클러스터 시작

1. 관리자 노드를 실행하는 인스턴스에서 각 CockroachDB 노드에 대해 하나의 스웜 서비스 생성합니다:

    {% include copy-clipboard.html %}
    ~~~
    # Create the first service:
    $ sudo docker service create \
    --replicas 1 \
    --name cockroachdb-1 \
    --hostname cockroachdb-1 \
    --network cockroachdb \
    --mount type=volume,source=cockroachdb-1,target=/cockroach/cockroach-data,volume-driver=local \
    --stop-grace-period 60s \
    --publish 8080:8080 \
    --secret source=ca-crt,target=ca.crt \
    --secret source=cockroachdb-1-crt,target=node.crt \
    --secret source=cockroachdb-1-key,target=node.key,mode=0600 \
    --secret source=cockroachdb-root-crt,target=client.root.crt \
    --secret source=cockroachdb-root-key,target=client.root.key,mode=0600 \
    cockroachdb/cockroach:{{page.release_info.version}} start \
    --join=cockroachdb-1:26257,cockroachdb-2:26257,cockroachdb-3:26257 \
    --cache=.25 \
    --max-sql-memory=.25 \
    --logtostderr \
    --certs-dir=/run/secrets
    ~~~

    {% include copy-clipboard.html %}
    ~~~
    # Create the second service:
    $ sudo docker service create \
    --replicas 1 \
    --name cockroachdb-2 \
    --hostname cockroachdb-2 \
    --network cockroachdb \
    --stop-grace-period 60s \
    --mount type=volume,source=cockroachdb-2,target=/cockroach/cockroach-data,volume-driver=local \
    --secret source=ca-crt,target=ca.crt \
    --secret source=cockroachdb-2-crt,target=node.crt \
    --secret source=cockroachdb-2-key,target=node.key,mode=0600 \
    --secret source=cockroachdb-root-crt,target=client.root.crt \
    --secret source=cockroachdb-root-key,target=client.root.key,mode=0600 \
    cockroachdb/cockroach:{{page.release_info.version}} start \
    --join=cockroachdb-1:26257,cockroachdb-2:26257,cockroachdb-3:26257 \
    --cache=.25 \
    --max-sql-memory=.25 \
    --logtostderr \
    --certs-dir=/run/secrets
    ~~~

    {% include copy-clipboard.html %}
    ~~~
    # Create the third service:
    $ sudo docker service create \
    --replicas 1 \
    --name cockroachdb-3 \
    --hostname cockroachdb-3 \
    --network cockroachdb \
    --mount type=volume,source=cockroachdb-3,target=/cockroach/cockroach-data,volume-driver=local \
    --stop-grace-period 60s \
    --secret source=ca-crt,target=ca.crt \
    --secret source=cockroachdb-3-crt,target=node.crt \
    --secret source=cockroachdb-3-key,target=node.key,mode=0600 \
    --secret source=cockroachdb-root-crt,target=client.root.crt \
    --secret source=cockroachdb-root-key,target=client.root.key,mode=0600 \
    cockroachdb/cockroach:{{page.release_info.version}} start \
    --join=cockroachdb-1:26257,cockroachdb-2:26257,cockroachdb-3:26257 \
    --cache=.25 \
    --max-sql-memory=.25 \
    --logtostderr \
    --certs-dir=/run/secrets
    ~~~

    이러한 명령은 각각 컨테이너를 시작하고 오버레이 네트워크에 결합하는 서비스를 생성하고, 영구 저장을 위해 로컬 볼륨에 마운트된 컨테이너 내부에 있는 CockroachDB 노드를 시작합니다. 각각의 파트를 살펴보면:
    - `sudo docker service create`: 도커명령은 새로운 서비스를 만듭니다.
    - `--replicas`: 서비스가 제어하는 컨테이너의 수. 각 서비스는 CockroachDB 노드 1개를 실행하는 컨테이너 1개를 제어하기 때문에, 이것은 항상 `1`이 될 것입니다.
    - `--name`: 서비스 이름.
    - `--hostname`: 컨테이너의 호스트 이름. 이 주소에서 연결에 관련됩니다.
    - `--network`: 컨테이너가 결합할 전체적인 네트워크 입니다. 더 자세한 설명을 위해 [단계 4. 오버레이 네트워크 만들기](#step-4-create-an-overlay-network) 를 보십시오.
    - `--mount`: 이 플래그는 서비스와 동일한 이름의 로컬 볼륨을 탑재합니다. 즉, 이 컨테이너에서 실행 중인 노드의 데이터 및 로그는 인스턴스의 `/cockroach/cockroach-data` 에 저장되며, 보증되지 않았지만 동일한 인스턴스에서 재시작 시 다시 사용할 수 있습니다.
     {{site.data.alerts.callout_info}} 인스턴스를 교체하거나 추가할 계획이라면, 로컬 디스크 대신 원격 스토리지를 사용하는 것이 좋습니다. 이를 위해, 선택한 볼륨 드라이버를 사용하여 각 CockroachDB 인스턴스에 대해 <a href="https://docs.docker.com/engine/reference/commandline/volume_create/">원격 볼륨을 생성</a> 하고, 만약 <a href="https://github.com/mcuadros/gce-docker">GCE 볼륨 드라이버</a>를 사용한다면, <code>volume-driver=local</code> 대신에 볼륨 드라이버를 분류합니다. e.g., <code>volume-driver=gce</code>
    - `--stop-grace-period`: 이 플래그는 CockroachDB가 적절하게 폐쇄할 수 있는 충분한 시간을 줄 수 있는 유예기간을 설정합니다.
    - `--publish`: 이 플래그는 관리 UI가 스웜 노드가 `8080`에서 실행되고 있는 어떤 인스턴스의 IP에 접근할 수 있게 합니다. 이 플래그는 첫 번째 노드의 서비스에서만 정의되지만, 스웜은 라우팅 메쉬를 사용하여 모든 스웜 노드에 이 포트를 노출한다는 점에 유의하십시오. 자세한 설명을 위해 [포트 게시](https://docs.docker.com/engine/swarm/services/#publish-ports) 를 참고하십시오.
    - `--secret`: 이 깃발들은 노드 보안에 사용할 비밀들을 식별합니다. 그것들은 5단계에서 정의한 비밀 이름을 참조해야 합니다. 노드와 클라이언트 인증서, 핵심 기밀에 대해서는 `source` 필드가 관련 기밀을 식별하며, `target` 필드는 `cockroach start` 와 `cockroach sql` 플래그에 사용할 이름을 정의합니다. 노드와 클라이언트의 키 기밀에 대해서도 `mode` 필드는 파일 권한을 `0600` 으로 설정하고, 이 권한이 설정되어 있지 않으면 Docker는 CockroachDB의 기본 SQL 클라이언트에서는 작동하지 않는 `0444` 의 기본 파일 권한을 할당합니다.
    - `cockroachdb/cockroach:{{page.release_info.version}} start ...`: CockroachDB는 안전하지 않은 모드의 컨테이너에서 [노드 시작](start-a-node.html)을 명령하고, 다른 클러스터 구성원에게 서비스의 이름과 일치하는 영구 네트워크 주소를 사용하여 서로 커뮤니케이션하도록 지시 합니다.

2. 세가지 서비스가 모두 생성되었는지 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker service ls
    ~~~

    ~~~
    ID            NAME           MODE        REPLICAS  IMAGE
    a6g0ur6857j6  cockroachdb-1  replicated  1/1       cockroachdb/cockroach:{{page.release_info.version}}
    dr81a756gaa6  cockroachdb-2  replicated  1/1       cockroachdb/cockroach:{{page.release_info.version}}
    il4m7op1afg9  cockroachdb-3  replicated  1/1       cockroachdb/cockroach:{{page.release_info.version}}
    ~~~

    {{site.data.alerts.callout_success}}서비스의 정의는 CockroachDB 노드가 <code>stderr</code>에 로그하도록 지시해서, 문제 해결을 위해 노드의 로그에 액세스해야 하는 경우 , 컨테이너가 실행 중인 인스턴스에서 <a href="https://docs.docker.com/engine/reference/commandline/logs/"><code>sudo docker logs &lt;container id&gt;</code></a> 을 사용하십시오.{{site.data.alerts.end}}

3. 이제 CockroachDB의 모든 노드가 실행중이지만, 여전히 그것들이 함께 새로운 클러스터를 초기화하게 해야 합니다. 그러기 위해서, `sudo docker run` 명령을 사용하여 노드 중 하나에 대해 `cockroach init` 을 실행하십시오. `cockroach init` 명령은 클러스터를 초기화하고, 그것을 사용 가능한 상태로 만들것입니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker run -it --rm --network cockroachdb --mount type=bind,source="$(pwd)/certs",target=/cockroach/certs,readonly cockroachdb/cockroach:{{page.release_info.version}} init --host=cockroachdb-1 --certs-dir=certs
    ~~~

    We mount the `certs` directory as a volume inside the container because it contains the `root` user's client certificate and key, which we need to talk to the cluster.

## 단계 7. 기본제공 SQL 클라이언트 사용

1. `sudo docker run` 명령을 사용하여 CockroachDB 네트워크에 연결된 새 컨테이너를 시작하고 내장된 SQL 셸을 실행한 후 클러스터에 연결하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker run -it --rm --network cockroachdb --mount type=bind,source="$(pwd)/certs",target=/cockroach/certs,readonly cockroachdb/cockroach:{{page.release_info.version}} sql --host=cockroachdb-1 --certs-dir=certs
    ~~~

2. `insecurenodetest` 데이터베이스 생성:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE securenodetest;
    ~~~

3. [Create a user with a password](create-user.html#create-a-user-with-a-password).

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE USER roach WITH PASSWORD 'Q7gc8rEdS';
    ~~~

  You will need this username and password to access the Admin UI in Step 8.

4. SQL 셸을 종료하기 위해 **CTRL-D**, **CTRL-C**, 또는 `\q` 를 사용합니다.

## 단계 8. 클러스터 모니터링

클러스터의 관리자 UI를 보기위해:

1. 브라우저 `http://<any node's external IP address>:8080` 로 접속합니다.
{{site.data.alerts.callout_info}}It's possible to access the Admin UI from outside of the swarm because you published port <code>8080</code> externally in the first node's service definition. However, your browser will consider the CockroachDB-created certificate invalid, so you’ll need to click through a warning message to get to the UI.{{site.data.alerts.end}}

2. On accessing the Admin UI, you will see a Login screen, where you will need to enter your username and password created in Step 7.

3. 관리 UI에서, 클러스터가 예상대로 실행되고 있는지 확인하십시오.:

1. **노드 목록** 을 보고 모든 노드가 클러스터에 성공적으로 가입했는지 확인하십시오.

2. 왼쪽에 있는 **데이터베이스** 탭을 클릭하여 `insecurenodetest` 이 나열되어 있는지 확인하십시오.

## 단계 9. 노드 고장 시뮬레이션

각 노드마다 하나씩, 3개의 서비스 정의를 가지고 있기 때문에, 도커 스웜은 항상 3개의 노드가 실행되도록 할 것입니다. 노드가 실패할 경우, 도커 스웜은 자동으로 동일한 네트워크 ID와 스토리지를 가진 다른 노드를 생성합니다.

이 행동을 보려면:

1. 어떤 경우든, `수도 도커 ps` 명령을 사용하여 CockroachDB 노드를 실행하는 컨테이너의 ID를 얻으십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker ps | grep cockroachdb
    ~~~

    ~~~
    32769a6dd664        cockroachdb/cockroach:{{page.release_info.version}}   "/cockroach/cockroach"   10 minutes ago        Up 10 minutes         8080/tcp, 26257/tcp   cockroachdb-2.1.0wigdh8lx0ylhuzm4on9bbldq
    ~~~

2. 노드를 암시적으로 중단하는 컨테이너를 제거하려면, `수도 도커 죽이기` 를 사용하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker kill <container ID>
    ~~~

3. 새로운 컨테이너에서 노드가 재시작하는 것을 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker ps | grep cockroachdb
    ~~~

    ~~~
    4a58f86e3ced        cockroachdb/cockroach:{{page.release_info.version}}   "/cockroach/cockroach"   7 seconds ago       Up 1 seconds        8080/tcp, 26257/tcp   cockroachdb-2.1.cph86kmhhcp8xzq6a1nxtk9ng
    ~~~

4. 관리 UI에서, **노드 목록** 을 보고 3개 노드가 모두 활성 상태인지 확인하십시오.

## 단계 10. 클러스터 확장

CockroachDB 클러스터에서 노드 수를 늘리려면 다음과 같이 하십시오:

1. 추가적인 인스턴스를 생성하십시오 ([단계 1](#step-1-create-instances)을 보십시오).
2. 인스턴스에 도커엔진을 설치하십시오 ([단계 2](#step-2-install-docker-engine)을 보십시오).
3. 워커노드로서 인스터를 스웜에 결합하십시오 ([단계 3.2](#step-3-start-the-swarm)을 보십시오).
4. 새 서비스를 생성하여 다른 노드를 시작하고,이를 CockroachDB 클러스터와 결합하십시오 ([단계 5.1](#step-5-start-the-cockroachdb-cluster)을 보십시오).

## 단계 11. 클러스터 중단

CockroachDB 클러스터를 중단하려면,관리자 노드를 실행하는 인스턴스에서 서비스를 제거하십시오:
{% include copy-clipboard.html %}
~~~ shell
$ sudo docker service rm cockroachdb-1 cockroachdb-2 cockroachdb-3
~~~

~~~
cockroachdb-1
cockroachdb-2
cockroachdb-3
~~~

서비스에 사용되는 영구적인 볼륨을 제거하고 싶을 수 있습니다. 각 인스턴스에서 이 작업을 수행하려면 다음과 같이 하십시오:

{% include copy-clipboard.html %}
~~~ shell
# Identify the name of the local volume:
$ sudo docker volume ls
~~~

~~~
cockroachdb-1
~~~

{% include copy-clipboard.html %}
~~~ shell
# Remove the local volume:
$ sudo docker volume rm cockroachdb-1
~~~

{% include copy-clipboard.html %}
~~~ shell
# Identify the name of secrets:
$ sudo docker secrets ls
~~~

~~~
ca-crt
cockroachdb-1-crt
cockroachdb-1-key
~~~

{% include copy-clipboard.html %}
~~~ shell
# Remove the secrets:
$ sudo docker secret rm ca-crt cockroachdb-1-crt cockroachdb-1-key
~~~

## 또 다른 참고

{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
