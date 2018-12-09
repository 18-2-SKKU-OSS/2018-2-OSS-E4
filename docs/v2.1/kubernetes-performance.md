---
title: CockroachDB Performance on Kubernetes
summary: How running CockroachDB in Kubernetes affects its performance and how to get the best possible performance when running in Kubernetes.
toc: true
---

Kubernetes는 분산 시스템 배포 및 운영에 대한 많은 유용한 추상화를 제공하지만, 일부 추상화는 성능 오버헤드와 기본 시스템의 복잡성 증가를 수반합니다. 이 페이지에서는 [Kubernetes에서 CockroachDB를 실행](orchestrate-cockroachdb-with-kubernetes.html)할 때 인지해야 할 잠재적인 병목 현상을 설명하고 더 나은 성능을 위해 배포를 최적화하는 방법을 보여줍니다.

<div id="toc"></div>

## 전제 조건

Kubernetes-orchestrated CockroachDB 클러스터 최적화를 진행하기 전에 다음을 수행하십시오:

1. [Kubernetes에서 CockroachDB 클러스터를 실행](orchestrate-cockroachdb-with-kubernetes.html)하기 위한 문서를 살펴보고 필요한 Kubernetes 용어와 배포 추상화에 익숙해지십시오.
2. CockroachDB가 Kubernetes가 없는 동일한 하드웨어에서도 작업량(workload)에 대한 요구 사항을 수행할 수 있는지 확인하십시오. 필요한 성능을 얻기 위해 [작업량을 수정](performance-best-practices-overview.html)하거나 [다른 시스템 사양](recommended-production-settings.html#hardware)을 사용해야 할 수도 있으며, Kubernetes 배포를 최적화하기 위해 많은 시간을 소비하는 것보다 먼저 이를 결정하는 것이 더 좋습니다.

## 성능 요소

Kubernetes에서 CockroachDB를 실행할 때 관찰 중인 성능에 영향을 미치는 많은 독립 요인이 있습니다. 어떤 것들은 다른 것들보다 중요하거나 수정하기가 쉽기 때문에 상황에 가장 적합한 개선 사항을 선택하십시오. 이러한 변경 사항의 대부분은 CockroachDB 클러스터를 생성하기 전에 수행하는 것이 가장 쉽습니다. Kubernetes에서 이미 실행 중인 상태의 CockroachDB 클러스터를 수정해야 하는 경우 추가 작업이 필요할 수 있으므로 추가적인 주의 및 테스트를 적극 권장합니다.

아래의 여러 섹션에서, 제공된 Kubernetes 구성 YAML 파일의 발췌 부분을 수정하는 방법을 살펴 보았었습니다. 이 파일들의 최신 버전은 Github에서 찾을 수 있으며, 한 가지 방법은 [CockroachDB를 시큐어 모드에서 실행](https://github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/cockroachdb-statefulset-secure.yaml)하는 것이고 다른 방법은 [CockroachDB를 인시큐어 모드에서 실행](https://github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/cockroachdb-statefulset.yaml)하는 것입니다.

또한 [시큐어 모드](https://github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/performance/cockroachdb-statefulset-secure.yaml) 또는 [인시큐어 모드에 성능이 최적화된 구성 파일](https://github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/performance/cockroachdb-statefulset-insecure.yaml)을 사용할 수도 있습니다. `TODO` 코멘트가 있는 곳이라면 반드시 파일을 수정하십시오.

### CockroachDB 버전

CockroachDB는 매우 활발한 개발 중에 있기 때문에 일반적으로 매 버전마다 상당한 성능 향상이 있습니다. 최신이 아닌 버전에서 원하는 성능을 얻지 못했다면 최신 버전을 사용해 보고 얼마나 도움이 되는지 확인해 보십시오.

### 클라이언트 작업량(workload)

작업량(workload)은 데이터베이스 성능에서 가장 중요한 요소 중 하나입니다. [SQL 성능 모범 사례](performance-best-practices-overview.html)를 읽고 애플리케이션 속도를 높이기 위해 쉽게 변경할 수 있는 사항이 있는지 확인하십시오.

### 시스템 사양

현재 사용 중인 시스템의 사양은 Kubernetes에만 해당되는 것은 아니며, 성능을 향상시키려면 언제나 먼저 고려해야 하는 부분입니다. 구체적인 방법에 대해서는 [하드웨어 권장 사항](recommended-production-settings.html#hardware)을 참조하십시오. 그러나 더 많은 CPU가 있는 시스템을 사용하면 거의 항상 처리량이 향상됩니다. Kubernetes는 클러스터의 모든 시스템에서 일련의 프로세스를 실행하기 때문에 많고 작은 시스템을 사용하는 것보다 적고 큰 시스템을 사용함으로써 더 많은 이점을 얻을 수 있습니다.

### 디스크 유형

CockroachDB는 사용자가 제공하는 디스크를 많이 사용하므로 더 빠른 디스크를 사용하면 클러스터의 성능을 향상시킬 수 있습니다. 제공되는 구성에서는 원하는 디스크 유형을 지정하지 않으므로 대부분의 환경에서 Kubernetes는 기본 유형의 디스크를 자동으로 제공합니다. 일반적인 클라우드 환경(AWS, GCP, Azure)에서는 데이터베이스 작업량에 최적화되지 않은 느린 디스크(예: GCE의 HDD, AWS에서 IOPS를 제공하지 않은 SSD)를 얻을 수 있습니다. 그러나 최상의 성능을 위해서는 [SSD를 사용할 것을 적극 권장](recommended-production-settings.html#hardware)하며, Kubernetes는 SSD를 비교적 쉽게 사용할 수 있도록 합니다.

#### 다른 디스크 유형 생성

Kubernetes는 볼륨 제공자가 사용하는 디스크 유형을 [`StorageClass` API 객체](https://kubernetes.io/docs/concepts/storage/storage-classes/)를 통해 나타냅니다. 각 클라우드 환경에는 고유의 기본 `StorageClass`가 있지만, 기본값을 쉽게 변경하거나 볼륨이 필요할 떄 요청할 수 있는 새로운 클래스를 만들 수 있습니다. 이렇게 하려면 [Kubernetes 문서](https://kubernetes.io/docs/concepts/storage/storage-classes/)의 목록에서 사용할 볼륨 제공자 유형을 선택하고, 해당 유형이 제공하는 YAML 파일 예제를 가져와서 원하는 디스크 유형으로 수정한 다음, `kubectl create -f <your- storage-class-file.yaml>`을 실행하십시오. 예를 들어, Google Compute Engine 또는 Google Kubernetes Engine에서 `pd-ssd` 디스크 유형을 사용하려면 다음과 같이 `StorageClass` 파일을 사용할 수 있습니다:

~~~ yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <your-ssd-class-name>
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
~~~

그런 다음 이러한 새로운 디스크 유형을 사용하려면 CockroachDB YAML 파일이 이를 요청하도록 구성하거나 이를 기본값으로 설정하십시오. 또한 Kubernetes 저장소 클래스 목록에 기록된 대로 추가 매개 변수를 설정할 수도 있습니다; 예를 들어 AWS의 `io1` Provisioned IOPS 볼륨 유형을 위한 `StorageClass`를 생성하는 경우 `iopsPerGB`를 구성합니다.

#### CockroachDB에서 사용되는 디스크 유형 구성

새로운 `StorageClass`을 클러스터의 기본값으로 만들지 않고 사용하려면 애플리케이션의 YAML 파일을 수정해야 합니다. CockroachDB `StatefulSet` 구성에서, 이것은 `VolumeClaimTemplates` 섹션에 라인을 추가하는 것을 의미합니다. 예를 들어, CockroachDB config 파일에서 다음 라인들을 수정하는 것을 의미합니다:

~~~ yaml
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes:
        - "ReadWriteOnce"
      resources:
        requests:
          storage: 1Gi
~~~

그리고 `spec`에 `storageClassName` 필드를 추가하고 그것들을 다음과 같이 변경합니다:

~~~ yaml
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes:
        - "ReadWriteOnce"
      storageClassName: <your-ssd-class-name>
      resources:
        requests:
          storage: 1Gi
~~~

이러한 변경 이후 YAML 파일에서 `kubectl create -f`를 실행하면, Kubernetes는 새로운 `StorageClass`를 사용하여 볼륨을 생성해야 합니다.

#### 기본 디스크 유형 변경

새로운 `StorageClass`를 클러스터의 모든 볼륨에 대한 기본값으로 사용하려면 몇 가지 명령을 실행하여 Kubernetes에 원하는 내용을 알려줘야 합니다. 먼저 새로운 `StorageClass`의 이름을 얻으십시오. 그런 다음 현재 기본값을 제거하고 새 기본값으로 추가하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ kubectl get storageclasses
~~~

~~~
NAME                 PROVISIONER
ssd                  kubernetes.io/gce-pd
standard (default)   kubernetes.io/gce-pd
~~~

{% include copy-clipboard.html %}
~~~ shell
$ kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
~~~

~~~
storageclass "standard" patched
~~~

{% include copy-clipboard.html %}
~~~ shell
$ kubectl patch storageclass ssd -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
~~~

~~~
storageclass "ssd" patched
~~~

이전 버전의 Kubernetes를 사용하는 경우 위에 사용된 양식 대신에 베타 버전의 주석을 사용해야 할 수도 있습니다. 특히 Kubernetes v1.8에서는`storageclass.beta.kubernetes.io/is-default-class`를 사용해야 합니다. 어떤 것을 사용할지 확실히 결정하려면 `kubectl describe storageclass`를 실행하고 현재 기본값에서 사용한 주석을 복사하십시오.

### 디스크 용량

일부 클라우드 제공자(특히 모든 GCP 디스크와 AWS io1 디스크 유형 포함)에서 디스크에 사용할 수 있는 IOPS 수는 디스크 크기와 직접적인 상관 관계가 있습니다. 이러한 경우 디스크 크기를 늘리면 CockroachDB의 성능이 크게 향상될 뿐만 아니라 디스크 공간이 가득 찰 위험이 줄어 듭니다. 방법은 매우 간단합니다 -- CockroachDB 클러스터를 만들기 전에 CockroachDB YAML 파일의 `VolumeClaimTemplate`을 수정하여 더 많은 공간을 확보하십시오. 예를 들어 각 CockroachDB 인스턴스에 1TB의 디스크 공간을 할당하려면 다음과 같이 수정하십시오:

~~~ yaml
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes:
        - "ReadWriteOnce"
      resources:
        requests:
          storage: 1Gi
~~~

그 대신에:

~~~ yaml
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes:
        - "ReadWriteOnce"
      resources:
        requests:
          storage: 1024Gi
~~~

[GCE 디스크 IOPS는 디스크 크기에 따라 선형적으로 확장](https://cloud.google.com/compute/docs/disks/performance#type_comparison)되기 때문에, 1TiB 디스크가 1GiB 디스크보다 1024배 더 많은 IOPS를 제공한다는 사실은 쓰기 횟수가 많은(write-heavy) 작업량의 경우 큰 차이를 만듭니다.

### 로컬 디스크

이제까지 우리는 원격으로 연결된 자동 제공 디스크를 사용하여 `StatefulSet`에서 CockroachDB를 실행한다고 가정했습니다. 그러나 로컬 디스크를 사용하면 일반적으로 원격으로 연결된 디스크보다 성능이 향상됩니다; 예를 들어 AWS의 EBS 볼륨 대신 SSD 인스턴스 저장소 볼륨을 사용하거나 GCE의 영구 디스크 대신 로컬 SSD를 사용합니다. `StatefulSet`은 역사적으로 로컬 디스크를 사용을 지원하지 않았지만 [Kubernetes v1.10에서는 "local" `PersistentVolume`을 사용하기 위한 베타 지원이 추가](https://kubernetes.io/docs/concepts/storage/volumes/#local)되었습니다. 기능이 더 완성되기 전까지 배포 데이터 용으로 사용하지 않는 것을 권장하지만, 유망한 개발임은 틀림없습니다.

또한 `StatefulSet` 대신 `DaemonSet`에서 CockroachDB를 실행할 경우 로컬 디스크를 사용하는 옵션도 있습니다. 이 기능에 대한 자세한 내용은 [DaemonSet에서 실행](#running-in-a-daemonset) 섹션을 참조하십시오.

로컬 디스크를 사용하여 실행하는 경우, 종종 커버 아래에 복제되는 클라우드 제공자의 네트워크 연결 디스크를 사용할 때보다 디스크 장애가 발생할 확률이 더 높습니다. 따라서 로컬 디스크를 사용할 때 데이터의 복제 팩터를 기본값인 3에서 5로 늘리도록 [복제 영역을 구성](configure-replication-zones.html)할 수 있습니다.

### 리소스 요청 및 제한

Kubernetes에게 `StatefulSet`과 같은 다른 자원 유형을 통해 직접 또는 간접적으로 포드를 실행하도록 요청하면, 포드의 각 컨테이너에 일정량의 CPU 및/또는 메모리를 예약하거나 각 컨테이너의 CPU 및/또는 메모리를 제한하도록 지시할 수 있습니다. 이 중 하나 또는 둘 다를 수행하는 것은 Kubernetes 클러스터가 어떻게 활용되는지에 따라 다른 의미를 가질 수 있습니다. 이 항목에 대한 자세한 내용은 [Kubernetes 문서](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)를 참조하십시오.

#### 리소스 요청

리소스 요청을 통해 컨테이너에 일정량의 CPU 또는 메모리를 예약할 수 있습니다. CockroachDB YAML 파일에 리소스 요청을 추가하면 Kubernetes는 예약되지 않은 리소스가 충분히 있는 노드에 각 CockroachDB 포드를 예정하고 해당 Linux 컨테이너 기본 요소(primitive)를 사용하여 포드가 예약된 리소스를 보장하는지 확인합니다. Kubernetes 클러스터에서 다른 작업량을 실행 중인 경우 우수한 성능을 위해 리소스 요청을 설정하는 것이 매우 권장됩니다. 설정하지 않으면 CockroachDB가 덜 중요한 프로세스 전에 CPU 사이클이 부족하거나 OOM이 종료될 수 있습니다.

Kubernetes 노드에서 사용할 수 있는 리소스 개수를 확인하려면 다음을 실행하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ kubectl describe nodes
~~~

~~~
Name:               gke-perf-default-pool-aafee20c-k4t8
[...]
Capacity:
 cpu:     4
 memory:  15393536Ki
 pods:    110
Allocatable:
 cpu:     3920m
 memory:  12694272Ki
 pods:    110
[...]
Non-terminated Pods:         (2 in total)
  Namespace                  Name                                              CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------                  ----                                              ------------  ----------  ---------------  -------------
  kube-system                kube-dns-778977457c-kqtlr                         260m (6%)     0 (0%)      110Mi (0%)       170Mi (1%)
  kube-system                kube-proxy-gke-perf-default-pool-aafee20c-k4t8    100m (2%)     0 (0%)      0 (0%)           0 (0%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ------------  ----------  ---------------  -------------
  360m (9%)     0 (0%)      110Mi (0%)       170Mi (1%)
~~~

이렇게 하면 클러스터의 각 노드에 대한 많은 정보가 출력되지만, 오른쪽 부분에 초점을 맞추면 각 노드에서 사용할 수 있는 "할당 가능한" 리소스의 수와 다른 포드에서 이미 사용 중인 리소스의 수를 알 수 있습니다. "할당 가능한" 리소스는 Kubernetes가 시스템에서 실행 중인 포드에 제공하려는 CPU 및 메모리의 양입니다. 노드의 "용량"과 "할당 가능한" 리소스의 차이점은 운영 체제와 Kubernetes의 관리 프로세스에 의해 결정됩니다. "3920m"의 "m"은 "milli-CPUs", 즉 "CPU의 1000분의 1"을 의미합니다.

또한 클러스터에서 인식하지 못한 많은 포드가 실행되고 있음을 볼 수 있습니다. Kubernetes는 클러스터 인프라의 일부인 `kube-system` 네임스페이스에서 소량의 포드를 실행합니다. 이러한 소량의 포드는 CockroachDB를 위한 노드의 모든 할당 가능한 공간을 예약하는 것을 어렵게 할 수 있는데, 이들 중 일부가 Kubernetes 클러스터의 상태에 필수적이기 때문입니다. 클러스터의 모든 노드에서 CockroachDB를 실행하려면 이러한 프로세스를 위한 공간을 남겨 두어야 합니다. 클러스터에 있는 노드의 일부분에서만 CockroachDB를 실행하는 경우, `kube-proxy` 또는`fluentd` 로깅 에이전트와 같은 클러스터의 모든 노드에 있는 `kube-system` 포드에서 사용되는 것 이외의 모든 "할당 가능한" 공간을 차지하도록 선택할 수 있습니다.

CockroachDB가 계속 실행하기를 바라는 노드에 이미 존재하는 `kube-system` 포드를 수동으로 선점해야 하기 때문에 현재 버전의 Kubernetes(v1.10 또는 그 이전 버전)의 할당 가능한 공간을 실제로 모두 사용하는 것은 어렵습니다. 이는 [포드 우선순위](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/) 기능이 알파에서 베타로 승격되는 Kubernetes의 차기 버전에서 쉬워질 것입니다. 이 기능이 더 널리 사용되면, CockroachDB 포드를 더 높은 우선순위로 설정할 수 있으며, 이로 인해 Kubernetes 스케줄러가 다른 시스템에`kube-system` 포드를 선점하고 일정을 재조정할 수 있게 됩니다.

Cockroach를 위해 예약할 CPU와 메모리를 선택한 후에는 CockroachDB YAML 파일에서 리소스 요청을 구성해야 합니다. 그들은 `containers`의 제목 아래에 들어가야 합니다. 예를 들어, 위에서 설명한 시스템에서 사용 가능한 리소스 대부분을 사용하려면 YAML config 파일의 다음 라인을 변경하십시오:

~~~ yaml
      containers:
      - name: cockroachdb
        image: {{page.release_info.docker_image}}:{{page.release_info.version}}
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 26257
          name: grpc
        - containerPort: 8080
          name: http
~~~

다음과 같이 수정하십시오:

~~~ yaml
      containers:
      - name: cockroachdb
        image: {{page.release_info.docker_image}}:{{page.release_info.version}}
        imagePullPolicy: IfNotPresent
        resources:
          requests:
            cpu: "3500m"
            memory: "12300Mi"
        ports:
        - containerPort: 26257
          name: grpc
        - containerPort: 8080
          name: http
~~~

`StatefulSet`을 만들고 모든 CockroachDB 포드가 성공적으로 예약되었는지 확인하고 싶을 것입니다. 보류 상태에서 멈추는 것을 발견하면 `kubectl describe pod <podname>`을 실행하고 `Events`에서 그들이 보류중인 이유에 대한 정보를 확인하십시오. CockroachDB 포드를 위한 공간을 만들기 위해 `kubectl delete pod`를 실행하여 하나 이상의 노드에서 포드를 수동으로 선점해야 할 수도 있습니다. 삭제된 포드가 `Deployment`나 `StatefulSet`과 같은 상위 레벨 Kubernetes 객체에 의해 생성된 것이라면 그것들은 다른 노드에서 안전하게 재생성됩니다.

#### 리소스 제한

리소스 제한은 개념적으로 리소스 요청과 비슷하지만 다른 목적으로 사용됩니다. 리소스 제한은 포드에 의해 사용되는 리소스를 제공된 한도 이하로 제한을 두는데, 이는 몇 가지의 다른 용도로 사용될 수 있습니다. 첫째, 포드는 시스템에서 초과 용량을 사용할 수 없기 때문에 리소스 제한을 사용하면 성능이 더 예측 가능해집니다. 즉, 포드는 때때로(트래픽의 소강 상태 동안) 다른 때보다(기계의 다른 포드들 또한 예약된 리소스를 충분히 활용하고 있는 바쁜 상태) 더 많은 리소스를 사용할 수 없습니다. 둘째, Kubernetes 버전 1.8 이하에서 [Kubernetes 런타임에 의해 보장되는 "서비스 품질"](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/resource-qos.md)을 향상시켜 시스템이 초과할당 되었을 때 포드가 덜 선점되게 합니다. 마지막으로, 메모리 제한은 특히 컨테이너가 사용할 수 있는 메모리 양을 제한하는데, 이는 기본 구성 파일과 마찬가지로 CockroachDB `--cache` 및 `--max-sql-memory` 플래그에 대한 백분율을 지정할 때 도움이 됩니다.

리소스 제한 설정은 리소스 요청 설정과 거의 동일합니다. 위의 [리소스 요청](#resource-requests) 섹션에서 config에 대한 요청 외에도 리소스 제한을 설정하려면, config를 다음과 같이 변경하십시오:

~~~ yaml
      containers:
      - name: cockroachdb
        image: {{page.release_info.docker_image}}:{{page.release_info.version}}
        imagePullPolicy: IfNotPresent
        resources:
          requests:
            cpu: "3500m"
            memory: "12300Mi"
          limits:
            cpu: "3500m"
            memory: "12300Mi"
        ports:
        - containerPort: 26257
          name: grpc
        - containerPort: 8080
          name: http
~~~

포드는 매우 예외적인 경우를 제외하고는 자신이 예약하고 보류하지 않을 것이 보장된 리소스만 사용하도록 제한됩니다. 이는 일반적으로 활용률이 낮은 Kubernetes 클러스터에서 더 나은 성능을 제공하지는 않지만 다른 작업량이 실행될 때 예측 가능한 성능을 제공합니다.

{{site.data.alerts.callout_danger}}메모리 제한을 설정하는 것을 강력히 권장하지만, <a href="https://github.com/kubernetes/kubernetes/issues/51135">CPU 제한을 설정하면 현재 Kubernetes에서 구현해 놓은 지연 시간(tail latencies)이 손상될 수 있습니다</a>. Kubernetes 클러스터를 설정할 때 기본값이 아닌 <a href="https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#static-policy">정적 CPU 관리 정책</a>을 분명하게 활성화하지 않았거나 해당 요청과 정확히 동일한 정수의(소수가 아닌) CPU 제한 및 메모리 제한을 설정하지 않은 경우 CPU 제한을 전혀 설정하지 않는 것이 좋습니다.{{site.data.alerts.end}}

#### 기본 리소스 요청 및 제한

리소스 요청을 수동으로 설정하지 않았어도 자신도 모르게 리소스 요청을 사용할 가능성이 높습니다. 많은 Kubernetes의 설치에서 [`LimitRange`](https://kubernetes.io/docs/tasks/administer-cluster/cpu-default-namespace/)는 `100m`의 기본 CPU 요청, 즉 CPU의 10분의 1을 적용하는 `default` 네임스페이스에 맞게 미리 설정되어 있습니다. 다음을 실행하여 이러한 구성을 볼 수 있습니다:

{% include copy-clipboard.html %}
~~~ shell
$ kubectl describe limitranges
~~~

~~~
Name:       limits
Namespace:  default
Type        Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---  ---------------  -------------  -----------------------
Container   cpu       -    -    100m             -              -
~~~

실험적으로, Kubernetes 클러스터가 많이 활용되지 않을 때 이것이 CockroachDB의 성능에 눈에 띄는 영향을 미치는 것으로 보이지는 않지만, 설정하지 않은 포드에서 CPU 요청을 봐도 놀라지 마십시오.

### CockroachDB와 동일한 시스템의 다른 포드

[리소스 요청 및 제한](#resource-requests-and-limits)에 대한 위의 섹션에서 볼 수 있듯이, 사용자가 직접 다른 포드를 만들지 않아도 Kubernetes 클러스터에서 실행중인 CockroachDB 이외의 포드가 항상 존재합니다. 다음을 실행하여 언제든지 확인할 수 있습니다:

{% include copy-clipboard.html %}
~~~ shell
$ kubectl get pods --all-namespaces
~~~

~~~
NAMESPACE     NAME                                             READY     STATUS    RESTARTS   AGE
kube-system   event-exporter-v0.1.7-5c4d9556cf-6v7lf           2/2       Running   0          2m
kube-system   fluentd-gcp-v2.0.9-6rvmk                         2/2       Running   0          2m
kube-system   fluentd-gcp-v2.0.9-m2xgp                         2/2       Running   0          2m
kube-system   fluentd-gcp-v2.0.9-sfgps                         2/2       Running   0          2m
kube-system   fluentd-gcp-v2.0.9-szwwn                         2/2       Running   0          2m
kube-system   heapster-v1.4.3-968544ffd-5tsb8                  3/3       Running   0          1m
kube-system   kube-dns-778977457c-4s7vv                        3/3       Running   0          1m
kube-system   kube-dns-778977457c-ls6fq                        3/3       Running   0          2m
kube-system   kube-dns-autoscaler-7db47cb9b7-x2cc4             1/1       Running   0          2m
kube-system   kube-proxy-gke-test-default-pool-828d39a7-dbn0   1/1       Running   0          2m
kube-system   kube-proxy-gke-test-default-pool-828d39a7-nr06   1/1       Running   0          2m
kube-system   kube-proxy-gke-test-default-pool-828d39a7-rc4m   1/1       Running   0          2m
kube-system   kube-proxy-gke-test-default-pool-828d39a7-trd1   1/1       Running   0          2m
kube-system   kubernetes-dashboard-768854d6dc-v7ng8            1/1       Running   0          2m
kube-system   l7-default-backend-6497bcdb4d-2kbh4              1/1       Running   0          2m
~~~

이러한 ["클러스터 추가 기능"](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)은 클러스터 내의 서비스에 대한 DNS 항목 관리, Kubernetes 대시보드 UI 전원 공급, 또는 클러스터에서 실행중인 모든 포드의 로그 또는 행렬 수집과 같은 다양한 기본 서비스를 제공합니다. 클러스터 내의 공간을 차지하는 것을 꺼리는 경우 Kubernetes 클러스터를 적절히 구성하여 일부가 실행되지 않도록 할 수 있습니다. 예를 들어, GKE에서 다음을 실행하여 최소한의 추가 기능 세트로 클러스터를 생성할 수 있습니다:

{% include copy-clipboard.html %}
~~~ shell
$ gcloud container clusters create <your-cluster-name> --no-enable-cloud-logging --no-enable-cloud-monitoring --addons=""
~~~

그러나 `kube-proxy`와 `kube-dns`와 같은 필수 요소는 Kubernetes 클러스터를 따르기 위해 효과적으로 요구됩니다. 즉, 사용자의 클러스터에서 실행되지 않는 일부 포드가 항상 있을 수 있으므로, CockroachDB가 다른 프로세스와 시스템을 공유해야 할 때 발생할 수 있는 영향을 이해하고 설명하는 것이 중요합니다. CockroachDB 포드와 동일한 시스템에 더 많은 프로세스가 존재할수록 그 성능은 더 나빠지고 예측 가능성은 떨어질 것입니다. 이로부터 보호하기 위해서 CockroachDB 포드에서 [리소스 요청](#resource-requests)을 실행하여 어느 수준의 CPU 및 메모리 격리를 제공하는 것을 강력히 권장합니다.

하지만 리소스 요청 설정은 만병 통치약이 아닙니다. 네트워크 I/O 또는 [예외적인](https://sysdig.com/blog/container-isolation-gone-wrong/) [경우](https://hackernoon.com/another-reason-why-your-docker-containers-may-be-slow-d37207dec27f), 내부 커널 데이터 구조와 같은 공유 리소스에 대해서는 여전히 경쟁(contention)이 있을 수 있습니다. 이러한 이유와 더불어 각 시스템에서 실행되는 Kubernetes 인프라 프로세스로 인해, Kubernetes에서 실행되는 CockroachDB는 전용 시스템에서 직접 실행하는 것과 동일한 수준의 성능을 달성할 수 없습니다. 다행히도 Kubernetes를 현명하게 사용하면 꽤 가까워질 수 있습니다.

어떤 이유로 적절한 리소스 요청을 설정해도 여전히 예상되는 성능을 얻지 못하면, [전용 노드](#dedicated-nodes)로 가는 것이 좋습니다.

#### CockroachDB와 동일한 시스템의 클라이언트 애플리케이션

CockroachDB와 동일한 시스템에서 클라이언트 애플리케이션(예: 벤치마킹 애플리케이션)을 실행하는 것은 동일한 시스템에서 Kubernetes 시스템 포드를 사용하는 것보다 훨씬 어려울 수 있습니다. 애플리케이션이 평소보다 많은 로드가 발생하면 CockroachDB 프로세스도 그러하기 때문에, 리소스를 놓고 경쟁하게 될 가능성이 매우 큽니다. 이를 방지하는 가장 좋은 방법은 [리소스 요청과 제한을 설정](#resource-requests-and-limits)하는 것이지만, 어떤 이유로든 원하지 않거나 할 수 없는 경우 클라이언트 애플리케이션에서 [anti-affinity 스케줄링 정책](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)을 설정할 수도 있습니다. anti-affinity 정책은 포드 스펙에 포함되어 있으므로, 제공된 예제 로드  애플리케이션을 변경하려면 [다음 라인](https://github.com/cockroachdb/cockroach/blob/98c506c48f3517d1ac1aadb6a09e1b23ad672c37/cloud/kubernetes/example-app.yaml#L11-L12)을 수정하십시오:

~~~ yaml
    spec:
      containers:
~~~

다음과 같이 수정하십시오:

~~~ yaml
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - loadgen
              topologyKey: kubernetes.io/hostname
          - weight: 99
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - cockroachdb
              topologyKey: kubernetes.io/hostname
      containers:
~~~

이러한 설정은 먼저 서로 다른 노드에 `loadgen` 포드를 두는 것을 선호하는데, 이는 `loadgen` 포드 자체의 고장 허용 범위(fault tolerance)에 중요한 역할을 합니다. 다음 우선 순위로서, 아직 실행중인 `CockroachDB` 포드가 없는 노드에 포드를 배치하려고 합니다. 이렇게 하면 로드 제네레이터 및 CockroachDB 클러스터의 고장 허용 범위와 성능의 최상의 균형을 유지할 수 있습니다.

### 네트워킹

Kubernetes는 클러스터의 각 포드에 라우팅 가능한 IP 주소와 분리된 Linux 네트워크 네임스페이스를 다른 요구 사항과 함께 제공하기 위해 [실행되는 많은 네트워크를 요구](https://kubernetes.io/docs/concepts/cluster-administration/networking/)합니다. 이 문서는 세부 사항을 제대로 설명할 만큼 충분히 많지 않고 세부 내용 자체는 클러스터에 네트워크를 설정한 방식에 크게 좌우될 수 있지만, Docker 및 Kubernetes의 네트워킹 추상화가 종종 CockroachDB와 같은 높은 처리량의 분산 애플리케이션에 대한 성능 저하를 초래한다고만 해도 충분할 것입니다.

클러스터에서 더 많은 성능을 얻길 원한다면 네트워킹은 적어도 실험하기에 좋은 대상입니다. 클러스터의 네트워킹 솔루션을 보다 성능이 뛰어난 것으로 교체하거나 호스트 시스템의 네트워크를 직접 사용하여 대부분의 네트워킹 오버헤드를 우회할 수 있습니다.

#### 네트워킹 솔루션

호스팅된 Kubernetes 서비스를 사용하지 않는 경우, 일반적으로 Kubernetes 클러스터를 만들 때 네트워크를 설정하는 방법을 선택해야 합니다. [많은 솔루션이 있으며](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) 성능 특성과 기능이 크게 다를 수 있습니다. 특별히 어떤 네트워킹 소프트웨어나 구성을 지지하지는 않지만, 사용자의 선택이 Kubernetes 외부에서 CockroachDB를 실행하는 것에 비하여 성능에 의미있는 영향을 미칠 수 있음을 알려드립니다.

#### 호스트 네트워크 사용

이미 클러스터의 네트워킹 설정에 만족하고 있거나 설정에 문제를 일으키고 싶지 않다면, Kubernetes는 네트워크 성능 오버헤드를 피할 수 있는 예외적인 경우를 대비한 탈출구를 제공합니다 -- 호스트 시스템의 네트워크를 사용하여 포드를 직접 실행하고 추상화의 계층을 우회할 수 있는 `hostNetwork` 설정. 물론, 여러 가지 단점이 있습니다. 예를 들어, 같은 시스템에서 `hostNetwork`를 사용하는 두 개의 포드는 동일한 포트를 사용할 수 없으며, 공용 인터넷에서 시스템에 접속할 수 있는 경우 심각한 보안 문제가 발생할 수 있습니다. 그럼에도 불구하고, 이것이 작업량에 어떤 영향을 미치는지 확인하고 싶다면 CockroachDB YAML 구성 파일과 더 나은 성능이 절실히 요구되는 클라이언트 애플리케이션에 다음 두 줄을 추가하십시오:

~~~ yaml
    spec:
      affinity:
~~~

다음과 같이 수정하십시오:

~~~ yaml
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      affinity:
~~~

`hostNetwork: true`는 Kubernetes에게 IP 주소, 호스트 이름 및 전체 네트워킹 스택을 사용하여 호스트 시스템의 네트워크 네임스페이스에 포드를 넣도록 지시합니다. `dnsPolicy: ClusterFirstWithHostNet` 라인은 Kubernetes에게 서비스 검색을 위해 클러스터의 DNS 인프라를 계속 사용할 수 있도록 포드를 구성하도록 지시합니다.

이것은 기적을 일으키지 않으므로 신중히 사용하십시오. 테스트에서 GKE의 3-노드 클러스터에 대해 [`kv` 로드 제네레이터](https://hub.docker.com/r/cockroachdb/loadgen-kv/)를 실행했을 때 데이터베이스 처리량은 약 6% 향상되었습니다.

### DaemonSet에서 실행

지금까지 모든 예제에서 표준 CockroachDB `StatefulSet` 구성 파일을 사용하여 조금씩 수정했습니다. 다른 절충을 할 수 있는 대안은 오케스트레이션을 위해 `StatefulSet`을 사용하는 것에서 `DaemonSet`을 사용하는 것으로 완전히 전환하는 것입니다. [`DaemonSet`](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)은 Kubernetes 유형으로, 일부 선택 기준과 일치하는 모든 노드에서 포드를 실행합니다.

여기에는 몇 가지 주요 이점이 있습니다 -- [전용 노드](#dedicated-nodes)로의 코드 변환에 대한 보다 자연스러운 추상화이며, 이미 노드와 CockroachDB 프로세스를 일대일로 결합하고 있으므로 [호스트의 네트워크를 사용](#using-the-hosts-network)하여 자연스럽게 짝을 이루고, `StatefulSets`에서 로컬 디스크를 사용하기 위한 베타 지원에 의존하지 않고 [로컬 디스크](#local-disks)를 사용하는 것이 가능합니다. 가장 큰 단점은 클러스터를 장애로부터 복구하는 Kubernetes의 기능을 제한한다는 것입니다. 일치하는 모든 노드에서 CockroachDB 포드를 이미 실행하고 있기 때문에 장애가 발생한 노드에서 포드를 대체할 새 포드를 생성할 수 없습니다. 이는 운용자가 수동으로만 교체하는 물리적 시스템 세트에서 직접 CockroachDB를 실행하는 동작과 일치합니다.

CockroachDB `DaemonSet`을 설정하려면 `StatefulSet`보다는 조금 더 많은 작업이 필요합니다. [CockroachDB Github 저장소에서 제공된 `DaemonSet` 구성 파일 템플릿](https://github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/performance/cockroachdb-daemonset-insecure.yaml)을 기반으로 사용할 것입니다.

먼저, CockroachDB를 Kubernetes 클러스터의 모든 시스템에서 실행하지 않으려면 [노드 레이블](#node-labels) 또는 [노드 테인트(taint)](#node-taints)를 사용하여 CockroachDB를 실행할 노드를 선택해야 합니다. 노드를 선택하거나 생성했다면 관련 [전용 노드](#dedicated-nodes)에 설명된 대로 해당 노드와 `DaemonSet` YAML 파일을 적절하게 구성하십시오.

그런 다음 YAML 파일에서 CockroachDB `--join` 플래그에 주소를 설정해야 합니다. 이 파일은 기본적으로 호스트의 네트워크를 사용하므로 호스트 시스템의 IP 주소나 호스트 이름을 결합 주소로 사용해야 합니다. 이 중 제공된 파일에 포함할 두 개 또는 세 개를 선택하여 목록 (`10.128.0.4,10.128.0.5,10.128.0.3`)을 대체합니다. 선택한 시스템이 Kubernetes 클러스터에서 제거되면 `--join` 플래그 값을 업데이트해야 하며, 그렇지 않으면 새로운 CockroachDB 인스턴스가 클러스터에 결합될 수 없습니다.

그런 다음 CockroachDB의 데이터를 저장할 호스트에서 디렉토리를 선택하고 config 파일의 `path: /tmp/cockroach-data` 라인을 원하는 디렉토리로 바꿉니다. 로컬 SSD를 사용하는 경우 SSD가 시스템에 마운트되어 있는 곳이면 어디든 가능합니다.

이러한 단계들을 수행하고 원하는 다른 변경을 모두 수행한 후에는 모든 세트가 `DaemonSet`을 생성하도록 설정해야 합니다:

{% include copy-clipboard.html %}
~~~ shell
$ kubectl create -f cockroachdb-daemonset.yaml
~~~

~~~
daemonset "cockroachdb" created
~~~

클러스터를 초기화하려면 포드 이름 중 하나를 선택하고 다음을 실행하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ kubectl exec -it <pod-name> -- ./cockroach init --insecure
~~~

~~~
Cluster successfully initialized
~~~

### 전용 노드

Kubernetes 클러스터가 이기종(heterogeneous) 하드웨어로 구성된 경우, CockroachDB가 특정 시스템에서만 실행되는지 확인하고 싶을 것입니다. 또한 시스템 세트에서 가능한 한 많은 성능을 얻고 싶다면, CockroachDB를 제외한 그 어떤 프로그램도 시스템에서 실행되지 않기를 확인하고 싶을 것입니다.

#### 노드 레이블

노드 레이블과 노드 선택기(selector)는 Kubernetes에게 포드를 허용할 노드를 알려주는 방법입니다. 노드에 레이블을 지정하려면, 노드 이름과 원하는 키-값 쌍을 레이블에 대체하는 `kubectl label node` 명령을 사용하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ kubectl label node <node-name> <key>=<value>
~~~

일부 Kubernetes 설치 도구를 사용하면 레이블을 특정 노드에 자동으로 적용할 수 있습니다. 예를 들어, 새로운 [GKE 노드 풀(pool)](https://cloud.google.com/kubernetes-engine/docs/concepts/node-pools)을 생성 할 때 `--node-labels` 플래그를 `gcloud container node-pools create` 명령에 사용할 수 있습니다.

원하는 모든 노드에 대해 레이블을 설정하면 [`NodeSelector`를 사용](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector)하여 포드를 예약할 위치를 제어할 수 있습니다. 예를 들어, 위 예제의 `DaemonSet` 파일에서 다음 라인을 변경하십시오:

~~~ yaml
    spec:
      hostNetwork: true
      containers:
~~~

다음과 같이 수정하십시오:

~~~ yaml
    spec:
      nodeSelector:
        <key>: <value>
      hostNetwork: true
      containers:
~~~

#### 노드 테인트(taint)

또는, CockroachDB만이 시스템 세트에서 실행되게 하려면 [`Taints` 및 `Tolerations`](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)라는 상호보완 기능을 사용하여 Kubernetes에게 시스템에서 다른 어떤 것도 계획하지 말라고 지시하는 것이 좋습니다. 노드 레이블과 노드 선택기를 설정하는 방법과 매우 유사한 방식으로 설정할 수 있습니다.

{% include copy-clipboard.html %}
~~~ shell
$ kubectl taint node <node-name> <key>=<value>:NoSchedule
~~~

[노드 레이블](#node-labels)과 마찬가지로, 일부 Kubernetes 설치 도구를 사용하면 특정 노드에 자동으로 테인트(taint)를 적용할 수 있습니다. 예를 들어, 새로운 [GKE 노드 풀(pool)](https://cloud.google.com/kubernetes-engine/docs/concepts/node-pools)을 생성 할 때 `--node-taints` 플래그를 `gcloud container node-pools create` 명령에 사용할 수 있습니다.

CockroachDB만 실행하려는 각 시스템에 적절한 `Taint`를 적용한 후, CockroachDB config 파일에 해당 `Toleration`을 추가하십시오. 예를 들어, 위 예제의 `DaemonSet` 파일에서 다음 라인을 변경하십시오:

~~~ yaml
    spec:
      hostNetwork: true
      containers:
~~~

다음과 같이 수정하십시오:

~~~ yaml
    spec:
      tolerations:
      - key: <key>
        operator: "Equal"
        value: <value>
        effect: "NoSchedule"
      hostNetwork: true
      containers:
~~~

이렇게 하면 CockroachDB가 아닌 포드가 이 시스템에서 실행되는 것을 막을 뿐입니다. CockroachDB가 다른 모든 시스템에서 실행되는 것을 막지는 않으므로, 대부분의 경우 해당 [노드 레이블](#node-labels)과 노드 선택기를 쌍으로 사용하여 진정한 전용 노드를 만들면 다음과 같은 config 파일 조각이 만들어집니다:

~~~ yaml
    spec:
      tolerations:
      - key: <key>
        operator: "Equal"
        value: <value>
        effect: "NoSchedule"
      nodeSelector:
        <key>: <value>
      hostNetwork: true
      containers:
~~~

## 기존 CockroachDB 클러스터 수정

Kubernetes를 사용하면 기존 리소스 구성 중 전부는 아니지만 일부를 쉽게 수정할 수 있게 해줍니다. CPU 및 메모리 요청 변경, 클러스터에 노드 추가, 또는 새로운 CockroachDB Docker 이미지로의 업그레이드와 같은 특정 변경이 쉽습니다. 다른 것들은 `StatefulSet`에서 `DaemonSet`으로 변경하는 것과 같이, 매우 어렵고 오류가 발생하기 쉽습니다. 리소스 구성을 업데이트하려면 몇 가지 명령을 사용하십시오:

* 원하는 수정 사항이 포함된 구성 파일이 있는 경우 다음을 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl apply -f <your-file>.yaml
    ~~~
* 텍스트 편집기를 열고 `StatefulSet`의 YAML 구성 파일을 수동으로 변경하려면 다음을 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl edit statefulset cockroachdb
    ~~~

    DaemonSet의 경우 다음을 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl edit daemonset cockroachdb
    ~~~

* one-liner을 원하는 경우 적절한 JSON을 구성하고 다음과 같이 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl patch statefulset cockroachdb --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"{{page.release_info.docker_image}}:VERSION"}]
    ~~~

이러한 명령에 대한 자세한 내용은 [내부 업데이트에 대한 Kubernetes 설명서](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#in-place-updates-of-resources)나 `kubectl <command> --help` 출력을 참조하십시오.


## 더 보기

- [Kubernets로 CockroachDB 조정](orchestrate-cockroachdb-with-kubernetes.html)
- [배포 체크리스트](recommended-production-settings.html)
- [SQL 성능 모범 사례](performance-best-practices-overview.html)
- [성능 문제 해결](query-behavior-troubleshooting.html#performance-issues)
