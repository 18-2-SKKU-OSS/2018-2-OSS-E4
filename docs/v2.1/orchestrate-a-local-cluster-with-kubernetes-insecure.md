---
title: Orchestration
summary: Orchestrate the deployment and management of an local cluster using Kubernetes.
toc: true
---

이 섹션의 다른 튜토리얼에서는 CockroachDB가 작업을 자동화하는 방법을 설명합니다. 이 기본 제공 자동화(automation) 기능 외에도 타사 조정 ( [orchestration](orchestration.html) ) 시스템을 사용하여 구축 및 확장, 전체 클러스터 관리에 이르기까지 훨씬 더 많은 작업을 간소화하고 자동화할 수 있습니다.

이 페이지에서는 오픈 소스 Kubernetes 조정 시스템을 사용하여 간단한 데모를 보여줍니다. 몇 개의 구성 파일을 시작으로, 보안되지 않은 3-노드 로컬 클러스터를 빠르게 만들 수 있다. 클러스터에 대해 부하 발생기(load generator)를 실행한 다음 노드 장애를 시뮬레이션하여 Kubernet이 수동 개입 없이 auto-restarts 를 수행하는 방법을 확인하십시오. 그런 다음 단일 명령으로 클러스터를 축소하고 클러스터를 종료합니다.

{{site.data.alerts.callout_info}}프로덕션에서 물리적으로 분산된 클러스터를 조정하려면 <a href="orchestration.html">Orchestrated Deployment</a> 를 참조하십시오.{{site.data.alerts.end}}


## 시작하기 전에

시작하기 전에, Kubernetes 관련 용어를 알아봅시다:

