---
title: Orchestrate CockroachDB with Docker Swarm
summary: How to orchestrate the deployment and management of an insecure three-node CockroachDB cluster as a Docker swarm.
toc: true
---

<div class="filters filters-big clearfix">
  <a href="orchestrate-cockroachdb-with-docker-swarm.html"><button class="filter-button">Secure</button>
  <button class="filter-button current"><strong>Insecure</strong></button></a>
</div>

이 페이지에서는 안전하지 않은 세 노드 CockroachDB 클러스터의 배포, 관리를 [도커 엔진 무리](https://docs.docker.com/engine/swarm/)로 바꾸는 것을 보여줍니다.

프로덕션에서 CockroachDB를 사용하는 것을 계획하는 경우, 대신 보안 클러스터를 사용하는 것을 추천합니다. 설명서를 보러면 위의 **Secure** 을 선택하십시오.


## 시작하기 전에

시작하기 전에, 용어를 복습하는 것은 도움이 됩니다:

특징 | 묘사
--------|------------
인스턴스 | 실제 또는 가상의 머신. 이 튜토리얼에서는, CockroachDB노드 당 3개씩을 사용하십시오.
[도커 엔진](https://docs.docker.com/engine/) | 이것은 컨테이너를 생성하고 실행하는 핵심 Docker 어플리케이션입니다. 이 튜토리얼에서는, 세 개의 예시 각각에 도커 엔진을 설치하고 시작하십시오.
[스웜](https://docs.docker.com/engine/swarm/key-concepts/#/swarm) | 스웜은 단일, 가상의 호스트로 된 도커 엔진 그룹입니다.
[노드 무리](https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/) | 스웜의 각 멤버는 하나의 노드로 구성됩니다. 이 튜토리얼에서 각 인스턴스는 스웜 노드가 되며, 하나는 마스터 노드, 다른 두 개는 워커 노드 역할을 한다. 작업노드로 작업을 전달하는 마스터 노드에 서비스를 제출하십시오.
[서비스](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/) | 서비스는 스웜 노드에서 실행할 테스크의 정의입니다. 이 튜토리얼에서, 컨테이너 안에서 CockrochDB 노드를 시작하고 단일 클러스터에 결합하는 각각의 세 가지 서비스를 정의하십시오. 각 서비스는 확인 가능한 DNS 이름을 통해 재시작 시 안정적인 네트워크 ID를 보장합니다.
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

2. 도커 데몬이 백그라운드에서 실행 중인지 화긴:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker version
    ~~~

## 단계 3. 스웜 시작

1. 관리자 노드를 실행할 인스턴스에, [스웜을 설치](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/) 하십시오.

    Take note of the output for `docker swarm init` as it includes the command you'll use in the next step. It should look like this:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker swarm init --advertise-addr 10.142.0.2
    ~~~

    ~~~
    Swarm initialized: current node (414z67gr5cgfalm4uriu4qdtm) is now a manager.

    To add a worker to this swarm, run the following command:

      $ docker swarm join \
      --token SWMTKN-1-5vwxyi6zl3cc62lqlhi1jrweyspi8wblh2i3qa7kv277fgy74n-e5eg5c7ioxypjxlt3rpqorh15 \
      10.142.0.2:2377

    To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
    ~~~

2. 다른 두 개의 인스턴스, 단계 1의 출력 명령인 `도커 스웜 합치기`를 실행함으로써 [스웜에 합쳐지는 워커 노드를 만드십시오](https://docs.docker.com/engine/swarm/swarm-tutorial/add-nodes/):

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker swarm join \
         --token SWMTKN-1-5vwxyi6zl3cc62lqlhi1jrweyspi8wblh2i3qa7kv277fgy74n-e5eg5c7ioxypjxlt3rpqorh15 \
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

관리자 노드를 실행하는 인스턴스에서 스웜의 컨테이너가 서로 통신할 수 있도록 오버레이 네트워크를 생성하십시오.:

{% include copy-clipboard.html %}
~~~ shell
$ sudo docker network create --driver overlay --attachable cockroachdb
~~~

The `--attachable` option enables non-swarm containers running on Docker to access services on the network, which makes the service easier to use interactively.

## 단계 5. CockroachDB 클러스터 시작

1. 관리자 노드를 실행하는 인스턴스에서 각 CockroachDB 노드에 대해 하나의 스웜 서비스 생성합니다:

    {% include copy-clipboard.html %}
    ~~~
    # Start the first service:
    $ sudo docker service create \
    --replicas 1 \
    --name cockroachdb-1 \
    --hostname cockroachdb-1 \
    --network cockroachdb \
    --mount type=volume,source=cockroachdb-1,target=/cockroach/cockroach-data,volume-driver=local \
    --stop-grace-period 60s \
    --publish 8080:8080 \
    cockroachdb/cockroach:{{page.release_info.version}} start \
    --join=cockroachdb-1:26257,cockroachdb-2:26257,cockroachdb-3:26257 \
    --cache=.25 \
    --max-sql-memory=.25 \
    --logtostderr \
    --insecure
    ~~~

    {% include copy-clipboard.html %}
    ~~~
    # Start the second service:
    $ sudo docker service create \
    --replicas 1 \
    --name cockroachdb-2 \
    --hostname cockroachdb-2 \
    --network cockroachdb \
    --mount type=volume,source=cockroachdb-2,target=/cockroach/cockroach-data,volume-driver=local \
    --stop-grace-period 60s \
    cockroachdb/cockroach:{{page.release_info.version}} start \
    --join=cockroachdb-1:26257,cockroachdb-2:26257,cockroachdb-3:26257 \
    --cache=.25 \
    --max-sql-memory=.25 \
    --logtostderr \
    --insecure
    ~~~

    {% include copy-clipboard.html %}
    ~~~
    # Start the third service:
    $ sudo docker service create \
    --replicas 1 \
    --name cockroachdb-3 \
    --hostname cockroachdb-3 \
    --network cockroachdb \
    --mount type=volume,source=cockroachdb-3,target=/cockroach/cockroach-data,volume-driver=local \
    --stop-grace-period 60s \
    cockroachdb/cockroach:{{page.release_info.version}} start \
    --join=cockroachdb-1:26257,cockroachdb-2:26257,cockroachdb-3:26257 \
    --cache=.25 \
    --max-sql-memory=.25 \
    --logtostderr \
    --insecure
    ~~~

    These commands each create a service that starts a container, joins it to the overlay network, and starts a CockroachDB node inside the container mounted to a local volume for persistent storage. Let's look at each part:
    - `sudo docker service create`: The Docker command to create a new service.
    - `--replicas`: The number of containers controlled by the service. Since each service will control one container running one CockroachDB node, this will always be `1`.
    - `--name`: The name for the service.
    - `--hostname`: The hostname of the container. It will listen for connections on this address.
    - `--network`: The overlay network for the container to join. See [Step 4. Create an overlay network](#step-4-create-an-overlay-network) for more details.
    - `--mount`: This flag mounts a local volume with the same name as the service. This means that data and logs for the node running in this container will be stored in `/cockroach/cockroach-data` on the instance and will be reused on restart as long as restart happens on the same instance, which is not guaranteed.
     {{site.data.alerts.callout_info}}If you plan on replacing or adding instances, it's recommended to use remote storage instead of local disk. To do so, <a href="https://docs.docker.com/engine/reference/commandline/volume_create/">create a remote volume</a> for each CockroachDB instance using the volume driver of your choice, and then specify that volume driver instead of the <code>volume-driver=local</code> part of the command above, e.g., <code>volume-driver=gce</code> if using the <a href="https://github.com/mcuadros/gce-docker">GCE volume driver</a>.
    - `--stop-grace-period`: This flag sets a grace period to give CockroachDB enough time to shut down gracefully, when possible.
    - `--publish`: This flag makes the Admin UI accessible at the IP of any instance running a swarm node on port `8080`. Note that, even though this flag is defined only in the first node's service, the swarm exposes this port on every swarm node using a routing mesh. See [Publishing ports](https://docs.docker.com/engine/swarm/services/#publish-ports) for more details.
    - `cockroachdb/cockroach:{{page.release_info.version}} start ...`: The CockroachDB command to [start a node](start-a-node.html) in the container in insecure mode and instruct other cluster members to talk to each other using their persistent network addresses, which match the services' names.

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

    {{site.data.alerts.callout_success}}The service definitions tell the CockroachDB nodes to log to <code>stderr</code>, so if you ever need access to a node's logs for troubleshooting, use <a href="https://docs.docker.com/engine/reference/commandline/logs/"><code>sudo docker logs &lt;container id&gt;</code></a> from the instance on which the container is running.{{site.data.alerts.end}}

3. Now all the CockroachDB nodes are running, but we still have to explicitly tell them to initialize a new cluster together. To do so, use the `sudo docker run` command to run the `cockroach init` command against one of the nodes. The `cockroach init` command will initialize the cluster, bringing it into a usable state.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker run -it --rm --network=cockroachdb cockroachdb/cockroach:{{page.release_info.version}} init --host=cockroachdb-1 --insecure
    ~~~


## 단계 6. 기본제공 SQL 클라이언트 사용

1. Use the `sudo docker run` command to start a new container attached to the CockroachDB network, run the built-in SQL shell, and connect it to the cluster:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker run -it --rm --network=cockroachdb cockroachdb/cockroach:{{page.release_info.version}} sql --host=cockroachdb-1 --insecure
    ~~~

2. Create an `insecurenodetest` database:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE insecurenodetest;
    ~~~

3. Use **CTRL-D**, **CTRL-C**, or `\q` to exit the SQL shell.

## 단계 7. 클러스터 모니터링

클러스터의 관리자 UI를 보기위해, 브라우저 `http://<any node's external IP address>:8080` .

{{site.data.alerts.callout_info}}It's possible to access the Admin UI from outside of the swarm because you published port <code>8080</code> externally in the first node's service definition.{{site.data.alerts.end}}

이 페이지에서, 클러스터가 예상대로 실행되고 있는지 확인하십시오.:

1. **노드 목록** 을 보고 모든 노드가 클러스터에 성공적으로 가입했는지 확인하십시오.

2. 왼쪽에 있는 **Databases** 탭을 클릭하여 `insecurenodetest` 이 나열되어 있는지 확인하십시오.

## 단계 8. 노드 고장 시뮬레이션

각 노드마다 하나씩, 3개의 서비스 정의를 가지고 있기 때문에, 도커 스웜은 항상 3개의 노드가 실행되도록 할 것입니다. 노드가 실패할 경우, 도커 스웜은 자동으로 동일한 네트워크 ID와 스토리지를 가진 다른 노드를 생성합니다.

이 행동을 보러면:

1. 어떤 경우든, use the `수도 도커 ps` 명령을 사용하여 CockroachDB 노드를 실행하는 컨테이너의 ID를 얻으십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker ps | grep cockroachdb
    ~~~

    ~~~
    9539871cc769        cockroachdb/cockroach:{{page.release_info.version}}   "/cockroach/cockroach"   10 minutes ago        Up 10 minutes         8080/tcp, 26257/tcp   cockroachdb-0.1.0wigdh8lx0ylhuzm4on9bbldq
    ~~~

2. 노드를 암시적으로 중단하는 컨테이너를 제거하려면, `수도 도커 죽이기` 를 사용하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker kill 9539871cc769
    ~~~

3. 새로운 컨테이너에서 노드가 재시작하는 것을 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo docker ps | grep cockroachdb
    ~~~

    ~~~
    4a58f86e3ced        cockroachdb/cockroach:{{page.release_info.version}}   "/cockroach/cockroach"   7 seconds ago       Up 1 seconds        8080/tcp, 26257/tcp   cockroachdb-0.1.cph86kmhhcp8xzq6a1nxtk9ng
    ~~~

4. 관리 UI에서, **노드 목록** 을 보고 3개 노드가 모두 활성 상태인지 확인하십시오.

## 단계 9. 클러스터 확장

CockroachDB 클러스터에서 노드 수를 늘리려면 다음과 같이 하십시오:

1. 추가적인 인스턴스를 생성하십시오 ([단계 1](#step-1-create-instances)을 보십시오).
2. 인스턴스에 도커엔진을 설치하십시오 ([단계 2](#step-2-install-docker-engine)을 보십시오).
3. 워커노드로서 인스터를 스웜에 결합하십시오 ([단계 3.2](#step-3-start-the-swarm)을 보십시오).
4. 새 서비스를 생성하여 다른 노드를 시작하고,이를 CockroachDB 클러스터와 결합하십시오 ([단계 5.1](#step-5-start-the-cockroachdb-cluster)을 보십시오).

## 단계 10. 클러스터 중단

CockroachDB 클러스터를 중단하려면,관리자 노드를 실행하는 인스턴스에서 서비스를 제거하십시오.:

{% include copy-clipboard.html %}
~~~ shell
$ sudo docker service rm cockroachdb-0 cockroachdb-1 cockroachdb-2
~~~

~~~
cockroachdb-0
cockroachdb-1
cockroachdb-2
~~~

서비스에 사용되는 영구적인 볼륨을 제거하고 싶을 수 있습니다. 각 인스턴스에서 이 작업을 수행하려면 다음과 같이 하십시오.:

{% include copy-clipboard.html %}
~~~ shell
# Identify the name of the local volume:
$ sudo docker volume ls
~~~

~~~
cockroachdb-0
~~~

{% include copy-clipboard.html %}
~~~ shell
# Remove the local volume:
$ sudo docker volume rm cockroachdb-0
~~~

## 또 다른 참고

{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
