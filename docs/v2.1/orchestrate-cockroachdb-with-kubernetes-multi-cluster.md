---
title: Orchestrate CockroachDB Across Multiple Kubernetes Clusters
summary: How to use Kubernetes to orchestrate the deployment, management, and monitoring of an insecure CockroachDB cluster across multiple Kubernetes clusters in different regions.
toc: true
redirect_from: orchetrate-cockroachdb-with-kubernetes-multi-region.html
---

이 페이지에서는 [StatefulSet](http://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/) 기능을 사용하여 각 클러스터 내의 컨테이너를 관리하고 DNS를 통해 연결하여 서로 다른 지역에있는 3개의 [Kubernetes](http://kubernetes.io/) 클러스터 간에 안전한 CockroachDB 배포를 조정하는 방법을 보여줍니다.

단일 Kubernetes에 배포하려는 경우, [Kubernetes Single-Cluster Deployment](orchestrate-cockroachdb-with-kubernetes.html)를 참조하십시오. 또한 Kubernet에서 CockroachDB를 실행할 때 알아야 할 잠재적 성능 병목 현상에 대한 자세한 내용과 더 나은 성능을 위해 배포를 최적화하는 방법에 대한 지침은 [CockroachDB Performance on Kubernetes](kubernetes-performance.html)를 참조하십시오.

## Before you begin

시작하기 전에, Kubernetes 관련 용어 및 제약을 알아봅시다:

### Kubernetes 관련 용어

Feature | Description
--------|------------
instance | 물리 또는 가상 머신. 이 튜토리얼에서는 각각 다른 지역에 있는 세 개의 독립된 Kubernet 클러스터의 일부로 인스턴스를 실행합니다.
[pod](http://kubernetes.io/docs/user-guide/pods/) | 포드는 더커(Docker) 컨테이너 중 하나의 그룹이다. 이 튜토리얼에서 각 포드는 별도의 인스턴스에서 실행되며, 단일 CockroachDB 노드를 실행하는 Docker 컨테이너 하나를 포함합니다. 포드는 3 포드로 시작하여 4포드로 늘어납니다.
[StatefulSet](http://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/) | StatefulSet은 상태 저장 장치로 취급되는 포드의 그룹이며, 각 포드는 구별할 수 있는 네트워크 ID를 가지며, 재시작 시 항상 동일한 영구 스토리지에 다시 바인딩됩니다. StatefulSets는 버전 1.5에서 베타 버전에 도달한 후 Kubernets 버전 1.9에서는 안정적인 것으로 간주됩니다.
[persistent volume](http://kubernetes.io/docs/user-guide/persistent-volumes/) | 영구 볼륨은 포드에 마운트된 네트워크 스토리지(GCE의 영구 디스크, AWS의 Elastic Block Store)의 한 부분이다. 영구 볼륨의 수명은 해당 볼륨을 사용하는 포드의 수명과 분리되어 각 CockroachDB 노드가 재시작 시 동일한 스토리지에 다시 바인딩되도록 보장합니다. 이 튜토리얼에서는 동적 볼륨 프로비저닝을 사용할 수 있다고 가정합니다. 그렇지 않은 경우, [persistent volume claims(영구 볼륨 요청)](http://kubernetes.io/docs/user-guide/persistent-volumes/#persistentvolumeclaims) 이 이루어져야 합니다.
[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) | RBAC 또는 Role 기반 액세스 제어는 Kubernetes가 클러스터 내에서 사용 권한을 관리하는 데 사용하는 시스템입니다. API 리소스(예:`pod`)에 대한 작업(예: `get` 또는 `create`)을 수행하려면 고객은 이를 허용하는 `Role`을 가져야 합니다.
[namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) | 네임스페이스는 Kubernets 클러스터 내의 리소스와 이름에 대한 범위를 제공합니다. 리소스 이름은 네임스페이스 내에서 고유해야 하지만 네임스페이스 사이에서는 그렇지 않습니다. 대부분의 Kubernetes 클라이언트 명령은 기본적으로 `default` 네임스페이스를 사용하지만, 다른 네임스페이스의 리소스에도 사용할 수 있다.
[kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) | `kubectl`은 Kubernetes 클러스터에 대한 명령을 실행하기 위한 커맨드라인 인터페이스입니다.
[kubectl context](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-context-and-configuration) | `kubectl` "context" 는 연결 및 인증할 Kubernetes 클러스터를 명시합니다. 향후 모든 `kubectl` 명령어가 클러스터와 대화할 수 있도록 `kubectl use-context <context-name>` 명령을 사용하여 컨텍스트를 기본값으로 설정하거나, 거의 모든 `kubectl` 명령에서 `--context=<context-name>`플래그를 지정하여 명령을 실행할 클러스터를 지정할 수 있습니다. 각각 다른 지역의 Kubernetes 클러스터에 대한 명령을 실행하기 위해, 이 튜토리얼에서는 `--context` 플래그를 많이 사용합니다.

### UX differences from running in a single cluster

이러한 지침은 구성 스크립트에 제공하는 각 Kubernets 클러스터에서 CockroachDB를 실행하는 StatefulSet을 생성합니다. 이러한 StatefulSets는 적절한 클러스터에 대해 `kubectl` 명령을 실행함으로써 서로 독립적으로 확장할 수 있습니다. 또한 이 단계는 다른 클러스터의 DNS 서버에서 각 Kubernetes 클러스터의 DNS 서버를 가리켜 특정 영역 범위 접미어(e.g., "*.us-west1-a.svc.cluster.local")에 대한 DNS 조회를 적절한 클러스터의 DNS 서버로 지연될 수 있도록 합니다. 그러나, 이 작업을 수행하기 위해, 클러스터가 실행 중인 영역의 이름이 지정된 네임스페이스에 StatefulSets를 생성하게 됩니다. 따라서 포드 중 하나에 대해 명령을 실행하려면, 예컨대, `kubectl logs cockroachdb-0` 대신 `kubectl logs cockroachdb-0 --namespace=us-west1-a`와 같이 실행해야합니다. 또는,`kubectl` 컨텍스트가 기본적으로 해당 네임 스페이스를 사용하여 해당 클러스터에 대해 명령을 실행하도록 구성 할 수 있습니다. [configure your `kubectl` context to default to using that namespace for commands run against that cluster](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#setting-the-namespace-preference).


CockroachDB 포드는 기본 네임스페이스가 아닌 네임스페이스에 있는 경우, 기본 네임스페이스에서 CockroachDB와 대화하려는 모든 클라이언트 애플리케이션은 단일 클러스터 설정에서 사용되는 일반적인 "cockroachdb-public"가 아니라 "cockroachdb-public.us-west1-a"과 같은 영역 범위 지정(zone-scoped) 서비스 이름과 대화해야 한다는 것을 의미한다는 점에 유의하십시오. 그러나 이 지침에 사용되는 설치 스크립트는 기본 네임스페이스에 추가 [`ExternalName` service](https://kubernetes.io/docs/concepts/services-networking/service/#externalname)을 설정하여 기본 네임스페이스에 있는 클라이언트가 "cockroachdb-public" 주소와 간단히 대화할 수 있도록 한다.


마지막으로, 만약 사용자가 이전에 여러 개의 Kubernetes 클러스터로 작업한 경험이 없는 경우, 주어진 명령을 실행할 클러스터에 대해 생각하는 것을 잊어서 명령에 대해 혼동되는 결과를 가져올 수 있다. `kubectl use-context <context-name>`을 자주 실행하여 명령어 간 컨텍스트를 전환하거나, 올바른 클러스터에서 실행되도록 대부분의 명령어에 `--context=<context-name>`을 추가해야 한다는 점을 기억하십시오.

### 제약

#### Kubernetes 버전

Kubernetes 1.8 이상

#### DNS 노출

이 문서의 접근방식에서, 각 Kubernetes 클러스터의 DNS 서버는 공용 인터넷에서 볼 수 있는 로드 밸런싱된 IP 주소를 통해 이들을 노출시킴으로써 함께 연결된다. 이는 Google Cloud Platform의 내부 로드 밸런싱 장치가 현재 다른 region에 있는 로드 밸런싱 장치를 사용하는 클라이언트를 지원하지 않기 때문이다. [Google Cloud Platform's Internal Load Balancers do not currently support clients in one region using a load balancer in another region](https://cloud.google.com/load-balancing/docs/internal/#deploying_internal_load_balancing_with_clients_across_vpn_or_interconnect).

Kubernetes 클러스터의 서비스는 공개적으로 액세스할 수 없지만, 그들의 이름은 동기화된 공격자에게 누설될 수 있습니다. 만약 당신이 이것을 용납할 수 없는 경우, 우리에게 알려준다면 다른 선택 사항들을 보여줄 수 있습니다. 
[여기](https://issuetracker.google.com/issues/111021512)에서 내부 로드 밸런서를 사용할 수 있도록 Google을 설득하는 데 당신의 목소리를 낼 수 있습니다.

## Step 1. Kubernetes 클러스터 시작하기

접근한 다중 영역 구축은 3개의 서로 다른 Kubernetes 클러스터와 지역에 걸쳐 라우팅 가능한 포드 IP 주소에 의존합니다. 호스팅되는 Google Kubernetes Engine(GKE) 서비스는 이 요구 사항을 충족하므로 여기에 해당 환경이 설명됩니다. 다른 클라우드 또는 사내에서 실행하려면 이 [기본 네트워크 test](https://github.com/cockroachdb/cockroach/tree/master/cloud/kubernetes/multiregion#pod-to-pod-connectivity)를 사용하여 작동되는지 확인하십시오.

1. [Google Cubernetes Engine Quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart)에 설명된 **시작하기 전** 단계를 완료하십시오.

여기에는 Kubernetes 엔진 클러스터를 만들고 삭제하는 데 사용되는 `gcloud` 설치와 사용자의 워크스테이션에서 Kubernet을 관리하는 데 사용되는 커맨드라인 도구인 `Kubectl` 설치가 포함됩니다.

    {{site.data.alerts.callout_success}}이 설명서는 Google의 Cloud Shell 제품 사용 또는 사용자 컴퓨터의 로컬 셸 사용 옵션을 제공합니다. 이 가이드의 단계를 사용하여 CockroachDB 관리 UI를 보려면 로컬 셸을 선택하십시오.{{site.data.alerts.end}}

2. 로컬 워크스테이션에서 실행해야 하는 [영역](https://cloud.google.com/compute/docs/regions-zones/)을 지정하여 첫 번째 Kubernet 클러스터를 시작하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ gcloud container clusters create cockroachdb1 --zone=<gce-zone>
    ~~~

    ~~~
    Creating cluster cockroachdb1...done.
    ~~~

    이는 지정된 영역에 GKE 인스턴스를 생성하고 이를 `cockroachdb1` 이라는 단일 Kubernetes 클러스터에 결합한다.

    이 프로세스는 몇 분 정도 걸릴 수 있으므로 `Creating cluster cockroachdb1...done` 메시지와 클러스터에 대한 세부 정보가 나타날 때까지 다음 단계로 진행하지 마십시오.

3. 실행해야 하는 [영역](https://cloud.google.com/compute/docs/regions-zones/)을 지정하여 두 번째 Kubernet 클러스터를 시작하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ gcloud container clusters create cockroachdb2 --zone=<gce-zone>
    ~~~

    ~~~
    Creating cluster cockroachdb2...done.
    ~~~

4. 실행해야 하는 [영역](https://cloud.google.com/compute/docs/regions-zones/)을 지정하여 세 번째 Kubernet 클러스터를 시작하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ gcloud container clusters create cockroachdb3 --zone=<gce-zone>
    ~~~

    ~~~
    Creating cluster cockroachdb3...done.
    ~~~

5. 클러스터에 대한 `kubectl` "컨텍스트(contexts)" 를 얻습니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl config get-contexts
    ~~~

    ~~~
    CURRENT   NAME                                                  CLUSTER                                               AUTHINFO                                              NAMESPACE
    *         gke_cockroach-shared_us-east1-b_cockroachdb1          gke_cockroach-shared_us-east1-b_cockroachdb1          gke_cockroach-shared_us-east1-b_cockroachdb1
              gke_cockroach-shared_us-west1-a_cockroachdb2          gke_cockroach-shared_us-west1-a_cockroachdb2          gke_cockroach-shared_us-west1-a_cockroachdb2
              gke_cockroach-shared_us-central1-a_cockroachdb3       gke_cockroach-shared_us-central1-a_cockroachdb3       gke_cockroach-shared_us-central1-a_cockroachdb3                        
    ~~~

    {{site.data.alerts.callout_info}}
    본 튜토리얼의 모든 `kubectl` 명령어는 `--context` 플래그를 사용하여 Kubernetes가 대화하고자 하는 `kubectl` 을 알려줍니다. 각 Kubernetes cluster는 독립적으로 운영되기 때문에 각 클러스터마다 개별적으로 무엇을 해야 하는지 지시하거나 특정 클러스터의 상태를 파악하려면 해당 클러스터의 `kubectl`을 명확히 해야 합니다.

    `CURRENT` 열에 있는 `*` 가 있는 컨텍스트는 `--context` 플래그를 지정하지 않으면 `kubectl`이 기본적으로 대화하는 클러스터를 나타냅니다.
    {{site.data.alerts.end}}
    

6. Google 클라우드 계정과 연결된 이메일 주소 가져오기::

    {% include copy-clipboard.html %}
    ~~~ shell
    $ gcloud info | grep Account
    ~~~

    ~~~
    Account: [your.google.cloud.email@example.org]
    ~~~

    {{site.data.alerts.callout_danger}}
    이 명령은 이메일 주소를 모두 소문자로 반환합니다. 그러나 다음 단계에서는 정확한 대문자를 사용하여 주소를 입력해야 합니다. 예를 들어, if 이메일 주소가 YourName@example.com 라면 yourname@example.com이 아닌 YourName@example.com으로 입력해야 합니다.
    {{site.data.alerts.end}}

6. 각 Kubernetes 클러스터에 대하여  CockroachDB는 이전 단계에서 가져온 전자 메일 주소와 "context" 이름을 사용하여 GKE에서 실행하기 위해 CockroachDB가 필요로 하는 [RBAC 역할을 생성합니다](https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control#prerequisites_for_using_role-based_access_control).

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=<your.google.cloud.email@example.org> --context=<context-name-of-kubernetes-cluster1>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=<your.google.cloud.email@example.org> --context=<context-name-of-kubernetes-cluster2>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=<your.google.cloud.email@example.org> --context=<context-name-of-kubernetes-cluster3>
    ~~~

## Step 2. CockroachDB 시작하기

1. 디렉터리를 생성하고 필요한 스크립트 및 구성 파일을 다운로드하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ mkdir multiregion
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cd multiregion
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ curl -OOOOOOOOO \
    https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/{README.md,client-secure.yaml,cluster-init-secure.yaml,cockroachdb-statefulset-secure.yaml,dns-lb.yaml,example-app-secure.yaml,external-name-svc.yaml,setup.py,teardown.py}
    ~~~

2. `setup.py` 스크립트의 맨 위에서부터, `contexts` 맵을 클러스터 영역 및 "context" 이름을 입력하여 채우십시오. 
    예시:

    ~~~
    context = {
        'us-east1-b': 'gke_cockroach-shared_us-east1-b_cockroachdb1',
        'us-west1-a': 'gke_cockroach-shared_us-west1-a_cockroachdb2',
        'us-central1-a': 'gke_cockroach-shared_us-central1-a_cockroachdb3',
    }
    ~~~

    이전 단계에서 `kubectl` "contexts를 회수했습니다. 다시 가져오려면 다음을 실행하십시오.:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl config get-contexts
    ~~~

3. `setup.py` 스크립트의 맨 위에서부터, `regions` 맵을 클러스터 영역 및 해당 region을 입력하여 채우십시오.  
   예시:

    ~~~
    $ regions = {
        'us-east1-b': 'us-east1',
        'us-west1-a': 'us-west1',
        'us-central1-a': 'us-central1',
    }
    ~~~

    설정 영역은 선택 사항이지만 동일한 영역에 둘 이상의 영역을 사용할 경우 데이터 배치를 다양화하는 CockroachDB의 능력이 향상되기 때문에 권장됩니다. 영역을 지정하지 않으면 맵을 비워 두십시오.
    
4. CockroachhDB를 아직 설치하지 않은 경우, [로컬에 CockroachDB 를 설치하고 `PATH`에 추가하십시오](install-cockroachdb.html). `cockroach` 바이너리는 인증서를 생성하는 데 사용됩니다.

    `cockroach` 바이너리가`PATH`에 없으면`setup.py` 스크립트에서`cockroach_path` 변수를 바이너리 경로로 설정하십시오.

5. 더 나은 성능을 위해 배포를 최적화하려면 [CockroachDB Performance on Kubernetes] (kubernetes-performance.html)를 검토하고`cockroachdb-statefulset-secure.yaml` 파일을 수정하십시오. 이 항목은 필수가 아닌 선택 사항입니다.

6. `setup.py` 스크립트를 실행합니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ python setup.py
    ~~~

    스크립트가 다양한 리소스를 생성하고 CockroachDB 클러스터를 생성하고 초기화하면 많은 출력을 볼 수 있고, 결과적으로`job "cluster-init-secure" created`가 생성됩니다.


7. 각 클러스터의 CockroachDB 포드가 `READY` 열에`1 / 1`이라고 표시되어 클러스터에 성공적으로 가입되었음을 나타내는지 확인하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods --selector app=cockroachdb --all-namespaces --context=<context-name-of-kubernetes-cluster1>
    ~~~

    ~~~
    NAMESPACE    NAME            READY     STATUS    RESTARTS   AGE
    us-east1-b   cockroachdb-0   1/1       Running   0          14m
    us-east1-b   cockroachdb-1   1/1       Running   0          14m
    us-east1-b   cockroachdb-2   1/1       Running   0          14m    
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods --selector app=cockroachdb --all-namespaces --context=<context-name-of-kubernetes-cluster2>
    ~~~

    ~~~
    NAMESPACE       NAME            READY     STATUS    RESTARTS   AGE
    us-central1-a   cockroachdb-0   1/1       Running   0          14m
    us-central1-a   cockroachdb-1   1/1       Running   0          14m
    us-central1-a   cockroachdb-2   1/1       Running   0          14m
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods --selector app=cockroachdb --all-namespaces --context=<context-name-of-kubernetes-cluster3>
    ~~~

    ~~~
    NAMESPACE    NAME            READY     STATUS    RESTARTS   AGE
    us-west1-a   cockroachdb-0   1/1       Running   0          14m
    us-west1-a   cockroachdb-1   1/1       Running   0          14m
    us-west1-a   cockroachdb-2   1/1       Running   0          14m
    ~~~    

    Kubernetes 클러스터의 포드 중 하나만 'READY'로 표시되면 다른 클러스터의 포드가 서로 통신 할 수 있도록 네트워크 방화벽 규칙을 구성해야합니다. 다음 명령을 실행하여 사설 GCE 네트워크에서 포트 26257 (노드 간 트래픽용으로 CockroachDB에서 사용하는 포트)에서 트래픽을 허용하는 방화벽 규칙을 만들 수 있습니다. 개인 네트워크(사설망) 외부에서 들어오는 트래픽은 허용하지 않습니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ gcloud compute firewall-rules create allow-cockroach-internal --allow=tcp:26257 --source-ranges=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
    ~~~

    ~~~
    Creating firewall...done.
    NAME                      NETWORK  DIRECTION  PRIORITY  ALLOW      DENY
    allow-cockroach-internal  default  INGRESS    1000      tcp:26257
    ~~~

{{site.data.alerts.callout_success}}
각 Kubernetes 클러스터에서 StatefulSet 구성은 모든 CockroachDB 노드가`stderr`에 기록하도록 설정합니다. 따라서 문제를 해결하기 위해 포드/노드의 로그에 접근해야하는 경우 persistent volume의 로그를 확인하지 않고 `kubectl logs <podname> --namespace = <cluster-namespace> --context = <cluster-context>`를 사용하십시오.
{{site.data.alerts.end}}

## Step 3. 빌트인 SQL 클라이언트 사용하기

1. Use the `client-secure.yaml` file to launch a pod and keep it running indefinitely, specifying the context of the Kubernetes cluster to run it in:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl create -f client-secure.yaml --context=<cluster-context>
    ~~~

    ~~~
    pod "cockroachdb-client-secure" created
    ~~~

   포드는 앞서 `setup.py` 스크립트에 의해 만들어진 `root` 클라이언트 인증서를 사용합니다. 올바른 네임스페이스와 컨텍스트 조합을 사용하여 이 작업이 세 개의 Kubernet 클러스터 중 하나에서 작동하도록 해야합니다.

2. 포드에 셸을 넣고 CockroachDB [built-in SQL client](use-the-built-in-sql-client.html)를 다시 시작하여 포드가 실행되는 Kubernetes 클러스터의 네임스페이스 및 컨텍스트를 지정합니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl exec -it cockroachdb-client-secure --context=<cluster-context> -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
    ~~~

    ~~~
    # Welcome to the cockroach SQL interface.
    # All statements must be terminated by a semicolon.
    # To exit: CTRL + D.
    #
    # Server version: CockroachDB CCL v2.0.5 (x86_64-unknown-linux-gnu, built 2018/08/13 17:59:42, go1.10) (same version as client)
    # Cluster ID: 99346e82-9817-4f62-b79b-fdd5d57f8bda
    #
    # Enter \? for a brief introduction.
    #
    warning: no current database set. Use SET database = <dbname> to change, CREATE DATABASE to make a new database.
    root@cockroachdb-public:26257/>
    ~~~

3. 기본적인 [CockroachDB SQL statements](learn-cockroachdb-sql.html) 몇 개를 실행합니다.

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

3. [사용자 만들고 비밀번호 지정하기](create-user.html#create-a-user-with-a-password):

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE USER roach WITH PASSWORD 'Q7gc8rEdS';
    ~~~

      Step 4에서 관리 UI에 액세스하려면 이 사용자 이름과 암호가 필요할 것입니다.

4. Exit the SQL shell and pod:

    {% include copy-clipboard.html %}
    ~~~ sql
    > \q
    ~~~
   
   포드는 무한정 계속 실행되므로, built-in SQL 클라이언트를 다시 열거나 다른 [`cockroach` 클라이언트 명령](cockroach-commands.html)(예: `cockroach node`)을 실행해야 할 때마다 적절한 명령을 사용하여 step 2를 반복하십시오.

   포드를 삭제하고 필요할 때 재생성하는 편을 선호하는 경우에는 다음을 실행하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl delete pod cockroachdb-client-secure --context=<cluster-context>
    ~~~

## Step 4. 웹 UI 액세스 하기

클러스어의 [Web UI(웹 UI)](admin-ui-overview.html) 에 액세스 하려면:

1. 로컬 컴퓨터에서 Kubernet 클러스터 중 하나의 포드에 포트 포워딩 하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl port-forward cockroachdb-0 8080 --namespace=<cluster-namespace> --context=<cluster-context>
    ~~~

    ~~~
    Forwarding from 127.0.0.1:8080 -> 8080
    ~~~

    {{site.data.alerts.callout_info}}
    `port-forward`은 웹 UI를 보려는 웹 브라우저와 동일한 컴퓨터에서 실행해야 합니다. 클라우드 인스턴스나 다른 로컬 셸에서 이 명령을 실행한 경우 로컬 컴퓨터에서 `kubectl`을 구성하고 위의 `port-forward` 명령을 실행하지 않으면 UI를 볼 수 없습니다.
    {{site.data.alerts.end}}

2. <a href="https://localhost:8080/" data-proofer-ignore>https://localhost:8080</a> 로 이동하여 [Use the built-in SQL client](#step-3-use-the-built-in-sql-client) 단계에서 생성된 사용자 이름과 암호를 사용하여 로그인하십시오.

3. UI에서 **Node List** 를 확인하여 모든 노드가 실행 중인지 확인한 다음 왼쪽의 **Databases** 탭을 클릭하여 `bank`가 나열되어 있는지 확인하십시오.

## Step 5. 데이터센터 장애 시뮬레이션 하기

다중 영역 클러스터를 실행할 때 얻을 수 있는 주요 이점 중 하나는 전체 데이터 센터 또는 영역이 전체적으로 CockroachDB 클러스터의 가용성에 영향을 주지 않고 중단될 수 있다는 것입니다.

이는 다음 작업을 통해 확인해볼 수 있습니다:

1. StatefulSets 중 하나를 포드 0으로 축소하고, 실행 중인 Kubernet 클러스터의 네임스페이스 및 컨텍스트를 지정하십시오.
    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl scale statefulset cockroachdb --replicas=0 --namespace=<cluster-namespace> --context=<cluster-context>
    ~~~

    ~~~
    statefulset "cockroachdb" scaled
    ~~~

2. 관리 UI에서 **Cluster Overview**는 곧 해당 영역의 3개 노드를 **Suspect**로 표시합니다. 5분 이상 기다리면 그 노드들은 **Dead**로 표시될 것입니다. 세 개의 비활성(dead) 노드가 있더라도 다른 노드는 모두 정상이며 다른 지역에서 데이터베이스를 사용하는 클라이언트는 계속 정상적으로 작동한다는 점에 주목하십시오.

3. 영역 중 하나가 중단된 경우에도 클러스터가 완전히 작동하는지 확인이 끝났다면, 다음을 실행하여 영역을 다시 백업하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl scale statefulset cockroachdb --replicas=3 --namespace=<cluster-namespace> --context=<cluster-context>
    ~~~

    ~~~
    statefulset "cockroachdb" scaled
    ~~~

## Step 6. 클러스터 유지 관리하기

### 클러스터 확장하기

각 Kubernetes 클러스터에는 포드가 실행될 수 있는 3개의 노드가 포함되어 있습니다. 동일한 노드에 두 개의 포드가 없도록 하려면 ([production best practices](recommended-production-settings.html)에서 권고한 바와 같이), 새 작업자 노드를 추가한 다음 StatefulSet 구성을 편집하여 다른 포드를 추가하십시오.

1. [Resize your cluster(클러스터 크기 조정하기)](https://cloud.google.com/kubernetes-engine/docs/how-to/resizing-a-cluster).

2. CockroachDB 노드를 추가할 Kubernetes 클러스터의 StatefulSet에 `kubectl scale` 명령을 사용하여 포드를 추가하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl scale statefulset cockroachdb --replicas=4 --namespace=<cluster-namespace> --context=<cluster-context>
    ~~~

    ~~~
    statefulset "cockroachdb" scaled
    ~~~

3. 4번째 노드가 성공적으로 추가되었는지 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods --namespace=<cluster-namespace> --context=<cluster-context>
    ~~~

    ~~~
    NAME                        READY     STATUS    RESTARTS   AGE
    cockroachdb-0               1/1       Running   0          1h
    cockroachdb-1               1/1       Running   0          1h
    cockroachdb-2               1/1       Running   0          7m
    cockroachdb-3               1/1       Running   0          44s
    cockroachdb-client-secure   1/1       Running   0          26m
    ~~~

### 클러스터 업그레이드 하기

CockroachDB의 새로운 버전이 출시됨에 따라 버그 수정, 성능 개선 및 새로운 기능을 얻기 위해 새로운 버전으로 업그레이드하는 것을 강력히 권장합니다. 일반 [CockroachDB 업그레이드 설명서](upgrade-cockroach-version.html)는 CockroachDB 클러스터의 업그레이드를 준비하고 실행하는 방법에 대한 모범 사례를 제공하지만, Kubernetes에서 실제로 프로세스를 중지하고 다시 시작하는 메커니즘은 다소 특별합니다.

Kubernetes는 CockroachDB 노드의 안전한 롤링 업그레이드 프로세스를 수행하는 방법을 알고 있습니다. 사용자가 CockroachDB StatefulSet에 사용된 Docker 이미지를 변경하라고 하면 Kubernetes는 노드를 중지하고 새 이미지로 다시 시작하고 다음 이미지로 이동하기 전에 클라이언트 요청을 수신할 준비가 될 때까지 대기합니다. 자세한 내용은 [Kubernetes documentation](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#updating-statefulsets).을 참조하십시오.


1. 업그레이드를 완료하는 방법을 결정하십시오.

    {{site.data.alerts.callout_info}}이 단계는 v2.0.x에서 v2.1로 업그레이드할 때만 관련이 있습니다. v2.1.x 시리즈 내에서 업그레이드하는 경우 이 단계를 건너뛰십시오.{{site.data.alerts.end}}

    기본적으로 모든 노드가 새 버전을 실행한 후 업그레이드 프로세스는 **auto-finalized(자동 완료)** 가 됩니다. 이를 통해 v2.1에 도입된 특정 성능 개선 및 버그 수정이 가능해집니다. 그러나 완료(finalization) 후에는 v2.0으로 다운그레이드를 더 이상 수행할 수 없을 것입니다. 심각한 오류나 손상이 발생할 경우에는 이전 바이너리를 사용하여 새 클러스터를 시작한 다음 업그레이드를 수행하기 전에 생성된 백업 중 하나에서 복원하는 것이 유일한 옵션입니다.

    업그레이드를 완료하기 전에 업그레이드된 클러스터의 안정성 및 성능을 모니터링할 수 있도록 자동 완료를 사용하지 않도록 설정하는 것을 권장합니다.

    1. 앞서 생성된 `cockroach` 바이너리로 쉘을 포드에 넣고 CockroachDB [built-in SQL client](use-the-built-in-sql-client.html)를 시작하십시오.

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-client-secure --context=<cluster-context> -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
        ~~~

    2. `cluster.preserve_downgrade_option` [cluster setting](cluster-settings.html) 설정:

        {% include copy-clipboard.html %}
        ~~~ sql
        > SET CLUSTER SETTING cluster.preserve_downgrade_option = '2.0';
        ~~~

2. 각 Kubernetes 클러스터에 대해 원하는 Docker 이미지를 변경하여 업그레이드 프로세스를 시작하십시오. 이렇게 하려면 업그레이드할 버전을 선택한 후 다음 명령을 실행하여 "VERSION"을 원하는 새 버전으로 교체하고 관련 네임스페이스 및 Kubernet 클러스터의 "context" 이름을 지정하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl patch statefulset cockroachdb --namespace=<namespace-of-kubernetes-cluster1> --context=<context-name-of-kubernetes-cluster1> --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"cockroachdb/cockroach:VERSION"}]'
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl patch statefulset cockroachdb --namespace=<namespace-of-kubernetes-cluster2> --context=<context-name-of-kubernetes-cluster2> --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"cockroachdb/cockroach:VERSION"}]'
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl patch statefulset cockroachdb --namespace=<namespace-of-kubernetes-cluster3> --context=<context-name-of-kubernetes-cluster3> --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"cockroachdb/cockroach:VERSION"}]'
    ~~~

3. 그런 다음 각 Kubernetes 클러스터에서 포드 상태를 확인할 경우 다음 중 하나가 다시 시작되고 있음을 확인하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods --selector app=cockroachdb --all-namespaces --context=<context-name-of-kubernetes-cluster1>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods --selector app=cockroachdb --all-namespaces --context=<context-name-of-kubernetes-cluster1>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods --selector app=cockroachdb --all-namespaces --context=<context-name-of-kubernetes-cluster1>
    ~~~

    This will continue until all of the pods have restarted and are running the new image.

4. 업그레이드를 완료하십시오.

    {{site.data.alerts.callout_info}}이 단계는 v2.0.x에서 v2.1로 업그레이드할 때만 관련이 있습니다. v2.1.x 시리즈 내에서 업그레이드하는 경우 이 단계를 건너뛰십시오.{{site.data.alerts.end}}

    위의 1단계에서 자동 완료(auto-finalization) 기능을 사용하지 않도록 설정한 경우 업그레이드에 익숙해질 필요가 있는 한(일반적으로 적어도 하루 이상) 클러스터의 안정성 및 성능을 모니터링하십시오. 만약 이 시간 동안 업그레이드를 롤백하기로 결정했다면 이전 바이너리를 사용하여 롤링 재시작 절차를 반복하십시오.

    새 버전이 마음에 들었다면, 자동 완성 기능을 다시 활성화하십시오.:

    1. 앞서 생성된 `cockroach` 바이너리로 쉘을 포드에 넣고 CockroachDB [built-in SQL client](use-the-built-in-sql-client.html)를 시작하십시오.

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-client-secure --context=<cluster-context> -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
        ~~~

    2. auto-finalization를 다시 사용하도록 설정합니다.

        {% include copy-clipboard.html %}
        ~~~ sql
        > RESET CLUSTER SETTING cluster.preserve_downgrade_option;
        ~~~

### 클러스터 중지하기

1. 클러스터에서 생성된 리소스를 모두 삭제하려면 `setup.py`에서 `contexts` 맵을 복사하고, `teardown.py`로 만든 다음 `teardown.py`를 실행합니다.

    ~~~ shell
    $ python teardown.py
    ~~~
    

    ~~~
    namespace "us-east1-b" deleted
    service "kube-dns-lb" deleted
    configmap "kube-dns" deleted
    pod "kube-dns-5dcfcbf5fb-l4xwt" deleted
    pod "kube-dns-5dcfcbf5fb-tddp2" deleted
    namespace "us-west1-a" deleted
    service "kube-dns-lb" deleted
    configmap "kube-dns" deleted
    pod "kube-dns-5dcfcbf5fb-8csc9" deleted
    pod "kube-dns-5dcfcbf5fb-zlzn7" deleted
    namespace "us-central1-a" deleted
    service "kube-dns-lb" deleted
    configmap "kube-dns" deleted
    pod "kube-dns-5dcfcbf5fb-6ngmw" deleted
    pod "kube-dns-5dcfcbf5fb-lcfxd" deleted
    ~~~

2. 각각의 Kubernetes 클러스터를 중지합니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ gcloud container clusters delete cockroachdb1 --zone=<gce-zone>
    ~~~

    ~~~
    Deleting cluster cockroachdb1...done.
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ gcloud container clusters delete cockroachdb2 --zone=<gce-zone>
    ~~~

    ~~~
    Deleting cluster cockroachdb2...done.
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ gcloud container clusters delete cockroachdb3 --zone=<gce-zone>
    ~~~

    ~~~
    Deleting cluster cockroachdb3...done.
    ~~~

## 더 알아보기

- [Kubernetes Single-Cluster Deployment](orchestrate-cockroachdb-with-kubernetes.html)
- [Kubernetes Performance Guide](kubernetes-performance.html)
{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