특징 | 설명
--------|------------
[minikube](http://kubernetes.io/docs/getting-started-guides/minikube/) | 로컬 워크스테이션의 VM 내에서 Kubernetes 클러스터를 실행하는 데 사용할 도구
[pod](http://kubernetes.io/docs/user-guide/pods/) | A pod is a group of one of more Docker containers. In this tutorial, all pods will run on your local workstation, each containing one Docker container running a single CockroachDB node. You'll start with 3 pods and grow to 4. 더커 (Docker) 컨테이너 중 하나의 그룹. 이 튜토리얼에서는 모든 포드가 로컬 워크스테이션에서 실행되며, 각 포드는 단일 CockroachDB 노드를 실행하는 Docker 컨테이너 하나를 포함합니다. 이 튜토리얼에서 포드는 3 포드로 시작하여 4포드로 늘어납니다.
[StatefulSet](http://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/) | StatefulSet은 상태 저장 장치로 취급되는 포드의 그룹이며, 각 포드는 구별할 수 있는 네트워크 ID를 가지며, 재시작 시 항상 동일한 영구 스토리지에 다시 바인딩된다. StatefulSets는 버전 1.5에서 베타 버전에 도달한 후 Kubernets 버전 1.9에서는 안정적인 것으로 간주됩니다.
[persistent volume](http://kubernetes.io/docs/user-guide/persistent-volumes/) | persistent volume 은 포드에 장착된 로컬 스토리지의 한 부분이다. persistent volume의 수명은 해당 볼륨을 사용하는 포드의 수명과 분리되어 각 CockroachDB 노드가 재시작 시 동일한 스토리지에 다시 바인딩되도록 보장한다.<br><br>'minikube'를 사용할 때 persistent volume은 외부 임시 디렉토리로, 수동으로 삭제하거나 전체 쿠베르네 클러스터를 삭제할 때까지 지속됩니다.
[persistent volume claim](http://kubernetes.io/docs/user-guide/persistent-volumes/#persistentvolumeclaims) | 포드 (CockroachDB 노드 당 하나씩)가 생성되면 각 포드는 해당 노드의 내구성 스토리지를 "claim(청구)"하기 위해 persistent volume claim을 요청합니다.

## Step 1. Kubernetes 시작하기

1. Follow Kubernetes' to install `minikube`, the tool used to run Kubernetes locally, for your OS. This includes installing a hypervisor and `kubectl`, the command-line tool used to managed Kubernetes from your local workstation. Kubernetes의 [documentation](https://kubernetes.io/docs/tasks/tools/install-minikube/)에 따라 Kubernetes를 로컬에서 실행하는 데 사용되는 도구 인 minikube를 OS에 설치하십시오. 여기에는 로컬 워크 스테이션에서 Kubernetes를 관리하는 데 사용되는 command-line 도구인 hypervisor 및 'kubectl' 설치가 포함됩니다. 

    {{site.data.alerts.callout_info}}<code>minikube</code> 버전 0.21.0 이상을 설치해야합니다. 이전 버전에는 CockroachDB StatefulSet 구성에 사용 된 <code>maxUnavailability</code> 필드 및 <code>PodDisruptionBudget</code> 리소스 유형을 지원하는 Kubernetes 서버가 포함되어 있지 않습니다.{{site.data.alerts.end}}

2. 로컬 Kubernetes 클러스터 시작하기:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ minikube start
    ~~~

## Step 2. CockroachDB 노드 시작하기

클러스터를 수동으로 시작할 때는 노드당 한 번씩 <code>cockroach start</code> 명령을 여러 번 실행하십시오. 이 단계에서는 Kubernetes StatefulSet 구성을 대신 사용하여 단일 명령으로 노드 3개를 시작하는 데 필요한 작업보다 간단하게 작업을 수행할 수 있습니다.
{% include {{ page.version.version }}/orchestration/start-cluster.md %}

## Step 3. 클러스터 초기화하기

{% include {{ page.version.version }}/orchestration/initialize-cluster-insecure.md %}

## Step 4. Test the cluster

클러스터를 테스트하려면, 기본 제공 SQL 클라이언트를 사용하기위한 임시 포드를 실행 한 다음 배포 구성(deployment configuration) 파일을 사용하여 다른 포드의 클러스터에 대해 high-traffic load generator(하이트래픽 부하 생성기)를 실행하십시오.

{% include {{ page.version.version }}/orchestration/test-cluster-insecure.md %}

4. Use our [`example-app.yaml`](https://github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/example-app.yaml) file to launch a pod and run a load generator against the cluster from the pod:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/example-app.yaml
    ~~~

    ~~~
    deployment "example" created
    ~~~

5. Verify that the pod for the load generator was added successfully:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    ~~~
    NAME                      READY     STATUS    RESTARTS   AGE
    cockroachdb-0             1/1       Running   0          28m
    cockroachdb-1             1/1       Running   0          27m
    cockroachdb-2             1/1       Running   0          10m
    example-545f866f5-2gsrs   1/1       Running   0          25m
    ~~~

## Step 5. Monitor the cluster

To access the [Admin UI](admin-ui-overview.html) and monitor the cluster's state and the load generator's activity:

1. Port-forward from your local machine to one of the pods:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl port-forward cockroachdb-0 8080
    ~~~

    ~~~
    Forwarding from 127.0.0.1:8080 -> 8080
    ~~~

2. Go to <a href="http://localhost:8080/" data-proofer-ignore>http://localhost:8080</a> and click **Metrics** on the left-hand navigation bar.

3. On the **Overview** dashboard, note that there are 3 healthy nodes with many SQL inserts executing per second across them.

    <img src="{{ 'images/v2.1/automated-operations1.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

4. Click the **Databases** tab on the left to verify that the `bank` database you created manually, as well as the `kv` database created by the load generated, are listed.

## Step 6. Simulate node failure

{% include {{ page.version.version }}/orchestration/kubernetes-simulate-failure.md %}

## Step 7. Scale the cluster

1. Use the `kubectl scale` command to add a pod for another CockroachDB node:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl scale statefulset cockroachdb --replicas=4
    ~~~

    ~~~
    statefulset "cockroachdb" scaled
    ~~~

2. Verify that the pod for a fourth node, `cockroachdb-3`, was added successfully:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    ~~~
    NAME                      READY     STATUS    RESTARTS   AGE
    cockroachdb-0             1/1       Running   0          28m
    cockroachdb-1             1/1       Running   0          27m
    cockroachdb-2             1/1       Running   0          10m
    cockroachdb-3             1/1       Running   0          5s
    example-545f866f5-2gsrs   1/1       Running   0          25m
    ~~~

## Step 8. Stop the cluster

- **If you plan to restart the cluster**, use the `minikube stop` command. This shuts down the minikube virtual machine but preserves all the resources you created:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ minikube stop
    ~~~

    ~~~
    Stopping local Kubernetes cluster...
    Machine stopped.
    ~~~

    You can restore the cluster to its previous state with `minikube start`.

- **If you do not plan to restart the cluster**, use the `minikube delete` command. This shuts down and deletes the minikube virtual machine and all the resources you created, including persistent volumes:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ minikube delete
    ~~~

    ~~~
    Deleting local Kubernetes cluster...
    Machine deleted.
    ~~~

    {{site.data.alerts.callout_success}}To retain logs, copy them from each pod's <code>stderr</code> before deleting the cluster and all its resources. To access a pod's standard error stream, run <code>kubectl logs &lt;podname&gt;</code>.{{site.data.alerts.end}}

## See also

Explore other core CockroachDB benefits and features:

{% include {{ page.version.version }}/misc/explore-benefits-see-also.md %}

You might also want to learn how to [orchestrate a production deployment of CockroachDB with Kubernetes](orchestrate-cockroachdb-with-kubernetes.html).
