---
title: Orchestrate CockroachDB in a Single Kubernetes Cluster
summary: How to orchestrate the deployment, management, and monitoring of a secure 3-node CockroachDB cluster with Kubernetes.
toc: true
secure: true
---

<div class="filters filters-big clearfix">
  <button class="filter-button current"><strong>Secure</strong></button>
  <a href="orchestrate-cockroachdb-with-kubernetes-insecure.html"><button class="filter-button">Insecure</button></a>
</div>

이 페이지에서는 [StatefulSet](http://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/) 기능을 사용하여 단일 [Kubernetes](http://kubernetes.io/) 클러스터에서 보안 3 노드 CockroachDB 클러스터의 배포, 관리 및 모니터링을 조율하는 방법을 보여줍니다.

대신 다른 지역의 다중 Kubernetes 클러스터에 배포하려면, [Kubernetes 다중 클러스터 배포](orchestrate-cockroachdb-with-kubernetes-multi-cluster.html)를 보십시오. 또한, Kubernetes에서 CockroachDB를 실행할 때 알아야 할 잠재적인 성능 병목 현상에 대한 자세한 내용과 성능 향상을 위해 배포를 최적화하는 방법에 대한 지침은 [Kubernetes에서 CockroachDB의 성능](kubernetes-performance.html)을 보십시오. 

## 시작하기 전에

시작하기 전에, Kubernetes 관련 용어 및 현재 제한 사항을 검토하는 것이 좋습니다.

### Kubernetes 용어

특징 | 설명
--------|------------
인스턴스 | 실제 또는 가상 머신. 이 튜토리얼에서는, GCE 또는 AWS 인스턴스를 만들고 로컬 워크 스테이션의 단일 Kubernetes 클러스터에 조인시킵니다.
[포드](http://kubernetes.io/docs/user-guide/pods/) | 포드는 도커 컨테이너 중 하나의 그룹입니다. 이 튜토리얼에서는, 각 포드를 별도의 인스턴스에서 실행하고 단일 CockroachDB 노드를 실행하는 도커 컨테이너 하나를 포함합니다. 3포드로 시작하여 4포드로 늘어납니다.
[StatefulSet](http://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/) | StatefulSet은 상태 저장 장치로 취급되는 포드의 그룹이며, 각 포드는 구별할 수 있는 네트워크 아이덴티티를 가지며, 재시작 시 항상 동일한 영구 저장소에 다시 바인딩된다. StatefulSet은 버전 1.5에서 베타 버전에 도달 한 후, Kubernetes 버전 1.9에서 안정적인 것으로 간주됩니다.
[영구 볼륨](http://kubernetes.io/docs/user-guide/persistent-volumes/) | 영구 볼륨은 포드에 마운트된 네트워크 저장소 (GCE의 영구 디스크, AWS의 탄성 블록 저장소)의 한 조각입니다. 영구 볼륨의 수명은 이를 사용하는 포드의 수명에서 분리되어, 각 CockroachDB 노드가 재시작시 동일한 저장소에 다시 바인드되도록 보장합니다.<br><br>이 튜토리얼에서는 동적 볼륨 공급을 사용할 수 있다고 가정합니다. 그런 경우가 아니라면, [영구 볼륨 소유권 주장](http://kubernetes.io/docs/user-guide/persistent-volumes/#persistentvolumeclaims)이 수동으로 생성되어야 합니다.
[CSR](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/) | CSR 또는 인증서 서명 요청은 Kubernetes 클러스터의 빌트인 CA가 서명한 TLS 인증서를 요청합니다. 각 포드가 생성될 때, 포드에서 실행되는 CockroachDB 노드에 대한 CSR을 발급하며, 수동으로 확인하고 승인해야 한다.
[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) | RBAC 또는 역할 기반 접근 제어는 Kubernetes가 클러스터 내에서 권한을 관리하는 데 사용하는 시스템입니다. API 자원 (예를 들어, `pod` 또는 `CSR`) 상에서 동작 (예를 들어, `get` 또는 `create`)을 취하기 위해서, 클라이언트는 그렇게 할 수 있는 `Role`을 가져야만 합니다. 이 튜토리얼에서는 CockroachDB가 인증서를 만들고 접근하는 데 필요한 RBAC 자원을 생성합니다.

### 제한 사항

{% include {{ page.version.version }}/orchestration/kubernetes-limitations.md %}

## 1단계. Kubernetes 시작

{% include {{ page.version.version }}/orchestration/start-kubernetes.md %}

## 2단계. CockroachDB 노드 시작

{% include {{ page.version.version }}/orchestration/start-cluster.md %}

## 3단계. 노드 인증서 승인
 
각 포드가 작성되면, Kubernetes CA가 노드의 인증서를 서명하도록 CSR (Certificate Signing Request)을 발행합니다. 각 노드의 인증서를 수동으로 확인하고 승인해야 하며, 여기서 CockroachDB 노드가 포드에서 시작되어야 한다.

1. 포드 1에 대한 `Pending` CSR의 이름 가져오기:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get csr
    ~~~

    ~~~
    NAME                                                   AGE       REQUESTOR                               CONDITION
    default.node.cockroachdb-0                             1m        system:serviceaccount:default:default   Pending
    node-csr-0Xmb4UTVAWMEnUeGbW4KX1oL4XV_LADpkwjrPtQjlZ4   4m        kubelet                                 Approved,Issued
    node-csr-NiN8oDsLhxn0uwLTWa0RWpMUgJYnwcFxB984mwjjYsY   4m        kubelet                                 Approved,Issued
    node-csr-aU78SxyU69pDK57aj6txnevr7X-8M3XgX9mTK0Hso6o   5m        kubelet                                 Approved,Issued
    ~~~

    `Pending` CSR이 표시되지 않으면, 잠시 후 다시 시도하십시오.

2. 포드 1에 대한 CSR을 검사하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl describe csr default.node.cockroachdb-0
    ~~~

    ~~~
    Name:               default.node.cockroachdb-0
    Labels:             <none>
    Annotations:        <none>
    CreationTimestamp:  Thu, 09 Nov 2017 13:39:37 -0500
    Requesting User:    system:serviceaccount:default:default
    Status:             Pending
    Subject:
      Common Name:    node
      Serial Number:
      Organization:   Cockroach
    Subject Alternative Names:
             DNS Names:     localhost
                            cockroachdb-0.cockroachdb.default.svc.cluster.local
                            cockroachdb-public
             IP Addresses:  127.0.0.1
                            10.48.1.6
    Events:  <none>
    ~~~

3. 모든 것이 올바르게 보이면, 포드 1의 CSR을 승인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl certificate approve default.node.cockroachdb-0
    ~~~

    ~~~
    certificatesigningrequest "default.node.cockroachdb-0" approved
    ~~~

4. 다른 2개의 포드에 대해 1-3 단계를 반복하십시오.

## 4단계. 클러스터 초기화

1. 3개의 포드가 성공적으로 `Running` 상태인지 확인하십시오. 클러스터가 초기화 될 때까지는 `Ready`로 간주되지 않습니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    ~~~
    NAME            READY     STATUS    RESTARTS   AGE
    cockroachdb-0   0/1       Running   0          2m
    cockroachdb-1   0/1       Running   0          2m
    cockroachdb-2   0/1       Running   0          2m
    ~~~

2. 영구 볼륨 및 해당 소유권 주장이 세 포드 모두에 대해 성공적으로 생성되었는지 확인합니다:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get persistentvolumes
    ~~~

    ~~~
    NAME                                       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                           REASON    AGE
    pvc-52f51ecf-8bd5-11e6-a4f4-42010a800002   1Gi        RWO           Delete          Bound     default/datadir-cockroachdb-0             26s
    pvc-52fd3a39-8bd5-11e6-a4f4-42010a800002   1Gi        RWO           Delete          Bound     default/datadir-cockroachdb-1             27s
    pvc-5315efda-8bd5-11e6-a4f4-42010a800002   1Gi        RWO           Delete          Bound     default/datadir-cockroachdb-2             27s
    ~~~

3. 우리의 [`cluster-init-secure.yaml`](https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/cluster-init-secure.yaml) 파일을 사용하여 노드를 단일 클러스터로 결합하는 일회성 초기화를 수행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/cluster-init-secure.yaml
    ~~~

    ~~~
    job "cluster-init-secure" created
    ~~~

4. 클러스터 초기화가 발생하는 일회용 포드에 대해 CSR을 승인합니다:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl certificate approve default.client.root
    ~~~

    ~~~
    certificatesigningrequest "default.client.root" approved
    ~~~

5. 클러스터 초기화가 성공적으로 완료되었는지 확인하십시오. 작업은 성공한 것으로 간주되어야 하며 CockroachDB 포드는 곧 `Ready`으로 간주되어야 합니다:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get job cluster-init-secure
    ~~~

    ~~~
    NAME                  DESIRED   SUCCESSFUL   AGE
    cluster-init-secure   1         1            2m
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    ~~~
    NAME            READY     STATUS    RESTARTS   AGE
    cockroachdb-0   1/1       Running   0          3m
    cockroachdb-1   1/1       Running   0          3m
    cockroachdb-2   1/1       Running   0          3m
    ~~~

{{site.data.alerts.callout_success}}StatefulSet 설정은 <code>stderr</code>로 쓰도록 모든 CockroachDB 노드를 설정하므로, 문제 해결을 위해 포드/노드의 로그에 접근해야하는 경우, 영구 볼륨의 로그를 확인하는 대신 <code>kubectl logs &lt;podname&gt;</code>를 사용하십시오.{{site.data.alerts.end}}

## 5단계. 빌트인 SQL 클라이언트 사용

빌트인 SQL 클라이언트를 사용하려면, 내부 `cockroach` 바이너리를 사용하여 무한정 실행되는 포드를 실행하고, 포드에 대한 CSR을 확인 및 승인하고, 포드에 쉘을 넣은 다음, 빌트인 SQL 클라이언트를 시작하십시오.

1. 로컬 워크 스테이션에서, 우리의 [`client-secure.yaml`](https://github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/client-secure.yaml) 파일을 사용하여 창을 시작하고 무한정 계속 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/client-secure.yaml
    ~~~

    ~~~
    pod "cockroachdb-client-secure" created
    ~~~

    포드는 앞에서 생성한 `root` 클라이언트 인증서를 사용하여 클러스터를 초기화하므로, CSR 승인이 필요하지 않습니다.

2. 포드로 쉘을 가져와 CockroachDB [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl exec -it cockroachdb-client-secure -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
    ~~~

    ~~~
    # Welcome to the cockroach SQL interface.
    # All statements must be terminated by a semicolon.
    # To exit: CTRL + D.
    #
    # Server version: CockroachDB CCL v1.1.2 (linux amd64, built 2017/11/02 19:32:03, go1.8.3) (same version as client)
    # Cluster ID: 3292fe08-939f-4638-b8dd-848074611dba
    #
    # Enter \? for a brief introduction.
    #
    root@cockroachdb-public:26257/>
    ~~~

3. 몇 가지 기본 [CockroachDB SQL 명령문](learn-cockroachdb-sql.html)을 실행하십시오:

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

3. [암호가 있는 사용자를 생성](create-user.html#create-a-user-with-a-password)하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE USER roach WITH PASSWORD 'Q7gc8rEdS';
    ~~~

      6단계에서 Admin UI에 접근하려면 이 사용자이름과 비밀번호가 필요합니다.

4. SQL 쉘 및 포드를 종료하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > \q
    ~~~

{{site.data.alerts.callout_success}}이 포드는 무기한으로 계속 실행되므로, 빌트인 SQL 클라이언트를 다시 열거나 다른 <a href="cockroach-commands.html"><code>cockroach</code> 클라이언트 명령어</a>를 실행해야 할 때마다 (e.g., <code>cockroach 노드</code>), 적절한 <code>cockroach</code>를 사용하여 2단계를 반복하십시오.<br></br> 포드를 삭제하고 필요에 따라 재생성하려면, <code>kubectl delete pod cockroachdb-client-secure</code>를 실행하십시오.{{site.data.alerts.end}}


## 6단계. Admin UI 접근

{% include {{ page.version.version }}/orchestration/monitor-cluster.md %}

## 7단계. 노드 오류 시뮬레이션

{% include {{ page.version.version }}/orchestration/kubernetes-simulate-failure.md %}

## 8단계. 모니터링 및 경고 설정

{% include {{ page.version.version }}/orchestration/kubernetes-prometheus-alertmanager.md %}

## 9단계. 클러스터 유지 보수

### 클러스터 크기 조정

{% include {{ page.version.version }}/orchestration/kubernetes-scale-cluster.md %}

3. 새 포드에 대한 'Pending' CSR의 이름을 가져오십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get csr
    ~~~

    ~~~
    NAME                                                   AGE       REQUESTOR                               CONDITION
    default.client.root                                    1h        system:serviceaccount:default:default   Approved,Issued
    default.node.cockroachdb-0                             1h        system:serviceaccount:default:default   Approved,Issued
    default.node.cockroachdb-1                             1h        system:serviceaccount:default:default   Approved,Issued
    default.node.cockroachdb-2                             1h        system:serviceaccount:default:default   Approved,Issued
    default.node.cockroachdb-3                             2m        system:serviceaccount:default:default   Pending
    node-csr-0Xmb4UTVAWMEnUeGbW4KX1oL4XV_LADpkwjrPtQjlZ4   1h        kubelet                                 Approved,Issued
    node-csr-NiN8oDsLhxn0uwLTWa0RWpMUgJYnwcFxB984mwjjYsY   1h        kubelet                                 Approved,Issued
    node-csr-aU78SxyU69pDK57aj6txnevr7X-8M3XgX9mTK0Hso6o   1h        kubelet                                 Approved,Issued
    ~~~

    `Pending` CSR이 표시되지 않으면, 잠시 후 다시 시도하십시오.

4. 새 포드에 대한 CSR을 검토하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl describe csr default.node.cockroachdb-3
    ~~~

    ~~~
    Name:               default.node.cockroachdb-0
    Labels:             <none>
    Annotations:        <none>
    CreationTimestamp:  Thu, 09 Nov 2017 13:39:37 -0500
    Requesting User:    system:serviceaccount:default:default
    Status:             Pending
    Subject:
      Common Name:    node
      Serial Number:
      Organization:   Cockroach
    Subject Alternative Names:
             DNS Names:     localhost
                            cockroachdb-0.cockroachdb.default.svc.cluster.local
                            cockroachdb-public
             IP Addresses:  127.0.0.1
                            10.48.1.6
    Events:  <none>
    ~~~

5. 모든 것이 올바르게 보이면, CSR에 새 포드를 승인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl certificate approve default.node.cockroachdb-3
    ~~~

    ~~~
    certificatesigningrequest "default.node.cockroachdb-3" approved
    ~~~

6. 새 포드를 성공적으로 시작했는지 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    ~~~
    NAME                        READY     STATUS    RESTARTS   AGE
    cockroachdb-0               1/1       Running   0          51m
    cockroachdb-1               1/1       Running   0          47m
    cockroachdb-2               1/1       Running   0          3m
    cockroachdb-3               1/1       Running   0          1m
    cockroachdb-client-secure   1/1       Running   0          15m
    ~~~

8. Admin UI로 돌아가서, **Node List**을 보고 네 번째 노드가 클러스터에 성공적으로 조인했는지 확인하십시오.

### 클러스터 업데이트

{% include {{ page.version.version }}/orchestration/kubernetes-upgrade-cluster.md %}

### 클러스터 종료

CockroachDB 클러스터를 종료하려면 다음을 수행하십시오:

1. 로그, 원격 영구 볼륨, 프로메테우스 및 Alertmanager 자원을 포함하여, `cockroachdb` 레이블과 관련된 모든 자원을 삭제하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl delete pods,statefulsets,services,persistentvolumeclaims,persistentvolumes,poddisruptionbudget,jobs,rolebinding,clusterrolebinding,role,clusterrole,serviceaccount,alertmanager,prometheus,prometheusrule,serviceMonitor -l app=cockroachdb
    ~~~

    ~~~
    pod "cockroachdb-0" deleted
    pod "cockroachdb-1" deleted
    pod "cockroachdb-2" deleted
    service "alertmanager-cockroachdb" deleted
    service "cockroachdb" deleted
    service "cockroachdb-public" deleted
    persistentvolumeclaim "datadir-cockroachdb-0" deleted
    persistentvolumeclaim "datadir-cockroachdb-1" deleted
    persistentvolumeclaim "datadir-cockroachdb-2" deleted
    poddisruptionbudget "cockroachdb-budget" deleted
    job "cluster-init-secure" deleted
    rolebinding "cockroachdb" deleted
    clusterrolebinding "cockroachdb" deleted
    clusterrolebinding "prometheus" deleted
    role "cockroachdb" deleted
    clusterrole "cockroachdb" deleted
    clusterrole "prometheus" deleted
    serviceaccount "cockroachdb" deleted
    serviceaccount "prometheus" deleted
    alertmanager "cockroachdb" deleted
    prometheus "cockroachdb" deleted
    prometheusrule "prometheus-cockroachdb-rules" deleted
    servicemonitor "cockroachdb" deleted
    ~~~

2. 이전에 하지 않았다면, `cockroach` 클라이언트 명령어 용으로 만든 포드를 삭제하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl delete pod cockroachdb-client-secure
    ~~~

    ~~~
    pod "cockroachdb-client-secure" deleted
    ~~~

3. 클러스터에 대한 CSR의 이름을 얻으십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get csr
    ~~~

    ~~~
    NAME                                                   AGE       REQUESTOR                               CONDITION
    default.client.root                                    1h        system:serviceaccount:default:default   Approved,Issued
    default.node.cockroachdb-0                             1h        system:serviceaccount:default:default   Approved,Issued
    default.node.cockroachdb-1                             1h        system:serviceaccount:default:default   Approved,Issued
    default.node.cockroachdb-2                             1h        system:serviceaccount:default:default   Approved,Issued
    default.node.cockroachdb-3                             12m       system:serviceaccount:default:default   Approved,Issued
    node-csr-0Xmb4UTVAWMEnUeGbW4KX1oL4XV_LADpkwjrPtQjlZ4   1h        kubelet                                 Approved,Issued
    node-csr-NiN8oDsLhxn0uwLTWa0RWpMUgJYnwcFxB984mwjjYsY   1h        kubelet                                 Approved,Issued
    node-csr-aU78SxyU69pDK57aj6txnevr7X-8M3XgX9mTK0Hso6o   1h        kubelet                                 Approved,Issued
    ~~~

4. 생성한 CSR를 삭제하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl delete csr default.client.root default.node.cockroachdb-0 default.node.cockroachdb-1 default.node.cockroachdb-2 default.node.cockroachdb-3
    ~~~

    ~~~
    certificatesigningrequest "default.client.root" deleted
    certificatesigningrequest "default.node.cockroachdb-0" deleted
    certificatesigningrequest "default.node.cockroachdb-1" deleted
    certificatesigningrequest "default.node.cockroachdb-2" deleted
    certificatesigningrequest "default.node.cockroachdb-3" deleted
    ~~~

5. 클러스터에 대한 비밀정보의 이름을 얻으십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get secrets
    ~~~

    ~~~
    NAME                         TYPE                                  DATA      AGE
    alertmanager-cockroachdb          Opaque                                1         1h
    default-token-d9gff               kubernetes.io/service-account-token   3         5h
    default.client.root               Opaque                                2         5h
    default.node.cockroachdb-0        Opaque                                2         5h
    default.node.cockroachdb-1        Opaque                                2         5h
    default.node.cockroachdb-2        Opaque                                2         5h
    default.node.cockroachdb-3        Opaque                                2         5h
    prometheus-operator-token-bpdv8   kubernetes.io/service-account-token   3         3h
    ~~~

6. 생성한 비밀정보를 삭제하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl delete secrets alertmanager-cockroachdb default.client.root default.node.cockroachdb-0 default.node.cockroachdb-1 default.node.cockroachdb-2 default.node.cockroachdb-3
    ~~~

    ~~~
    secret "alertmanager-cockroachdb" deleted
    secret "default.client.root" deleted
    secret "default.node.cockroachdb-0" deleted
    secret "default.node.cockroachdb-1" deleted
    secret "default.node.cockroachdb-2" deleted
    secret "default.node.cockroachdb-3" deleted
    ~~~

7. Kubernetes를 중지하십시오:

{% include {{ page.version.version }}/orchestration/stop-kubernetes.md %}

## 더 보기

- [Kubernetes 다중 클러스터 배포](orchestrate-cockroachdb-with-kubernetes-multi-cluster.html)
- [Kubernetes 성능 가이드](kubernetes-performance.html)
{% include {{ page.version.version }}/prod-deployment/prod-see-also.md %}
