---
title: Performance Benchmarking with TPC-C
summary: Learn how to benchmark CockroachDB against TPC-C.
toc: true
---

이 페이지는 CockroachDB의 [TPC-C](http://www.tpc.org/tpcc/) 성능 벤치마킹을 안내합니다. 2개의 TPC-C 데이터 세트에서 tpmC (신규 주문 트랜잭션/분)를 측정합니다:

- 3 노드의 1,000 개의 창고 (200GB의 전체 데이터 세트 크기)
- 노드 30 개의 1만 개의 창고 (2TB의 전체 데이터 세트 크기)

스펙트럼에서 이 두 가지 사항은 CockroachDB가 적당한 규모의 생산 워크로드에서 대규모 배포로 확장되는 방법을 보여줍니다. 이것은 CockroachDB가 2TB를 초과하는 TPC-C 데이터 세트에서 128,000 tpmC 이상의 높은 OLTP 성능을 달성하는 방법을 보여줍니다.

## 작은 클러스터 벤치마크

### 1단계. 3개의 Google Cloud Platform GCE 인스턴스 생성

1. CockroachDB 노드에 대한 [3 인스턴스를 생성](https://cloud.google.com/compute/docs/instances/create-start-instance)하십시오. 각 인스턴스를 생성하는 동안:
    - `n1-highcpu-16` 머신 타입을 사용하십시오.

        우리의 TPC-C 벤치마킹에서는, `n1-highcpu-16` 머신을 사용합니다. 현재, 높은 트래픽 시나리오에서 이것이 (혹은 vCPU 수가 많은 머신들) CockroachDB에 가장 적합한 구성이라고 생각합니다.
    - [SCSI 인터페이스를 사용하여 로컬 SSD를 생성 및 마운트](https://cloud.google.com/compute/docs/disks/local-ssd#create_local_ssd)하십시오.

         각 가상 컴퓨터에 단일 로컬 SSD를 연결합니다. 로컬 SSD는 각 VM에 연결된 지연 시간이 적은 디스크이므로, 성능을 최대화합니다. 이 구성은 베어 메탈 배포가 가장 유사한 것처럼 보이기 때문에, 하나의 물리적 디스크에 각각 직접 연결된 머신으로 이루어집니다. 네트워크 연결 블록 저장소는 사용하지 않는 것이 좋습니다.
    - [쓰기 성능을 위한 로컬 SSD를 최적화](https://cloud.google.com/compute/docs/disks/performance#optimize_local_ssd)하십시오 (**Disable write cache flushing** 섹션을 참조하십시오).
    - 앞에서 생성한 Admin UI 방화벽 규칙을 적용하려면, **Management, disk, networking, SSH keys**를 클릭하고, **Networking**, 탭을 선택한 다음 **Network tags** 필드에 `cockroachdb`를 입력하십시오.

2. 각 `n1-highcpu-16` 인스턴스의 내부 IP 주소를 기록하십시오. CockroachDB 노드를 시작할 때 이 주소가 필요합니다.

3. TPC-C 벤치 마크를 실행하기 위한 네 번째 인스턴스를 생성하십시오. 

{{site.data.alerts.callout_danger}}
 이 구성은 오직 성능 벤치마킹을 위한 것입니다. 프로덕션 배포의 경우, 복원 가능성을 위해 적어도 세 가지 가용성 영역에서 데이터의 균형을 유지하는 것과 같은 다른 중요한 고려 사항이 있습니다. 자세한 내용은 [프로덕션 체크리스트](recommended-production-settings.html)를 참조하십시오.
{{site.data.alerts.end}}

### 2단계. 3-노드 클러스터 시작

1. SSH를 첫 번째 `n1-highcpu-16` 인스턴스로 변경.

2. Linux용 [CockroachDB 아카이브](https://binaries.cockroachdb.com/cockroach-{{page.release_info.version}}.linux-amd64.tgz)를 다운로드하고, 바이너리를 추출한 후, `PATH`에 복사하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget -qO- https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz \
    | tar  xvz
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cp -i cockroach-{{ page.release_info.version }}.linux-amd64/cockroach /usr/local/bin
    ~~~

    권한 오류가 발생하,면 명령 앞에 `sudo`를 추가하십시오.

3. [`cockroach start`](start-a-node.html) 명령을 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --advertise-addr=<node1 internal address> \
    --join=<node1 internal address>,<node2 internal address>,<node3 internal address> \
    --cache=.25 \
    --max-sql-memory=.25 \
    --background
    ~~~

4. 다른 두 개의 `n1-highcpu-16` 인스턴스에 대해 1 - 3 단계를 반복하십시오.

5. 네 번째 `n1-highcpu-16` 인스턴스에서, [`cockroach init`](initialize-a-cluster.html) 명령을 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach init --insecure --host=<address of any node>
    ~~~

    그런 다음 각 노드는 CockroachDB 버전, 웹 UI의 URL 및 클라이언트의 SQL URL과 같은 [표준 출력](start-a-node.html#standard-output)에 유용한 정보를 출력합니다. 

### 3단계. 벤치 마크를 위한 데이터 로드

CockroachDB는 클러스터에 대한 클라이언트 트래픽을 시뮬레이션하기 위한 여러 로드 생성기가 포함된 미리 빌드된 Linux용 `workload` 바이너리를 제공합니다. 이 단계에는 CockroachDB의 TPC-C 워크로드 버전이 있습니다.

1. SSH를 네 번째 인스턴스 (CockroachDB 노드를 실행하지 않는 인스턴스)로 변경하고, `workload`를 다운로드하여, 실행 가능하게 만듭니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget https://edge-binaries.cockroachdb.com/cockroach/workload.LATEST | chmod 755 workload.LATEST
    ~~~

2. `PATH`로 `workload`의 이름을 변경하고 복사하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cp -i workload.LATEST /usr/local/bin/workload
    ~~~

3. TPC-C 워크로드를 시작하여, [노드의 연결 문자열](connection-parameters.html#connect-using-a-url)에서 가리키고 연결 매개 변수를 포함하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./workload.LATEST fixtures load tpcc \
    --warehouses=1000 \
    "postgres://root@<node1 address>?sslmode=disable"
    ~~~

    이 명령은 클러스터에 대해 TPC-C 워크로드를 실행합니다. 이 작업에는 약 1시간이 걸리고 1,000 개의 데이터 창고를 로드합니다. 

    {{site.data.alerts.callout_success}}
    더 많은 `tpcc` 옵션을 보려면, `workload run tpcc --help`를 사용하십시오. `workload`에 포함 된 다른 로드 생성기에 대한 자세한 내용은, `workload run --help`를 사용하십시오.
    {{site.data.alerts.end}}

4. 로드 생성기의 진행 상황을 모니터링하려면, **Admin UI > Jobs** 테이블의 프로세스를 따르십시오.

     시작시 임의 노드의 표준 출력에서 `admin` 필드의 주소를 브라우저로 지정하여 [Admin UI](admin-ui-access-and-navigate.html)를 엽니다. **Admin UI > Jobs** 테이블의 프로세스를 따르십시오.

### 4단계. 벤치마크 실행

여전히 네 번째 인스턴스에서, 다른 3 인스턴스와 `workload`를 5분간 실행하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ ./workload.LATEST run tpcc \
--ramp=30s \
--warehouses=1000 \
--duration=300s \
--split \
--scatter \
"postgres://root@<node1 address>?sslmode=disable postgres://root@<node2 address>?sslmode=disable postgres://root@<node3 address>?sslmode=disable"
~~~

### 5단계. 결과 해석

`workload`가 실행을 마쳤으면, 최종 출력 줄을 보아야 합니다:

~~~ shell
_elapsed_______tpmC____efc__avg(ms)__p50(ms)__p90(ms)__p95(ms)__p99(ms)_pMax(ms)
  298.9s    13154.0 102.3%     75.1     71.3    113.2    130.0    184.5    436.2
~~~

또한 각 개별 쿼리에 대한 감사 확인 및 지연 시간 통계도 볼 수 있다. 이런 경우, 검사 중 일부는 불충분한 데이터로 인해 `SKIPPED` 되었음을 나타낼 수 있습니다. 좀 더 포괄적인 테스트를 위해, 더 긴 기간 (예 : 2시간) 동안 `workload`을 실행하십시오. `tpmC` (새로운 주문 거래/분) 번호는 헤드 라인 번호이고 `efc` ("효율성")은 CockroachDB가 이론상 최대 `tpmC`에 얼마나 근접하는지 알려줍니다.

[TPC-C 사양](http://www.tpc.org/tpc_documents_current_versions/pdf/tpc-c_v5.11.0.pdf)은 p90 대기 시간 요구 사항을 초 단위로 처리하지만, 여기에서 볼 수 있듯이, CockroachDB는 수백 밀리 초 내에 p90 대기 시간을 초과하여 요구 사항을 훨씬 능가합니다.

## 대규모 클러스터 벤치 마크

CockroachDB의 30개 노드, 10,000개의 창고 TPC-C 결과를 재현하는 방법론은 [3-노드, 1,000개의 창고 예제](#benchmark-a-small-cluster)와 비슷합니다. (더 큰 노드 수 및 데이터 세트 외에도) CockroachDB의 [파티셔닝](partitioning.html) 기능을 사용하여 특정 데이터 섹션에 대한 복제본이 해당 데이터 섹션에 대해 로드 생성기가 쿼리할 동일한 노드에 위치하도록 하는 것이 유일한 차이점이다. 파티셔닝은 워크로드를 클러스터 전체에 골고루 분산시키는 데 도움이 됩니다.

### 시작하기 전에

대규모 클러스터 벤치마킹은 [파티셔닝](partitioning.html)을 사용합니다. 파티셔닝 기능을 사용하려면 유효한 엔터프라이즈 라이센스가 있어야 합니다. 
평가판 또는 전체 엔터프라이즈 라이센스 요청 및 설정에 대한 자세한 내용은, [엔터프라이즈 라이센싱](enterprise-licensing.html)을 참고하십시오. 

### 1단계. 30개의 Google Cloud Platform GCE 인스턴스 생성

1. CockroachDB 노드에 대한 [30 인스턴스를 생성](https://cloud.google.com/compute/docs/instances/create-start-instance)하십시오. 각 인스턴스를 생성하는 동안:
    - `n1-highcpu-16` 머신 타입을 사용하십시오.

        우리의 TPC-C 벤치마킹에서는, `n1-highcpu-16` 머신을 사용합니다. 현재, 높은 트래픽 시나리오에서 이것이 (혹은 vCPU 수가 많은 머신들) CockroachDB에 가장 적합한 구성이라고 생각합니다.
    - [로컬 SSD를 생성 및 마운트](https://cloud.google.com/compute/docs/disks/local-ssd#create_local_ssd)하십시오. 

        각 가상 컴퓨터에 단일 로컬 SSD를 연결합니다. 로컬 SSD는 각 VM에 연결된 지연 시간이 적은 디스크이므로, 성능을 최대화합니다. 이 구성은 베어 메탈 배포가 가장 유사한 것처럼 보이기 때문에, 하나의 물리적 디스크에 각각 직접 연결된 머신으로 이루어집니다. 네트워크 연결 블록 저장소는 사용하지 않는 것이 좋습니다.
    - [쓰기 성능을 위해 로컬 SSD를 최적화](https://cloud.google.com/compute/docs/disks/performance#optimize_local_ssd)하십시오 (**Disable write cache flushing** 섹션을 참조하십시오).
    - 앞에서 생성한 Admin UI 방화벽 규칙을 적용하려면, **Management, disk, networking, SSH keys**를 클릭하고, **Networking**, 탭을 선택한 다음 **Network tags** 필드에 `cockroachdb`를 입력하십시오.

2. 각 `n1-highcpu-16` 인스턴스의 내부 IP 주소를 기록하십시오. CockroachDB 노드를 시작할 때 이 주소가 필요합니다.

3. TPC-C 벤치 마크 실행을 위한 31번째 인스턴스를 생성하십시오. 

{{site.data.alerts.callout_danger}}
이 구성은 오직 성능 벤치마킹을 위한 것입니다. 프로덕션 배포의 경우, 복원 가능성을 위해 적어도 세 가지 가용성 영역에서 데이터의 균형을 유지하는 것과 같은 다른 중요한 고려 사항이 있습니다. 자세한 내용은 [프로덕션 체크리스트](recommended-production-settings.html)를 참조하십시오.
{{site.data.alerts.end}}

### 2단계. 30-노드 클러스터 시작

1. SSH를 첫 번째 `n1-highcpu-16` 인스턴스로 변경.

2. Linux용 [CockroachDB 아카이브]를 다운로드하고, 바이너리를 추출한 후, `PATH`에 복사하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget -qO- https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz \
    | tar  xvz
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cp -i cockroach-{{ page.release_info.version }}.linux-amd64/cockroach /usr/local/bin
    ~~~

    권한 오류가 발생하면, 명령 앞에 `sudo`를 추가하십시오.

3. [`cockroach start`](start-a-node.html) 명령을 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --advertise-addr=<node1 internal address> \
    --join=<node1 internal address>,<node2 internal address>,<node3 internal address>, [...] \
    --cache=.25 \
    --max-sql-memory=.25 \
    --locality=rack=1 \
    --background
    ~~~

    각 노드는 인공 "랙 번호"를 포함하는 [지역성](start-a-node.html#locality)으로 시작합니다 (예: `--locality=rack=1`). 10번째 노드가 모두 같은 랙의 일부가 되도록 30개의 노드에 10개의 랙을 사용하십시오 (예: `--locality=rack=2`, `--locality=rack=3`, ...).

4. 다른 29 `n1-highcpu-16` 인스턴스에 대해 1 - 3 단계를 반복하십시오.

5. 31번째 `n1-highcpu-16` 인스턴스에서, [`cockroach init`](initialize-a-cluster.html) 명령을 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach init --insecure --host=<address of any node>
    ~~~

    그런 다음 각 노드는 CockroachDB 버전, 웹 UI의 URL 및 클라이언트의 SQL URL과 같은 [표준 출력](start-a-node.html#standard-output)에 유용한 정보를 출력합니다.

### 3단계. 엔터프라이즈 라이센스 추가

이 벤치 마크에서는, 엔터프라이즈 기능인 파티셔닝을 사용합니다. 평가판 또는 전체 엔터프라이즈 라이센스 요청 및 설정에 대한 자세한 내용은 [엔터프라이즈 라이센싱](enterprise-licensing.html)을 참조하십시오. 

엔터프라이즈 라이센스를 시작한 후에 클러스터에 엔터프라이즈 라이센스를 추가하려면, [빌트인 SQL 클라이언트 사용](use-the-built-in-sql-client.html)을 다음과 같이 하십시오:

1. SSH를 31번째 인스턴스 (CockroachDB 노드를 실행하지 않는 인스턴스)로 변경하고 빌트인 SQL 클라이언트를 시작합니다:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure --host=localhost
    ~~~

2. 엔터프라이즈 라이센스 추가:

    {% include copy-clipboard.html %}
    ~~~ shell
    > SET CLUSTER SETTING enterprise.license = '<secret>';
    ~~~

3. `\q` 또는 `ctrl-d`를 사용하여, 대화형 쉘을 종료하십시오.

### 4단계. 벤치마크를 위한 데이터 로드

CockroachDB는 클러스터에 대한 클라이언트 트래픽을 시뮬레이션하기 위한 여러 로드 생성기가 포함된 미리 빌드된 Linux용 `workload` 바이너리를 제공합니다. 이 단계에는 CockroachDB의 TPC-C 워크로드 버전이 있습니다.

1. 여전히 31번째 인스턴스 (CockroachDB 노드를 실행하지 않는 인스턴스)에서, `workload`를 다운로드하고, 실행 가능하게 만듭니다:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget https://edge-binaries.cockroachdb.com/cockroach/workload.LATEST | chmod 755 workload.LATEST
    ~~~

2. `PATH`로 `workload`의 이름을 변경하고 복사하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cp -i workload.LATEST /usr/local/bin/workload
    ~~~

3. TPC-C 워크로드를 시작하여, [노드의 연결 문자열](connection-parameters.html#connect-using-a-url)에서 가리키고 연결 매개 변수를 포함하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $  ./workload.LATEST fixtures load tpcc \
    --warehouses=10000 \
    "postgres://root@<node1 address>?sslmode=disable postgres://root@<node2 address>?sslmode=disable postgres://root@<node3 address>?sslmode=disable [...space separated list]"
    ~~~

    이 명령은 클러스터에 대해 TPC-C 워크로드를 실행합니다. 이 작업에는 약 1시간이 걸리고 10,000 개의 "창고" 데이터를 로드합니다.

    {{site.data.alerts.callout_success}}
    더 많은 `tpcc` 옵션을 보려면, `workload run tpcc --help`를 사용하십시오. `workload`에 포함된 다른 로드 생성기에 대한 자세한 내용은, `workload run --help`를 사용하십시오.
    {{site.data.alerts.end}}

4. 로드 생성기의 진행 상황을 모니터링하려면, **Admin UI > Jobs** 테이블의 프로세스를 따르십시오.

     시작시 임의 노드의 표준 출력에서 `admin` 필드의 주소를 브라우저로 지정하여 [Admin UI](admin-ui-access-and-navigate.html)를 엽니다.

### 5단계. 스냅샷 비율 높이기

[스냅샷 비율 높이기](cluster-settings.html)를 통해, 이 대규모 데이터 이동 속도를 높일 수 있습니다:

1. 여전히 31번째 인스턴스에서, 빌트인 SQL 클라이언트를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure --host=localhost
    ~~~

2. 스냅샷 비율을 높이기 위해 클러스터 설정을 지정하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SET CLUSTER SETTING kv.snapshot_rebalance.max_rate='64MiB';
    ~~~

3. Exit the interactive shell, using `\q` or `ctrl-d`.

### 6단계. 데이터베이스 파티션

다음으로 [데이터베이스를 파티션](partitioning.html)하여 모든 TPC-C 테이블과 인덱스를 랙당 하나씩 10 개의 파티션으로 나눈 다음, [영역 구성](configure-replication-zones.html)을 사용하여 그 파티션들을 특정 랙에 고정하십시오.

1. 31번째 인스턴스에서, 파티셔닝을 시작합니다:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ulimit -n 10000 && workload.LATEST run tpcc \
    --partitions=10 \
    --split \
    --scatter \
    --warehouses=10000 \
    --duration=1s \
    "postgres://root@<node31 address>?sslmode=disable"
    ~~~

    이 명령은 파티션을 추가할 수 있을 만큼 충분히 긴 1초 동안 클러스터에 대해 TPC-C 워크로드를 실행합니다.

    데이터 파티셔닝에는 최소 12시간이 걸립니다. TPC-C-10K 용으로 복제된 2TB 이상의 모든 데이터를 올바른 위치로 이동해야 하므로 이 작업이 오래 걸립니다.

2. 진행 상황을 확인하려면, **Admin UI > Metrics > Queues > Replication Queue** 그래프의 프로세스를 따르십시오. 보다 세분화된 그래프를 보려면 기간을 **Last 10 Min**으로 변경하십시오.

    시작시 임의 노드의 표준 출력에서 `admin` 필드의 주소를 브라우저로 지정하여 [Admin UI](admin-ui-access-and-navigate.html)를 엽니다.

    모든 작업에 대해 복제 큐가 '0'으로 설정되면, 클러스터가 재조정을 완료하고 테스트 준비가 완료되어야 합니다.

### 7단계. 벤치마크 실행

여전히 31번째 인스턴스에서, 다른 30 인스턴스에 대해 5분 동안 `workload`을 실행하십시오:

~~~ shell
$ ulimit -n 10000 && ./workload.LATEST run tpcc \
--warehouses=10000 \
--ramp=30s \
--duration=300s \
--split \
--scatter \
"postgres://root@<node1 address>?sslmode=disable postgres://root@<node2 address>?sslmode=disable postgres://root@<node3 address>?sslmode=disable [...space separated list]"
~~~

### 8단계. 결과 해석

`workload`가 실행을 마친 후에는, [작은 클러스터 벤치마크](#benchmark-a-small-cluster)의 결과와 비슷한 최종 출력 라인을 보게 될 것입니다. `tpmC`는 창고의 수가 증가함에 따라 약 10배 더 높아야합니다:

~~~ shell
_elapsed_______tpmC____efc__avg(ms)__p50(ms)__p90(ms)__p95(ms)__p99(ms)_pMax(ms)
  291.6s   131109.8 102.0%    115.3     88.1    184.5    268.4    637.5   4295.0
~~~

또한 각 개별 쿼리에 대한 감사 확인 및 지연 시간 통계도 볼 수 있다. 이런 경우, 검사 중 일부는 불충분한 데이터로 인해 `SKIPPED` 되었음을 나타낼 수 있습니다. 좀 더 포괄적인 테스트를 위해, 더 긴 기간 (예 : 2시간) 동안 `workload`을 실행하십시오. `tpmC` (새로운 주문 거래/분) 번호는 헤드 라인 번호이고 `efc` ("효율성")은 CockroachDB가 이론상 최대 `tpmC`에 얼마나 근접하는지 알려줍니다.

[TPC-C 사양](http://www.tpc.org/tpc_documents_current_versions/pdf/tpc-c_v5.11.0.pdf)은 p90 대기 시간 요구 사항을 초 단위로 처리하지만, 여기에서 볼 수 있듯이, CockroachDB는 수백 밀리 초 내에 p90 대기 시간을 초과하여 요구 사항을 훨씬 능가합니다.

<!-- ## Roachprod directions for performance benchmarking a small cluster

roachprod를 사용하여 클러스터를 생성: `roachprod create lauren-tpcc --gce-machine-type "n1-highcpu-16" --local-ssd --nodes 4`
 
최신 버전의 CockroachDB 다운로드:

- `roachprod run lauren-tpcc 'wget https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz'`

- `roachprod run lauren-tpcc "curl https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz | tar -xvz; mv cockroach-v2.0.4.linux-amd64/cockroach cockroach"`

SSD를 보다 효율적으로 구성:`roachprod run lauren-tpcc -- 'sudo umount /mnt/data1; sudo mount -o discard,defaults,nobarrier /dev/disk/by-id/google-local-ssd-0 /mnt/data1/; mount | grep /mnt/data1'`

3 노드를 시작: `roachprod start lauren-tpcc:1-3`

라이센스 추가: 

- `roachprod sql lauren-tpcc:1`
- Set CLUSTER SETTING enterprise.license = '<secret>'

샘플 워크로드 및 RESTORE TPC-C 데이터 실행: `roachprod run lauren-tpcc:4 "wget https://edge-binaries.cockroachdb.com/cockroach/workload.LATEST && chmod a+x workload.LATEST"`

데이터세트를 클러스터에 로드하도록 워크로드를 알립니다: `roachprod run lauren-tpcc:4 "./workload.LATEST fixtures load tpcc {pgurl:1} --warehouses=1000"` (약 1시간 소요)

Admin UI> 작업 대시 보드로 이동하여 진행 상황을 확인하십시오: `roachprod adminurl lauren-tpcc:1`

RESTORE가 완료되면 벤치 마크를 실행하십시오: `roachprod run lauren-tpcc:4 "./workload.LATEST run tpcc --ramp=30s --warehouses=1000 --duration=300s --split --scatter {pgurl:1-3}"`

워크로드가 끝나면, 최종 출력 줄이 표시됩니다:

~~~ shell
_elapsed_______tpmC____efc__avg(ms)__p50(ms)__p90(ms)__p95(ms)__p99(ms)_pMax(ms)
  298.8s    13149.8 102.3%    108.4    100.7    176.2    201.3    285.2    604.0
~~~

## 대규모 클러스터에서 성능 벤치마킹을 위한 Roachprod의 지침

roachprod를 사용하여 클러스터를 생성합니다: `roachprod create lauren-tpcc --gce-machine-type "n1-highcpu-16" --local-ssd --nodes 31`

주의: 파티셔닝에 12시간이 걸리므로, 테스트 클러스터의 수명을 연장하십시오 (roachprod를 사용하는 경우): `roachprod extend lauren-tpcc --lifetime=30h`


최신 버전의 CockroachDB 다운로드:

- `roachprod run lauren-tpcc 'wget https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz'`

- `roachprod run lauren-tpcc "curl https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz | tar -xvz; mv cockroach-v2.0.4.linux-amd64/cockroach cockroach"`

SSD를 보다 효율적으로 구성: `roachprod run lauren-tpcc -- 'sudo umount /mnt/data1; sudo mount -o discard,defaults,nobarrier /dev/disk/by-id/google-local-ssd-0 /mnt/data1/; mount | grep /mnt/data1'`

30 노드를 시작: `roachprod start lauren-tpcc:1-30 --racks 10`

라이센스 추가:

- `roachprod sql lauren-tpcc:1 -- -e "SET CLUSTER SETTING enterprise.license = '<secret>';"`

샘플 워크로드 및 RESTORE TPC-C 데이터 실행: `roachprod run lauren-tpcc:31 "wget https://edge-binaries.cockroachdb.com/cockroach/workload.LATEST && chmod a+x workload.LATEST"`

데이터세트를 클러스터에 로드하도록 워크로드을 알립니다: `roachprod run lauren-tpcc:31 "./workload.LATEST fixtures load tpcc {pgurl:1} --warehouses=10000"` (this will take about an hour)

Admin UI > 작업 대시보드로 이동하여 진행 상황을 확인하십시오: `roachprod adminurl lauren-tpcc:1`

RESTORE가 완료되면, 스냅샷 클러스터 설정을 다음과 같이 설정하십시오: `roachprod sql lauren-tpcc:1 -- -e "SET CLUSTER SETTING kv.snapshot_rebalance.max_rate='64MiB';"`

데이터베이스 파티션: `roachprod ssh lauren-tpcc:31 "ulimit -n 10000 && ./workload.LATEST run tpcc --partitions=10 --split --scatter --warehouses=10000 --duration=1s {pgurl:1-30}"`

이것은 ~12hrs 걸릴 것입니다. 일단 Replication Queue (Admin UI)가 `0`이 되어 거기에 머무르면, 클러스터는 재조정을 마쳐야하고 테스트 준비가 됩니다.

각 범위가 올바르게 분할되어 있는지 확인하려면, `SHOW testing_ranges FROM TABLE tpcc.new_order;`를 사용합니다. 이것은 `RANGE ID` 와 `REPLICAS` 테이블을 보여줍니다. `REPLICAS` 열 자체가 정렬되면 (예 : 범위 1,2,3 범위 4,5,6 등), 분할이 완료됩니다.

벤치 마크를 실행: `roachprod run lauren-tpcc:31 "ulimit -n 10000 && ./workload.LATEST run tpcc --ramp=30s --warehouses=10000 --duration=300s --split --scatter {pgurl:1-30}"`

워크로드가 끝나면, 최종 출력 라인을 볼 수 있습니다.-->

## 더 보기

- [CockroachDB 2.0 벤치마킹: 성능 보고서](https://www.cockroachlabs.com/guides/cockroachdb-performance/)
- [SQL 성능 우수 사례](performance-best-practices-overview.html)
- [Digital Ocean에 CockroachDB 배포](deploy-cockroachdb-on-digital-ocean.html)
