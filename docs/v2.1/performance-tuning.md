---
title: Performance Tuning
summary: Essential techniques for getting fast reads and writes in a single- and multi-region CockroachDB deployment.
toc: true
drift: true
---

이 튜토리얼은 단일 영역 배치에서 시작하여 여러 영역으로 확장한 경우까지 CockroachDB에서 빠른 읽기 및 쓰기를 수행하기 위한 필수 기술을 설명합니다.

여기에 일부 소개된 조정 권장 사항의 전체 목록을 보려면 [SQL 성능 모범 사례](performance-best-practices-overview.html)를 참조하십시오.

## 개요

### 토폴로지

3-노드 CockroachDB 클러스터를 단일 GCE(Google Compute Engine) 영역에 두고 클라이언트 애플리케이션 워크로드를 실행하기 위한 추가 인스턴스와 함께 시작하십시오.

<img src="{{ 'images/v2.1/perf_tuning_single_region_topology.png' | relative_url }}" alt="Perf tuning topology" style="max-width:100%" />

{{site.data.alerts.callout_info}}
단일 GCE 영역 내에서 인스턴스 간의 네트워크 지연 시간은 밀리초 미만이어야 합니다.
{{site.data.alerts.end}}

그런 다음 클러스터를 세 개의 GCE 지역 간에 실행되는 9개 노드로 확장하고 각 지역에서 클라이언트 애플리케이션 워크로드에 대한 추가 인스턴스를 포함하십시오.

<img src="{{ 'images/v2.1/perf_tuning_multi_region_topology.png' | relative_url }}" alt="Perf tuning topology" style="max-width:100%" />

이 튜토리얼에 설명된 성능을 재현하려면:

- 각 CockroachDB 노드에 대해,  Ubuntu 16.04 OS 이미지와 [local SSD](https://cloud.google.com/compute/docs/disks/#localssds) 디스크를 포함하는 [`n1-standard-4`](https://cloud.google.com/compute/docs/machine-types#standard_machine_types) 컴퓨터 유형을 사용합니다.
- 클라이언트 애플리케이션 워크로드를 실행에는 `n1-standard-1` 과 같은 더 작은 인스턴스를 사용하십시오.

### 스키마

스키마 및 데이터는 [CockroachDB 2.0 demo](https://www.youtube.com/watch?v=v2QK5VgLx6E)에 소개된 가상의 피어 투 피어 차량 공유 앱인 MovR을 기반으로 합니다.

<img src="{{ 'images/v2.1/perf_tuning_movr_schema.png' | relative_url }}" alt="Perf tuning schema" style="max-width:100%" />

스키마에 대한 몇 가지 참고 사항:

- 세 개의 자체 설명 테이블: 기본적으로 `users`(사용자)는 서비스를 이용하기 위해 등록한 사람들을, `vehicles`(차량)는 서비스 차량 풀을, `rides`(승차권)는 이용자들의 참여 시기와 장소를 나타낸다. 
- 각 테이블에는 복합 기본 키가 있으며,`city`가 그 첫번째입니다. 처음 단일 영역 배포에서 필요 없지만 ,클러스터를 여러 영역으로 확장하면 이 복합 기본 키를 사용하여 `city` 에 의해 [행 수준의 데이터 지역 분할](partitioning.html#partition-using-primary-key)을 사용할 수 있습니다. 이와 같이 본 튜토리얼은 향후 확장을 위해 설계된 스키마를 설명합니다.
- - 데이터를 가져오는 데 사용할 [`IMPORT`](import.html) 기능이 외부 키를 지원하지 않으므로 [외부 키 제약](foreign-key.html) 없이 데이터를 가져올 수 있습니다. 하지만, 이 기능은 나중에 외부 키를 추가하는 데 필요한 보조 인덱스를 생성합니다.
- `rides` 테이블은`city`와 중복되는 듯 보이는 `vehicle_city`를 포함하고 있습니다. 하나의 열에 둘 이상의 외부 키 제약 조건을 적용할 수는 없지만, 두 개의 외부 키 제약 조건을 `rides` 테이블에 적용해야 하며, 각 제약조건의 일부로 city가 필요합니다. 따라서 이 중복성이 필요하게 됩니다. `city` 와 동기화된 복제본 `vehicle_city`는 [`CHECK` 제약 조건](check.html)을 통해 [이런 한계점](https://github.com/cockroachdb/cockroach/issues/23580)을 극복하도록 합니다.

### 중요한 개념

이 튜토리얼의 기술을 이해하고 상황에 맞게 적용할 수 있도록 하기 위해, 먼저 중요한 [CockroachDB 아키텍처 개념들](architecture/overview.html)을 복습하는 것이 중요합니다.

{% include {{ page.version.version }}/misc/basic-terms.md %}

위에서 언급했듯이, 쿼리가 실행될 때 클러스터는 관련 데이터가 포함된 범위에 대한 요청을 임대자에게 라우팅합니다. 쿼리가 여러 범위에 도달하면 요청은 여러 명의 임대자에게 전달됩니다. 읽기 요청의 경우 관련 범위의 임대자만 데이터를 검색합니다. 쓰기 요청의 경우, Raft 합의 프로토콜이 쓰기가 완료되기 전에 관련 범위의 복제본의 과반수가 동의해야 한다고 규정합니다.

이 기계들이 어떤 가상의 쿼리에서 어떻게 작용하는지 생각해 봅시다.

#### Read 시나리오

첫째, 다음과 같은 간단한 읽기 시나리오를 상상해 보십시오.

- 클러스터에 노드가 3개 있다
- 3개의 작은 테이블이 있으며, 각 테이블은 단일 범위에 있다.
- 범위를 3회 복제한다 (기본값)
- 노드 2에 대해 쿼리를 실행하여 테이블 3에서 읽어온다

<img src="{{ 'images/v2.1/perf_tuning_concepts1.png' | relative_url }}" alt="Perf tuning concepts" style="max-width:100%" />

이 경우에서:

1. 노드 2(게이트 노드)는 테이블 3에서 읽기 요청을 수신한다.
2. 테이블 3의 임대자는 노드 3에 있으므로 요청은 노드 3에 배치된다.
3. 노드 3은 노드 2로 데이터를 반환한다.
4. 노드 2는 클라이언트에 응답한다.

관련 범위의 임대자를 가진 노드에서 쿼리를 수신하는 경우 네트워크 홉이 더 적습니다:

<img src="{{ 'images/v2.1/perf_tuning_concepts2.png' | relative_url }}" alt="Perf tuning concepts" style="max-width:100%" />

#### Write 시나리오

이제 노드 3에 대해 쿼리를 실행하여 테이블 1에 쓰는 간단한 쓰기 시나리오를 상상해 보십시오.


<img src="{{ 'images/v2.1/perf_tuning_concepts3.png' | relative_url }}" alt="Perf tuning concepts" style="max-width:100%" />

이 경우에서:

1. 노드 3(게이트 노드)은 표 1에 쓰기 요청을 수신한다.
2. 테이블 1의 임대자는 노드 1에 있으므로 요청은 노드 1에 배치된다.
3. 임대자는 Raft 리더와 동일한 복제본이므로 (일반적으로), Raft 로그에 쓰기를 동시에 추가하고 노드 2와 3에 후속 복제본을 통보한다.
4. 한 팔로워가 Raft 로그에 쓰기를 추가함과 동시에(따라서 대부분의 복제본이 동일한 Raft 로그에 근거해 합의), 리더에게 통지하고, 쓰기는 동의한 복제본의 키 값에 커밋된다. 이 다이어그램에서 노드 2의 팔로워가 쓰기를 인정했지만 이는 노드 3의 팔로워일 수도 있었다. 또한 합의안에 포함되지 않은 팔로워들은 보통 다른 이들이 커밋된 직후 커밋된다.
5. 노드 1은 노드 3에 대한 커밋 확인을 반환한다.
6. 노드 3은 클라이언트에 응답한다.

읽기 시나리오에서와 같이, 관련 범위의 임대자와 Raft 리더가 있는 노드에서 쓰기 요청을 수신하는 경우, 네트워크 홉이 더 적습니다.

<img src="{{ 'images/v2.1/perf_tuning_concepts4.png' | relative_url }}" alt="Perf tuning concepts" style="max-width:100%" />

#### 네트워크 및 I/O 병목 현상

위의 예를 염두에 두고, 네트워크 지연 시간 및 디스크 I/O를 항상 잠재적인 성능 병목 현상으로 고려하는 것이 중요합니다. 
요약:

- 읽기의 경우, 게이트웨이 노드와 임대자 사이의 홉은 대기 시간을 추가한다.
- 쓰기의 경우, 게이트웨이 노드와 임대자/Raft 리더 간 홉, 임대자/Raft 리더와 Raft 팔로워 간의 홉이 대기 시간을 더한다. 또한 Raft 로그 항목은 쓰기가 커밋되기 전에 디스크에 계속 남아 있으므로 디스크 I/O가 중요하다.

## 단일 영역 배포

<!-- roachprod instructions for single-region deployment
1. Reserve 12 instances across 3 GCE zone: roachprod create <yourname>-tuning --geo --gce-zones us-east1-b,us-west1-a,us-west2-a --local-ssd -n 12
2. Put cockroach` on all instances:
   - roachprod stage <yourname>-tuning release v2.1.0-beta.20181008
3. Start the cluster in us-east1-b: roachprod start <yourname>-tuning:1-3
4. You'll need the addresses of all instances later, so list and record them somewhere: roachprod list -d <yourname>-tuning
5. Import the Movr dataset:
   - SSH onto instance 4: roachprod run <yourname>-tuning:4
   - Run the SQL commands in Step 4 below.
8. Install the Python client:
   - Still on instance 4, run commands in Step 5 below.
9. Test/tune read performance:
   - Still on instance 4, run commands in Step 6.
10. Test/tune write performance:
   - Still on instance 4, run commands in Step 7.
-->

### 1단계. 네트워크 구성

CockroachDB는 두 개의 포트에서 TCP 통신을 필요로 합니다.

- **26257** (`tcp:26257`) : 노드 간 통신용 (클러스터로 작업)
- **8080** (`tcp:8080`) : 웹 UI 액세스용

기본적으로 GCE 인스턴스는 내부 IP 주소로 통신하므로, 노드 간 통신을 활성화하기 위해 어떤 조치도 취할 필요가 없습니다. 그러나 로컬 네트워크에서 웹 UI에 액세스하려면 [프로젝트에 대한 방화벽 규칙 생성](https://cloud.google.com/vpc/docs/using-firewalls)을 해야 합니다.

Field | Recommended Value
------|------------------
Name | **cockroachweb**
Source filter | IP ranges
Source IP ranges | Your local network's IP ranges
Allowed protocols | **tcp:8080**
Target tags | `cockroachdb`

{{site.data.alerts.callout_info}}
The **tag** feature will let you easily apply the rule to your instances.
{{site.data.alerts.end}}

### 2단계. 인스턴스 생성

You'll start with a 3-node CockroachDB cluster in the `us-east1-b` GCE zone, with an extra instance for running a client application workload.

1. CockroachDB 노드에 대해 [3 instances를 생성](https://cloud.google.com/compute/docs/instances/create-start-instance) 하십시오. 각 인스턴스를 생성하는 동안:
    - `us-east1-b` [영역](https://cloud.google.com/compute/docs/regions-zones/)을 선택
    - `n1-standard-4` 컴퓨터 유형(4 vCPUs, 15 GB memory) 사용
    - Ubuntu 16.04 OS 이미지 사용
    - [로컬 SSD 생성 및 마운트](https://cloud.google.com/compute/docs/disks/local-ssd#create_local_ssd).
    - 앞서 생성한 웹 UI 방화벽 규칙을 적용하려면  **Management, disk, networking, SSH keys** 를 클릭하고 **Networking** 탭을 선택한 다음 **Network tags** 필드에`cockroachdb`를 입력

2. 각 `n1-standard-4` 인스턴스의 내부 IP 주소를 기록해 두십시오. CockroachDB 노드를 시작할 때 이 주소가 필요할 것입니다.

3. `us-east1-b` 영역에서도 클라이언트 애플리케이션 워크로드를 실행할 수 있는 별도의 인스턴스를 만드십시오. 이 경우는 `n1-standard-1` 와 같이 작아도 됩니다.

### 3단계. 3-노드 클러스터 시작하기

1. 첫번째 `n1-standard-4` 인스턴스에 대한 SSH.

2. [CockroachDB 아카이브](https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz)를 리눅스용으로 다운로드하고, 바이너리를 추출하여, `PATH`에 복사하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget -qO- https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz \
    | tar  xvz
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo cp -i cockroach-{{ page.release_info.version }}.linux-amd64/cockroach /usr/local/bin
    ~~~

3. [`cockroach start`](start-a-node.html) 명령을 실행합니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --advertise-host=<node1 internal address> \
    --join=<node1 internal address>:26257,<node2 internal address>:26257,<node3 internal address>:26257 \
    --locality=cloud=gce,region=us-east1,zone=us-east1-b \
    --cache=.25 \
    --max-sql-memory=.25 \
    --background
    ~~~

4. 나머지 두 개의 `n1-standard-4` 인스턴스에 1 - 3 번을 반복합니다.

5. `n1-standard-4` 인스턴스 중 하나에, [`cockroach init`](initialize-a-cluster.html) 명령을 실행합니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach init --insecure --host=localhost
    ~~~

    그런 다음 각 노드는 CockroachDB 버전, 웹 UI URL 및 클라이언트의 SQL URL과 같은 유용한 세부 정보를 [표준 출력](start-a-node.html#standard-output)에 출력합니다.

### 4단계. Movr 데이터 세트 가져오기

이제 3개의 미국 동부 도시(뉴욕, 보스턴, 워싱턴 DC)와 3개의 미국 서부 도시(Los Angeles, San Francisco, 시애틀)에서 사용자, 차량 및 승차권을 나타내는 Movr의 데이터를 가져올 것이다.

1. 네 번째 인스턴스에 대한 SSH(CockroachDB 노드를 실행하지 않는 경우)

2. [CockroachDB 아카이브](https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz)에서 리눅스용을 다운로드하고, 바이너리를 추출하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget -qO- https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz \
    | tar  xvz
    ~~~

3. `PATH`에 바이너리를 복사합니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo cp -i cockroach-{{ page.release_info.version }}.linux-amd64/cockroach /usr/local/bin
    ~~~

4. [built-in SQL shell](use-the-built-in-sql-client.html)를 시작하여 CockroachDB 노드 중 하나를 가리키십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure --host=<address of any node>
    ~~~

5. `movr` 데이터베이스를 만들고 기본값으로 설정하십시오.

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE movr;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > SET DATABASE = movr;
    ~~~

6. [`IMPORT`](import.html) 문을 사용하여 `users`, `vehicles,` `rides` 테이블을 만들고 채웁니다.

    {% include copy-clipboard.html %}
    ~~~ sql
    > IMPORT TABLE users (
      	id UUID NOT NULL,
        city STRING NOT NULL,
      	name STRING NULL,
      	address STRING NULL,
      	credit_card STRING NULL,
        CONSTRAINT "primary" PRIMARY KEY (city ASC, id ASC)
    )
    CSV DATA (
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/users/n1.0.csv'
    );
    ~~~

    ~~~
            job_id       |  status   | fraction_completed | rows | index_entries | system_records | bytes
    +--------------------+-----------+--------------------+------+---------------+----------------+--------+
      390345990764396545 | succeeded |                  1 | 1998 |             0 |              0 | 241052
    (1 row)

    Time: 2.882582355s
    ~~~    

    {% include copy-clipboard.html %}
    ~~~ sql
    > IMPORT TABLE vehicles (
        id UUID NOT NULL,
        city STRING NOT NULL,
        type STRING NULL,
        owner_id UUID NULL,
        creation_time TIMESTAMP NULL,
        status STRING NULL,
        ext JSON NULL,
        mycol STRING NULL,
        CONSTRAINT "primary" PRIMARY KEY (city ASC, id ASC),
        INDEX vehicles_auto_index_fk_city_ref_users (city ASC, owner_id ASC)
    )
    CSV DATA (
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/vehicles/n1.0.csv'
    );
    ~~~

    ~~~
            job_id       |  status   | fraction_completed | rows  | index_entries | system_records |  bytes
    +--------------------+-----------+--------------------+-------+---------------+----------------+---------+
      390346109887250433 | succeeded |                  1 | 19998 |         19998 |              0 | 3558767
    (1 row)

    Time: 5.803841493s
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > IMPORT TABLE rides (
      	id UUID NOT NULL,
        city STRING NOT NULL,
        vehicle_city STRING NULL,
      	rider_id UUID NULL,
      	vehicle_id UUID NULL,
      	start_address STRING NULL,
      	end_address STRING NULL,
      	start_time TIMESTAMP NULL,
      	end_time TIMESTAMP NULL,
      	revenue DECIMAL(10,2) NULL,
        CONSTRAINT "primary" PRIMARY KEY (city ASC, id ASC),
        INDEX rides_auto_index_fk_city_ref_users (city ASC, rider_id ASC),
        INDEX rides_auto_index_fk_vehicle_city_ref_vehicles (vehicle_city ASC, vehicle_id ASC),
        CONSTRAINT check_vehicle_city_city CHECK (vehicle_city = city)
    )
    CSV DATA (
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/rides/n1.0.csv',
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/rides/n1.1.csv',
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/rides/n1.2.csv',
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/rides/n1.3.csv',
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/rides/n1.4.csv',
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/rides/n1.5.csv',
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/rides/n1.6.csv',
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/rides/n1.7.csv',
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/rides/n1.8.csv',
        'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/perf-tuning/rides/n1.9.csv'
    );
    ~~~

    ~~~
            job_id       |  status   | fraction_completed |  rows  | index_entries | system_records |   bytes
    +--------------------+-----------+--------------------+--------+---------------+----------------+-----------+
      390346325693792257 | succeeded |                  1 | 999996 |       1999992 |              0 | 339741841
    (1 row)

    Time: 44.620371424s
    ~~~

    {{site.data.alerts.callout_success}}
    웹 UI의 [**Jobs** page](admin-ui-jobs-page.html) 에서 모든 스키마 변경 작업(예: 보조 인덱스 추가)뿐만 아니라 가져오기 작업의 진행률을 확인할 수 있다.
    {{site.data.alerts.end}}

7. 논리적인 결과로, 테이블 사이에는 여러 개의 [외부 키](foreign-key.html 관계가 있어야 한다.

    Referencing columns | Referenced columns
    --------------------|-------------------
    `vehicles.city`, `vehicles.owner_id` | `users.city`, `users.id`
    `rides.city`, `rides.rider_id` | `users.city`, `users.id`
    `rides.vehicle_city`, `rides.vehicle_id` | `vehicles.city`, `vehicles.id`

    앞서 언급한 바와 같이, 이러한 관계를 `IMPORT` 도중에는 구축할 수 없었지만, 필요한 보조 인덱스를 만들 수 있었습니다.
    그럼 이제 외부 키 제약 조건을 추가해 봅시다.

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER TABLE vehicles
    ADD CONSTRAINT fk_city_ref_users
    FOREIGN KEY (city, owner_id)
    REFERENCES users (city, id);
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER TABLE rides
    ADD CONSTRAINT fk_city_ref_users
    FOREIGN KEY (city, rider_id)
    REFERENCES users (city, id);
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > ALTER TABLE rides
    ADD CONSTRAINT fk_vehicle_city_ref_vehicles
    FOREIGN KEY (vehicle_city, vehicle_id)
    REFERENCES vehicles (city, id);
    ~~~

8. built-in SQL 쉘을 중지합니다.

    {% include copy-clipboard.html %}
    ~~~ sql
    > \q
    ~~~

### Step 5. Python 클라이언트 설치

SQL 성능을 측정할 때는 주어진 명령문을 여러 번 실행하고 평균 또는 누적 지연 시간을 확인하는 것이 가장 좋습니다. 
이를 위해 Python 테스트 클라이언트를 설치하고 사용하십시오.

1. 여전히 네 번째 인스턴스에서 모든 시스템 소프트웨어가 최신 상태인지 확인하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo apt-get update && sudo apt-get -y upgrade
    ~~~

2. `psycopg2` 드라이버 설치:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo apt-get install python-psycopg2
    ~~~

3. Python 클라이언트 다운로드:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/performance/tuning.py \
    && chmod +x tuning.py
    ~~~

    아래 그림에서 볼 수 있듯이 이 클라이언트는 커맨드라인 플래그를 통과하도록 합니다.

    Flag | Description
    -----|------------
    `--host` | The IP address of the target node. This is used in the client's connection string.
    `--statement` | The SQL statement to execute.
    `--repeat` | The number of times to repeat the statement. This defaults to 20.

    실행 시 클라이언트는 문장의 모든 반복에 걸쳐 중간값 시간(초)을 인쇄합니다. 또는 두 개의 다른 플래그를 이용하는 방법도 있습니다. 플래그  `--time`을 전달하면 성명이 반복될 때마다 실행 시간을 초 단위로, `--cumulative`를 전달하면 모든 반복에 대해 누적 시간을 초 단위로 출력할 수 있습니다.. 누적(cumulative)은 쓰기를 테스트할 때 특히 유용합니다.

    {{site.data.alerts.callout_success}}
    이와 유사한 도움을 사용자의 쉘에 직접 받기 위해서는`./tuning.py --help`를 사용하십시오.
    {{site.data.alerts.end}}

### 6단계. 읽기 성능 테스트/조정

- [기본 키로 필터링](#filtering-by-the-primary-key)
- [색인화되지 않은 열을 기준으로 필터링(전체 테이블 검색)(#filtering-by-a-non-indexed-column-full-table-scan)
- [보조 인덱스에 의해 필터링](#filtering by Secondary-a-secondary-index)
- [추가 열을 저장하는 보조 인덱스에 의해 필터링](#filtering-by-a-secondary-index-storing-additional-columns)
- [다른 테이블에서 데이터 결합](#joining-data-from-different-tables)
- [`IN (list)` 와 서브쿼리 사용](#using-in-list-with-a-subquery)
- [명시적인 값과 함께 `IN (list)` 사용](#using-in-list-with-explicit-values)

#### 기본 키로 필터링

기본 키를 기준으로 단일 행을 검색하면 일반적으로 2ms 이하로 반환됨:

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT * FROM rides WHERE city = 'boston' AND id = '000007ef-fa0f-4a6e-a089-ce74aa8d2276'" \
--repeat=50 \
--times
~~~

~~~
Result:
['id', 'city', 'vehicle_city', 'rider_id', 'vehicle_id', 'start_address', 'end_address', 'start_time', 'end_time', 'revenue']
['000007ef-fa0f-4a6e-a089-ce74aa8d2276', 'boston', 'boston', 'd66c386d-4b7b-48a7-93e6-f92b5e7916ab', '6628bbbc-00be-4891-bc00-c49f2f16a30b', '4081 Conner Courts\nSouth Taylor, VA 86921', '2808 Willis Wells Apt. 931\nMccoyberg, OH 10303-4879', '2018-07-20 01:46:46.003070', '2018-07-20 02:27:46.003070', '44.25']

Times (milliseconds):
[2.1638870239257812, 1.2159347534179688, 1.0809898376464844, 1.0669231414794922, 1.2650489807128906, 1.1401176452636719, 1.1310577392578125, 1.0380744934082031, 1.199960708618164, 1.0530948638916016, 1.1000633239746094, 1.3430118560791016, 1.104116439819336, 1.0750293731689453, 1.0609626770019531, 1.088857650756836, 1.1639595031738281, 1.2559890747070312, 1.1899471282958984, 1.0449886322021484, 1.1057853698730469, 1.127004623413086, 0.9729862213134766, 1.1131763458251953, 1.0879039764404297, 1.119852066040039, 1.065969467163086, 1.0371208190917969, 1.1181831359863281, 1.0409355163574219, 1.0859966278076172, 1.1398792266845703, 1.032114028930664, 1.1000633239746094, 1.1360645294189453, 1.146078109741211, 1.329183578491211, 1.1131763458251953, 1.1548995971679688, 0.9977817535400391, 1.1138916015625, 1.085042953491211, 1.0950565338134766, 1.0869503021240234, 1.0170936584472656, 1.0571479797363281, 1.0640621185302734, 1.1110305786132812, 1.1279582977294922, 1.1119842529296875]

Median time (milliseconds):
1.10495090485
~~~

일반적으로 열의 부분 집합을 검색하는 것은 훨씬 더 빠를 것입니다.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT rider_id, vehicle_id \
FROM rides \
WHERE city = 'boston' AND id = '000007ef-fa0f-4a6e-a089-ce74aa8d2276'" \
--repeat=50 \
--times
~~~

~~~
Result:
['rider_id', 'vehicle_id']
['d66c386d-4b7b-48a7-93e6-f92b5e7916ab', '6628bbbc-00be-4891-bc00-c49f2f16a30b']

Times (milliseconds):
[2.218961715698242, 1.2569427490234375, 1.3570785522460938, 1.1570453643798828, 1.3251304626464844, 1.3320446014404297, 1.0790824890136719, 1.0139942169189453, 1.0251998901367188, 1.1150836944580078, 1.1949539184570312, 1.2140274047851562, 1.2080669403076172, 1.238107681274414, 1.071929931640625, 1.104116439819336, 1.0230541229248047, 1.0571479797363281, 1.0519027709960938, 1.0688304901123047, 1.0118484497070312, 1.0051727294921875, 1.1889934539794922, 1.0571479797363281, 1.177072525024414, 1.0449886322021484, 1.0669231414794922, 1.004934310913086, 0.9818077087402344, 0.9369850158691406, 1.004934310913086, 1.0461807250976562, 1.0628700256347656, 1.1332035064697266, 1.1780261993408203, 1.0361671447753906, 1.1410713195800781, 1.1188983917236328, 1.026153564453125, 0.9629726409912109, 1.0199546813964844, 1.0409355163574219, 1.0440349578857422, 1.1110305786132812, 1.1761188507080078, 1.508951187133789, 1.2068748474121094, 1.3430118560791016, 1.4159679412841797, 1.3141632080078125]

Median time (milliseconds):
1.09159946442
~~~

#### 색인이 지정되지 않은 열에 의한 필터링(전체 테이블 검색)

기본 키 또는 보조 인덱스에 없는 열을 기반으로 단일 행을 검색할 때 일반적으로 성능이 저하됩니다.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT * FROM users WHERE name = 'Natalie Cunningham'" \
--repeat=50 \
--times
~~~

~~~
Result:
['id', 'city', 'name', 'address', 'credit_card']
['02cc9e5b-1e91-4cdb-87c4-726b4ea7219a', 'boston', 'Natalie Cunningham', '97477 Lee Path\nKimberlyport, CA 65960', '4532613656695680']

Times (milliseconds):
[33.271074295043945, 4.4689178466796875, 4.18400764465332, 4.327058792114258, 5.700111389160156, 4.509925842285156, 4.525899887084961, 4.294157028198242, 4.516124725341797, 5.700111389160156, 5.105018615722656, 4.5070648193359375, 4.798173904418945, 5.930900573730469, 4.445075988769531, 4.1790008544921875, 4.065036773681641, 4.296064376831055, 5.722999572753906, 4.827976226806641, 4.640102386474609, 4.374980926513672, 4.269123077392578, 4.422903060913086, 4.110813140869141, 4.091024398803711, 4.189014434814453, 4.345178604125977, 5.600929260253906, 4.827976226806641, 4.416942596435547, 4.424095153808594, 4.736185073852539, 4.462003707885742, 4.307031631469727, 5.10096549987793, 4.56690788269043, 4.641056060791016, 4.701137542724609, 4.538059234619141, 4.474163055419922, 4.561901092529297, 4.431009292602539, 4.756927490234375, 4.54401969909668, 4.415035247802734, 4.396915435791016, 5.9719085693359375, 4.543066024780273, 5.830049514770508]

Median time (milliseconds):
4.51302528381
~~~

이 쿼리가 성능이 좋지 않은 이유를 이해하려면 `cockroach` 바이너리에 내장된 SQL 클라이언트를 사용하여 쿼리 계획을 [`설명`](explain.html)으로 이동하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="EXPLAIN SELECT * FROM users WHERE name = 'Natalie Cunningham';"
~~~

~~~
  tree | field |  description
+------+-------+---------------+
  scan |       |
       | table | users@primary
       | spans | ALL
(3 rows)
~~~

`spans | ALL` 을 가진 행은 `name` 열에 보조 인덱스가 없으면 기본 키(`city`/`id`)로 정렬된 사용자 테이블의 모든 행을 검색해 정확한 `name` 값을 가진 행을 찾아낸다.

#### 보조 인덱스에 의한 필터링

이 쿼리의 속도를 높이려면 `name`에 보조 인덱스를 추가하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="CREATE INDEX on users (name);"
~~~


이제 쿼리가 훨씬 빠르게 반환될 것입니다.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT * FROM users WHERE name = 'Natalie Cunningham'" \
--repeat=50 \
--times
~~~

~~~
Result:
['id', 'city', 'name', 'address', 'credit_card']
['02cc9e5b-1e91-4cdb-87c4-726b4ea7219a', 'boston', 'Natalie Cunningham', '97477 Lee Path\nKimberlyport, CA 65960', '4532613656695680']

Times (milliseconds):
[3.545045852661133, 1.4619827270507812, 1.399993896484375, 2.0101070404052734, 1.672983169555664, 1.4941692352294922, 1.4650821685791016, 1.4579296112060547, 1.567840576171875, 1.5709400177001953, 1.4760494232177734, 1.6181468963623047, 1.6210079193115234, 1.6970634460449219, 1.6469955444335938, 1.7261505126953125, 1.7559528350830078, 1.875162124633789, 1.7170906066894531, 1.870870590209961, 1.641988754272461, 1.7061233520507812, 1.628875732421875, 1.6558170318603516, 1.7809867858886719, 1.6698837280273438, 1.8429756164550781, 1.6090869903564453, 1.7080307006835938, 1.74713134765625, 1.6620159149169922, 1.9519329071044922, 1.6849040985107422, 1.7440319061279297, 1.8851757049560547, 1.8699169158935547, 1.7409324645996094, 1.9140243530273438, 1.7828941345214844, 1.7158985137939453, 1.6720294952392578, 1.7750263214111328, 1.7368793487548828, 1.9288063049316406, 1.8749237060546875, 1.7838478088378906, 1.8091201782226562, 1.8210411071777344, 1.7669200897216797, 1.8210411071777344]

Median time (milliseconds):
1.72162055969
~~~

성능이 4.51ms(인덱스 미포함 시)에서 1.82ms(인덱스 포함 시)로 개선된 이유를 이해하려면 [`설명`](explain.html)을 사용하여 새 쿼리 계획을 확인하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="EXPLAIN SELECT * FROM users WHERE name = 'Natalie Cunningham';"
~~~

~~~
     tree    | field |                      description
+------------+-------+-------------------------------------------------------+
  index-join |       |
   ├── scan  |       |
   │         | table | users@users_name_idx
   │         | spans | /"Natalie Cunningham"-/"Natalie Cunningham"/PrefixEnd
   └── scan  |       |
             | table | users@primary
(6 rows)
~~~

이것은 CockroachDB가 보조 인덱스(`table | users@users_name_idx`)로 시작한다는 것을 보여줍니다. 이는 `name`에 따라 정렬되어 있어 관련 값(`spans | /"Natalie Cunningham"-/"Natalie Cunningham"/PrefixEnd`) 으로 바로 이동할 수 있습니다다. 단, 쿼리는 보조 인덱스에 없는 값을 반환해야 하므로 CockroachDB는 `name` 값(항상 보조 인덱스에 입력된 정보가 있는 기본 키)이 저장된 기본 키 (`city`/`id`)를 잡고 기본 인덱스의 해당 값으로 이동한 다음 전체 행을 반환합니다.

[earlier discussion of ranges and leaseholders](#important-concepts)를 다시 보면, `users` 테이블이 작기 때문에(64MiB 이하) 기본 인덱스 및 보조 인덱스가 단일 임대자를 가진 단일 범위에 포함됩니다. 그러나, 만약 테이블이 더 크다면 기본 인덱스와 보조 인덱스가 각각 자신의 임대자를 가진 별도의 범위에 존재할 수도 있습니다. 이 경우, 임대자가 서로 다른 노드에 있는 경우 쿼리는 더 많은 네트워크 홉이 필요하게 되어 대기 시간이 더욱 길어지게 됩니다.

#### 추가 열을 저장하는 보조 인덱스에 의한 필터링

특정 열을 기준으로 필터링하지만 테이블의 전체 열의 일부를 검색하는 쿼리가 있는 경우, 해당 쿼리가 기본 인덱스를 검색할 필요가 없도록 보조 인덱스에 추가 열을 [저장](indexes.html#storing-columns)하여 성능을 개선할 수 있습니다.

예를 들어, 사용자의 이름과 신용카드 번호를 자주 검색한다고 가정해 보십시오.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT name, credit_card FROM users WHERE name = 'Natalie Cunningham'" \
--repeat=50 \
--times
~~~

~~~
Result:
['name', 'credit_card']
['Natalie Cunningham', '4532613656695680']

Times (milliseconds):
[2.8769969940185547, 1.7559528350830078, 1.8100738525390625, 1.8839836120605469, 1.5971660614013672, 1.5900135040283203, 1.7750263214111328, 2.2847652435302734, 1.641988754272461, 1.4967918395996094, 1.4641284942626953, 1.6689300537109375, 1.9679069519042969, 1.8970966339111328, 1.8780231475830078, 1.7609596252441406, 1.68609619140625, 1.9791126251220703, 1.661062240600586, 1.9869804382324219, 1.5938282012939453, 1.8041133880615234, 1.5909671783447266, 1.5878677368164062, 1.7380714416503906, 1.638174057006836, 1.6970634460449219, 1.9309520721435547, 1.992940902709961, 1.8689632415771484, 1.7511844635009766, 2.007007598876953, 1.9829273223876953, 1.8939971923828125, 1.7490386962890625, 1.6179084777832031, 1.6510486602783203, 1.6078948974609375, 1.6129016876220703, 1.67083740234375, 1.786947250366211, 1.7840862274169922, 1.956939697265625, 1.8689632415771484, 1.9350051879882812, 1.789093017578125, 1.9249916076660156, 1.8649101257324219, 1.9619464874267578, 1.7361640930175781]

Median time (milliseconds):
1.77955627441
~~~

현재 보조 인덱스가 `name`으로 되어 있는 상태에서, CockroachDB는 여전히 신용카드 번호를 얻기 위해 기본 인덱스를 스캔해야 합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="EXPLAIN SELECT name, credit_card FROM users WHERE name = 'Natalie Cunningham';"
~~~

~~~
     tree    | field |                      description
+------------+-------+-------------------------------------------------------+
  index-join |       |
   ├── scan  |       |
   │         | table | users@users_name_idx
   │         | spans | /"Natalie Cunningham"-/"Natalie Cunningham"/PrefixEnd
   └── scan  |       |
             | table | users@primary
(6 rows)
~~~

이번에는 `name`'에 대한 인덱스를 삭제하고 `credit_card` 값을 저장하는 인덱스를 다시 만들어 봅시다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="DROP INDEX users_name_idx;"
~~~

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="CREATE INDEX ON users (name) STORING (credit_card);"
~~~

이제 `credit_card` 값이 인덱스에 이름`name`으로 저장되므로 CockroachDB는 해당 인덱스만 검사하면 됩니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="EXPLAIN SELECT name, credit_card FROM users WHERE name = 'Natalie Cunningham';"
~~~

~~~
  tree | field |                      description
+------+-------+-------------------------------------------------------+
  scan |       |
       | table | users@users_name_idx
       | spans | /"Natalie Cunningham"-/"Natalie Cunningham"/PrefixEnd
(3 rows)
~~~

이로 인해 성능이 훨씬 빨라지고 지연 시간이 1.77ms(저장되지 않은 인덱스)에서 0.99ms(저장된 인덱스)가 됩니다.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT name, credit_card FROM users WHERE name = 'Natalie Cunningham'" \
--repeat=50 \
--times
~~~

~~~
Result:
['name', 'credit_card']
['Natalie Cunningham', '4532613656695680']

Times (milliseconds):
[1.8029212951660156, 0.9858608245849609, 0.9548664093017578, 0.8459091186523438, 0.9710788726806641, 1.1639595031738281, 0.8571147918701172, 0.8800029754638672, 0.8509159088134766, 0.8771419525146484, 1.1739730834960938, 0.9100437164306641, 1.1181831359863281, 0.9679794311523438, 1.0800361633300781, 1.02996826171875, 1.2090206146240234, 1.0440349578857422, 1.210927963256836, 1.0418891906738281, 1.1951923370361328, 0.9548664093017578, 1.0848045349121094, 0.9748935699462891, 1.15203857421875, 1.0280609130859375, 1.0819435119628906, 0.9641647338867188, 1.0979175567626953, 0.9720325469970703, 1.0638236999511719, 0.9410381317138672, 1.0039806365966797, 1.207113265991211, 0.9911060333251953, 1.0039806365966797, 0.9810924530029297, 0.9360313415527344, 0.9589195251464844, 1.0609626770019531, 0.9949207305908203, 1.0139942169189453, 0.9899139404296875, 0.9818077087402344, 0.9679794311523438, 0.8809566497802734, 0.9558200836181641, 0.8878707885742188, 1.0380744934082031, 0.8897781372070312]

Median time (milliseconds):
0.990509986877
~~~

#### 다른 테이블에서 데이터 결합

보조 인덱스는 다른 테이블의 데이터를 [결합(joins.html)]할 때도 매우 중요합니다.

예를 들어, 특정 날에 `rides`를 시작한 사용자의 수를 세어보고 싶다고 합시다. 이를 위해서는 join을 사용하여 관련 놀이기구를 `rides` 테이블에서 가져온 다음 각 ride의 `rider_id`를 `users` 테이블의 해당 `id` 에 매핑하고 각 매핑을 한 번만 카운트합니다:

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT count(DISTINCT users.id) \
FROM users \
INNER JOIN rides ON rides.rider_id = users.id \
WHERE start_time BETWEEN '2018-07-20 00:00:00' AND '2018-07-21 00:00:00'" \
--repeat=50 \
--times
~~~

~~~
Result:
['count']
['1998']

Times (milliseconds):
[1443.5458183288574, 1546.0000038146973, 1563.858985900879, 1530.3218364715576, 1574.7389793395996, 1572.7760791778564, 1566.4539337158203, 1595.655918121338, 1588.2930755615234, 1567.6488876342773, 1564.5530223846436, 1573.4570026397705, 1581.406831741333, 1587.864875793457, 1575.7901668548584, 1565.0341510772705, 1519.8209285736084, 1599.7698307037354, 1612.4188899993896, 1582.5250148773193, 1604.076862335205, 1596.8739986419678, 1569.6821212768555, 1583.7080478668213, 1549.9720573425293, 1563.5790824890137, 1555.6750297546387, 1577.6000022888184, 1582.3569297790527, 1568.8848495483398, 1580.854892730713, 1566.9701099395752, 1578.8500308990479, 1592.677116394043, 1549.3559837341309, 1561.805009841919, 1561.812162399292, 1543.4870719909668, 1523.3290195465088, 1583.9049816131592, 1565.9120082855225, 1575.1979351043701, 1581.1400413513184, 1616.6048049926758, 1602.9179096221924, 1583.8429927825928, 1570.2300071716309, 1573.2421875, 1558.588981628418, 1548.7489700317383]

Median time (milliseconds):
1573.00913334
~~~

어떤 일이 일어나는지 정확히 이해하려면 [`EXPLAIN`](explain.html)을 사용하여 쿼리 계획을 확인하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="EXPLAIN SELECT count(DISTINCT users.id) \
FROM users \
INNER JOIN rides ON rides.rider_id = users.id \
WHERE start_time BETWEEN '2018-07-20 00:00:00' AND '2018-07-21 00:00:00';"
~~~

~~~
         tree         |    field    |     description
+---------------------+-------------+----------------------+
  group               |             |
   │                  | aggregate 0 | count(DISTINCT id)
   │                  | scalar      |
   └── render         |             |
        └── join      |             |
             │        | type        | inner
             │        | equality    | (id) = (rider_id)
             ├── scan |             |
             │        | table       | users@users_name_idx
             │        | spans       | ALL
             └── scan |             |
                      | table       | rides@primary
                      | spans       | ALL
(13 rows)
~~~

아래부터 읽어 보면, CockroachDB가 먼저 Rides에서 전체 테이블 스캔(`spans    | ALL`)을 실시하여 지정된 범위에서 `start_time` 으로 모든 행을 표시한 다음 `users` 에서 또 다른 전체 테이블 스캔을 수행하여 일치하는 행을 찾아 카운트를 계산한다는 것을 알 수 있습니다.

`rides` 테이블은 거대하기 때문에 데이터가 여러 범위에 걸쳐 나눠져 있습니다. 각 범위는 복제되었으며, 임대자가 있습니다. 따라서 이러한 임대자 중 적어도 일부는 서로 다른 노드에 위치할 가능성이 있습니다. 즉, `rides`의 전체 테이블 스캔은 여러 임대자에 의해 네트워크 홉을 몇 차례 포함한 후 최종적으로 `users` 에게 가서 전체 테이블스캔을 한다는 뜻입니다.

이를 구체적으로 추적하기 위해 [`SHOW EXPERIMENTAL_RANGES`](show-experimental-ranges.html) 문을 사용하여 `rides` 와 `users`에 대한 관련 임대자가 있는 지역을 파악하십시오.


{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="SHOW EXPERIMENTAL_RANGES FROM TABLE rides;"
~~~

~~~
                                  start_key                                  |                                  end_key                                   | range_id | replicas | lease_holder
+----------------------------------------------------------------------------+----------------------------------------------------------------------------+----------+----------+--------------+
  NULL                                                                       | /"boston"/"\xfe\xdd?\xbb4\xabOV\x84\x00M\x89#-a6"/PrefixEnd                |       34 | {1,2,3}  |            2
  /"boston"/"\xfe\xdd?\xbb4\xabOV\x84\x00M\x89#-a6"/PrefixEnd                | /"los angeles"/"<\x12\xe4\xce&\xfdH\u070f?)\xc7\xf92\a\x03"                |       35 | {1,2,3}  |            2
  /"los angeles"/"<\x12\xe4\xce&\xfdH\u070f?)\xc7\xf92\a\x03"                | /"new york"/"0\xa6p\x96\tmOԗ#\xaa\xb7\x90\x12\xe67"/PrefixEnd              |       39 | {1,2,3}  |            2
  /"new york"/"0\xa6p\x96\tmOԗ#\xaa\xb7\x90\x12\xe67"/PrefixEnd              | /"san francisco"/"(m*OM\x15J\xbc\xb6n\xaass\x10\xc4\xff"/PrefixEnd         |       37 | {1,2,3}  |            1
  /"san francisco"/"(m*OM\x15J\xbc\xb6n\xaass\x10\xc4\xff"/PrefixEnd         | /"seattle"/"\x17\xd24\a\xb5\xbdN\x9d\xa1\xd2Dθ^\xe1M"/PrefixEnd            |       40 | {1,2,3}  |            2
  /"seattle"/"\x17\xd24\a\xb5\xbdN\x9d\xa1\xd2Dθ^\xe1M"/PrefixEnd            | /"washington dc"/"\x135\xe5e\x15\xefNۊ\x10)\xba\x19\x04\xff\xdc"/PrefixEnd |       44 | {1,2,3}  |            2
  /"washington dc"/"\x135\xe5e\x15\xefNۊ\x10)\xba\x19\x04\xff\xdc"/PrefixEnd | NULL                                                                       |       46 | {1,2,3}  |            2
(7 rows)
~~~

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="SHOW EXPERIMENTAL_RANGES FROM TABLE users;"
~~~

~~~
  start_key | end_key | range_id | replicas | lease_holder
+-----------+---------+----------+----------+--------------+
  NULL      | NULL    |       49 | {1,2,3}  |            2
(1 row)
~~~

위의 결과는 다음을 말해줍니다:

- `rides` 테이블이 노드 2의 임대자 6개, 노드 1의 임대자 1개로 7개 영역에 걸쳐 구분된다 
- `users` 테이블은 노드 2의 임대자가 있는 한 가지의 범위에 불과하다

지금은 결합의 `WHERE` 조건을 볼 때 7개 영역에 걸친 `rides`의 전체 테이블스캔은 특히 낭비가 심합니다. 쿼리 속도를 높이려면 결합 키(`rides.rider_id`)를 저장하는 `WHERE` 조건 (`rides.start_time`)에서 보조 인덱스를 생성하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="CREATE INDEX ON rides (start_time) STORING (rider_id);"
~~~

{{site.data.alerts.callout_info}}
`rides` 테이블은 100만 행으로 되어 있어 이 지수를 추가하는 데는 몇 분 정도가 소요됩니다.
{{site.data.alerts.end}}

보조 인덱스를 추가하면 쿼리 시간이 1573ms에서 61.56ms로 단축됩니다:

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT count(DISTINCT users.id) \
FROM users \
INNER JOIN rides ON rides.rider_id = users.id \
WHERE start_time BETWEEN '2018-07-20 00:00:00' AND '2018-07-21 00:00:00'" \
--repeat=50 \
--times
~~~

~~~
Result:
['count']
['1998']

Times (milliseconds):
[66.78199768066406, 63.83800506591797, 65.57297706604004, 63.04502487182617, 61.54489517211914, 61.51890754699707, 60.935020446777344, 61.8891716003418, 60.71019172668457, 64.44311141967773, 64.82601165771484, 61.5849494934082, 62.136173248291016, 62.78491020202637, 62.70194053649902, 61.837196350097656, 64.13102149963379, 62.66903877258301, 71.14315032958984, 61.08808517456055, 58.36200714111328, 60.003042221069336, 58.743953704833984, 59.05413627624512, 60.63103675842285, 60.12582778930664, 61.02705001831055, 62.548160552978516, 61.45000457763672, 65.27113914489746, 60.18996238708496, 59.36002731323242, 60.13298034667969, 59.8299503326416, 59.168100357055664, 65.20915031433105, 60.43219566345215, 58.91895294189453, 58.67791175842285, 59.50117111206055, 59.977054595947266, 65.39011001586914, 62.3931884765625, 69.40793991088867, 61.64288520812988, 66.52498245239258, 69.78988647460938, 60.96601486206055, 57.71303176879883, 61.81192398071289]

Median time (milliseconds):
61.5649223328
~~~

성능이 향상된 이유를 이해하려면 [`EXPLAIN`](explain.html)을 사용하여 새 쿼리 계획을 확인하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="EXPLAIN SELECT count(DISTINCT users.id) \
FROM users \
INNER JOIN rides ON rides.rider_id = users.id \
WHERE start_time BETWEEN '2018-07-20 00:00:00' AND '2018-07-21 00:00:00';"
~~~

~~~
         tree         |    field    |                      description
+---------------------+-------------+-------------------------------------------------------+
  group               |             |
   │                  | aggregate 0 | count(DISTINCT id)
   │                  | scalar      |
   └── render         |             |
        └── join      |             |
             │        | type        | inner
             │        | equality    | (id) = (rider_id)
             ├── scan |             |
             │        | table       | users@users_name_idx
             │        | spans       | ALL
             └── scan |             |
                      | table       | rides@rides_start_time_idx
                      | spans       | /2018-07-20T00:00:00Z-/2018-07-21T00:00:00.000000001Z
(13 rows)
~~~

CockroachDB가 `rides@rides_start_time_idx` 보조 인덱스를 사용하여 전체 `rides` 테이블을 스캔할 필요 없이 관련 ride를 검색함으로써 시작된다는 점에 주목하십시오.

새 인덱스의 범위를 확인하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="SHOW EXPERIMENTAL_RANGES FROM INDEX rides@rides_start_time_idx;"
~~~

~~~
                                         start_key                                         |                                         end_key                                          | range_id | replicas | lease_holder
+------------------------------------------------------------------------------------------+------------------------------------------------------------------------------------------+----------+----------+--------------+
  NULL                                                                                     | /2018-07-11T01:37:36.138325Z/"new york"/"\xd4\xe3\u007f\xbc2\xc0Mv\x81B\xd6\xc7٘\x9f\xe6" |       45 | {1,2,3}  |            2
  /2018-07-11T01:37:36.138325Z/"new york"/"\xd4\xe3\u007f\xbc2\xc0Mv\x81B\xd6\xc7٘\x9f\xe6" | NULL                                                                                     |       50 | {1,2,3}  |            2
(2 rows)
~~~

이는 두 사람의 임대자가 노드 2에 있는 두 가지 범위로 인덱스를 저장한다는 것을 말해줍니다. `SHOW EXPERIMENTAL_RANGES FROM TABLE users`의 산출물을 근거로하여 우리는 이미 `users` 테이블의 임대자가 노드 2에 있다는 것을 알고 있습니다.

#### 서브쿼리와 함께 `IN (list)` 사용

이제 가장 많이 사용하는 5대 차량의 최신 주행 정보를 얻고 싶다고 가정해 보십시오. 이를 위해, 서브쿼리를 사용하여 가장 빈번한 5대 차량의 ID를 `rides` 테이블에서 얻고, 각 차량의 가장 최근 ride를 얻기 위한 또다른 쿼리의 `IN` 목록으로 전달합니다.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT vehicle_id, max(end_time) \
FROM rides \
WHERE vehicle_id IN ( \
    SELECT vehicle_id \
    FROM rides \
    GROUP BY vehicle_id \
    ORDER BY count(*) DESC \
    LIMIT 5 \
) \
GROUP BY vehicle_id" \
--repeat=20 \
--times
~~~

~~~
Result:
['vehicle_id', 'max']
['3c950d36-c2b8-48d0-87d3-e0d6f570af62', '2018-08-02 03:06:31.293184']
['0962cdca-9d85-457c-9616-cc2ae2d32008', '2018-08-02 03:01:25.414512']
['78fdd6f8-c6a1-42df-a89f-cd65b7bb8be9', '2018-08-02 02:47:43.755989']
['c6541da5-9858-4e3f-9b49-992e206d2c50', '2018-08-02 02:14:50.543760']
['35752c4c-b878-4436-8330-8d7246406a55', '2018-08-02 03:08:49.823209']

Times (milliseconds):
[3012.6540660858154, 2456.5110206604004, 2482.675075531006, 2488.3930683135986, 2474.393129348755, 2494.3790435791016, 2504.063129425049, 2491.326093673706, 2507.4589252471924, 2482.077121734619, 2495.9230422973633, 2497.60103225708, 2478.4271717071533, 2496.574878692627, 2506.395101547241, 2468.4300422668457, 2476.508140563965, 2497.958183288574, 2480.7958602905273, 2484.0168952941895]

Median time (milliseconds):
2489.85958099
~~~

그러나 보시다시피 쿼리의 `WHERE` 조건이 서브쿼리의 결과로 인해 발생하는 경우, CockroachDB는 사용 가능한 인덱스가 있더라도 전체 테이블을 스캔하기 때문에 이 쿼리의 성능은 느리게 됩니다. 자세한 내용을 보려면 `EXPLAIN`을 사용하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="EXPLAIN SELECT vehicle_id, max(end_time) \
FROM rides \
WHERE vehicle_id IN ( \
    SELECT vehicle_id \
    FROM rides \
    GROUP BY vehicle_id \
    ORDER BY count(*) DESC \
    LIMIT 5 \
) \
GROUP BY vehicle_id;"
~~~

~~~
              tree              |    field    |                     description
+-------------------------------+-------------+-----------------------------------------------------+
  group                         |             |
   │                            | aggregate 0 | vehicle_id
   │                            | aggregate 1 | max(end_time)
   │                            | group by    | @1
   └── join                     |             |
        │                       | type        | semi
        │                       | equality    | (vehicle_id) = (vehicle_id)
        ├── scan                |             |
        │                       | table       | rides@primary
        │                       | spans       | ALL
        └── limit               |             |
             └── sort           |             |
                  │             | order       | -agg0
                  └── group     |             |
                       │        | aggregate 0 | vehicle_id
                       │        | aggregate 1 | count_rows()
                       │        | group by    | @1
                       └── scan |             |
                                | table       | rides@rides_auto_index_fk_vehicle_city_ref_vehicles
                                | spans       | ALL
(20 rows)
~~~

이는 복잡한 쿼리 계획이지만, 주요하게 주목해야할 것은  `subquery` 위의 `rides@primary`의 전체 테이블 스캔입니다. 이는 서브쿼리가 상위 5개 차량의 ID를 반환하면 CockroachDB가 `vehicle_id`의 보조 인덱스를 보다 효율적으로 사용하여 검색할 것으로 예상되지만 실제 CockroachDB는 전체 기본 인덱스를 검색해 각 `vehicle_id`에 대해 `max(end_time)` 행을 찾아낸다는 것을 볼 수 있습니다.(CockroachDB는 향후 버전에서 이러한 제한을 제거하기 위해 노력 중입니다)

#### 명시적 값과 함께 `IN (list)` 사용

CockroachDB는 서브쿼리와 함께 `IN (list)`을 사용할 때 보조 인덱스를 사용하지 않기 때문에, 응용 프로그램이 먼저 상위 5개 차량을 선택하도록 하는 것이 성능 면에서 훨씬 더 효과적입니다.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT vehicle_id \
FROM rides \
GROUP BY vehicle_id \
ORDER BY count(*) DESC \
LIMIT 5" \
--repeat=20 \
--times
~~~

~~~
Result:
['vehicle_id']
['35752c4c-b878-4436-8330-8d7246406a55']
['0962cdca-9d85-457c-9616-cc2ae2d32008']
['c6541da5-9858-4e3f-9b49-992e206d2c50']
['78fdd6f8-c6a1-42df-a89f-cd65b7bb8be9']
['3c950d36-c2b8-48d0-87d3-e0d6f570af62']

Times (milliseconds):
[1049.2329597473145, 1038.0151271820068, 1037.7991199493408, 1036.5591049194336, 1037.7249717712402, 1040.544033050537, 1022.7780342102051, 1056.9651126861572, 1054.3549060821533, 1042.3550605773926, 1042.68217086792, 1031.7370891571045, 1051.880121231079, 1035.8471870422363, 1035.2818965911865, 1035.607099533081, 1040.0230884552002, 1048.8879680633545, 1056.014060974121, 1036.1089706420898]

Median time (milliseconds):
1039.01910782
~~~

그리고, 가장 최근의 차량 ride를 얻기 위해 결과를 `IN` 목록에 올립니다.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT vehicle_id, max(end_time) \
FROM rides \
WHERE vehicle_id IN ( \
  '35752c4c-b878-4436-8330-8d7246406a55', \
  '0962cdca-9d85-457c-9616-cc2ae2d32008', \
  'c6541da5-9858-4e3f-9b49-992e206d2c50', \
  '78fdd6f8-c6a1-42df-a89f-cd65b7bb8be9', \
  '3c950d36-c2b8-48d0-87d3-e0d6f570af62' \
) \
GROUP BY vehicle_id;" \
--repeat=20 \
--times
~~~

~~~
Result:
['vehicle_id', 'max']
['35752c4c-b878-4436-8330-8d7246406a55', '2018-08-02 03:08:49.823209']
['0962cdca-9d85-457c-9616-cc2ae2d32008', '2018-08-02 03:01:25.414512']
['3c950d36-c2b8-48d0-87d3-e0d6f570af62', '2018-08-02 03:06:31.293184']
['78fdd6f8-c6a1-42df-a89f-cd65b7bb8be9', '2018-08-02 02:47:43.755989']
['c6541da5-9858-4e3f-9b49-992e206d2c50', '2018-08-02 02:14:50.543760']

Times (milliseconds):
[1165.5981540679932, 1135.9851360321045, 1201.0550498962402, 1135.0820064544678, 1195.7061290740967, 1132.0109367370605, 1134.9878311157227, 1175.88210105896, 1174.0548610687256, 1152.566909790039, 1164.9351119995117, 1175.5108833312988, 1161.651849746704, 1195.3318119049072, 1162.4629497528076, 1156.1191082000732, 1127.0110607147217, 1165.4651165008545, 1159.6789360046387, 1190.3491020202637]

Median time (milliseconds):
1163.69903088
~~~

이런 접근법은 쿼리의 시간을 2489.85ms(서브쿼리를 포함하는 쿼리)에서 2202.70ms(2개의 고유 쿼리)로 줄였습니다.

### 7단계. 쓰기 성능 테스트/조정

- [기존 테이블로 대량 삽입](#bulk-inserting-into-an-existing-table)
- [사용하지 않는 인덱스 최소화](#minimizing-unused-indexes)
- [새로 삽입한 행의 ID 반환](#retrieving-the-id-of-a-newly-inserted-row)

#### 기존 테이블로 대량 삽입

쓰기로 넘어가서, 100명의 새로운 사용자를 `users`테이블에 넣는다고 가정해 봅시다. 가장 분명한 접근법은 100개의 [`INSERT`](insert.html) 문을 별도로 사용하여 각 행을 삽입하는 것입니다. 


{{site.data.alerts.callout_info}}
시연을 위해 아래 명령은 각각 다른 고유 ID를 가진 동일한 사용자를 100회 삽입합니다. 총 시간을 출력하기 위해서는 `--cumulative` 플래그를 추가하면 됩니다.
{{site.data.alerts.end}}

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="INSERT INTO users VALUES (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347')" \
--repeat=100 \
--times \
--cumulative
~~~

~~~
Times (milliseconds):
[10.773181915283203, 12.186050415039062, 9.711980819702148, 9.730815887451172, 10.200977325439453, 9.32002067565918, 9.002923965454102, 9.426116943359375, 9.312152862548828, 8.329153060913086, 9.626150131225586, 8.965015411376953, 9.562969207763672, 9.305000305175781, 9.34910774230957, 7.394075393676758, 9.3231201171875, 9.066104888916016, 8.419036865234375, 9.158134460449219, 9.278059005737305, 8.022069931030273, 8.542060852050781, 9.237051010131836, 8.165121078491211, 8.094072341918945, 8.025884628295898, 8.04591178894043, 9.728193283081055, 8.485078811645508, 7.967948913574219, 9.319067001342773, 8.099079132080078, 9.041070938110352, 10.046005249023438, 10.684013366699219, 9.672880172729492, 8.129119873046875, 8.10098648071289, 7.884979248046875, 9.484052658081055, 8.594036102294922, 9.479045867919922, 9.239912033081055, 9.16600227355957, 9.155988693237305, 9.392976760864258, 11.08694076538086, 9.402990341186523, 8.034944534301758, 8.053064346313477, 8.03995132446289, 8.891820907592773, 8.054971694946289, 8.903980255126953, 9.057998657226562, 9.713888168334961, 7.99107551574707, 8.114814758300781, 8.677959442138672, 11.178970336914062, 9.272098541259766, 9.281158447265625, 8.177995681762695, 9.47880744934082, 10.025978088378906, 8.352041244506836, 8.320808410644531, 10.892868041992188, 8.227825164794922, 8.220911026000977, 9.625911712646484, 10.272026062011719, 8.116960525512695, 10.786771774291992, 9.073972702026367, 9.686946868896484, 9.903192520141602, 9.887933731079102, 9.399890899658203, 9.413003921508789, 8.594036102294922, 8.433103561401367, 9.271860122680664, 8.529901504516602, 9.474992752075195, 9.005069732666016, 9.341001510620117, 9.388923645019531, 9.775876998901367, 8.558988571166992, 9.613990783691406, 8.897066116333008, 8.642911911010742, 9.527206420898438, 8.274078369140625, 9.073972702026367, 9.637832641601562, 8.516788482666016, 9.564876556396484]

Median time (milliseconds):
9.20152664185

Cumulative time (milliseconds):
910.985708237
~~~

100개의 삽입 작업은 910.98ms가 걸렸고, 이것은 그렇게 나쁜 결과는 아닙니다. 하지만, 다음과 같이 쉼표로 구분된 100개의 `VALUES` 조항과 함께 하나의 `INSERT` 문을 사용하면 훨씬 더 빨라집니다.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="INSERT INTO users VALUES \
(gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), \
(gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), \
(gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), \
(gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), \
(gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), \
(gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), \
(gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), \
(gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), \
(gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), \
(gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347'), (gen_random_uuid(), 'new york', 'Max Roach', '411 Drum Street', '173635282937347')" \
--repeat=1 \
--cumulative
~~~

~~~
Median time (milliseconds):
15.4001712799

Cumulative time (milliseconds):
15.4001712799
~~~

보시다시피 다중 행 `INSERT`  기법은 100개의 삽입 시간을 910.98ms에서 15.40ms로 단축시켰습니다. 이 기법은 [`UPSERT`](upsert.html)와 [`DELETE`](delete.html)문에도 동일하게 효과적이라는 점을 기억해 두시면 좋습니다.

#### 사용되지 않는 인덱스 최소화

앞에서 우리는 보조 인덱스가 읽기 성능에 얼마나 중요한지 보았습니다. 그러나 쓰기의 경우 보조 인덱스가 생성하는 오버헤드를 인식하는 것이 중요합니다.

`users` 테이블에서 이를 살펴봅시다:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="SHOW INDEXES FROM users;"
~~~

~~~
  table_name |   index_name   | non_unique | seq_in_index | column_name | direction | storing | implicit
+------------+----------------+------------+--------------+-------------+-----------+---------+----------+
  users      | primary        |   false    |            1 | city        | ASC       |  false  |  false
  users      | primary        |   false    |            2 | id          | ASC       |  false  |  false
  users      | users_name_idx |    true    |            1 | name        | ASC       |  false  |  false
  users      | users_name_idx |    true    |            2 | credit_card | N/A       |  true   |  false
  users      | users_name_idx |    true    |            3 | city        | ASC       |  false  |   true
  users      | users_name_idx |    true    |            4 | id          | ASC       |  false  |   true
(6 rows)
~~~

이 테이블은 기본 인덱스(전체 테이블)와 `credit_card` 또한 저장하고 있는 `name`에 대한 보조 인덱스를 가지고 있습니다. 이는 기존 행에 행이 삽입되거나 `name`, `credit_card`, `city`,가 수정될 때마다 두 인덱스가 모두 업데이트됨을 의미합니다.

더 구체적으로 보기 위해, 이름이 `C`로 시작하는 행의 수를 세어보고 그 행들을 모두 같은 이름으로 업데이트해봅시다.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT count(*) \
FROM users \
WHERE name LIKE 'C%'" \
--repeat=1
~~~

~~~
['count']
['179']

Median time (milliseconds):
2.52604484558
~~~

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="UPDATE users \
SET name = 'Carl Kimball' \
WHERE name LIKE 'C%'" \
--repeat=1
~~~

~~~
Median time (milliseconds):
52.2060394287
~~~

`name`(이름)은 `primary`(기본) 와 `users_name_idx` 인덱스에 모두 포함돼 있어 168행 각각에 대해 2개의 키가 업데이트됩니다.

이제 `users_name_idx` 인덱스가 더 이상 필요하지 않다고 가정하고, 인덱스를 삭제한 후 동일한 쿼리를 실행하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="DROP INDEX users_name_idx;"
~~~

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="UPDATE users \
SET name = 'Peedie Hirata' \
WHERE name = 'Carl Kimball'" \
--repeat=1
~~~

~~~
Median time (milliseconds):
22.7289199829
~~~

기본 인덱스 및 보조 인덱스를 모두 업데이트해야 할 때 업데이트에 52.20ms가 소요되었으나, 보조 인덱스를 삭제한 후에는 동일한 업데이트가 22.72ms 밖에 걸리지 않았음을 확인할 수 있습니다.

#### 새로 삽입한 행의 ID 반환

이제 표에 행을 삽입한 다음 새 행의 ID를 반환하여 후속 작업을 수행하는 일반적인 사례에 초점을 맞춰봅시다. 이를 위한 한 가지 방법은 행을 삽입하는 `INSERT`와 새 ID를 얻기 위한 `SELECT` 라는 두 개의 문을 실행하는 것입니다.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="INSERT INTO users VALUES (gen_random_uuid(), 'new york', 'Toni Brooks', '800 Camden Lane, Brooklyn, NY 11218', '98244843845134960')" \
--repeat=1
~~~

~~~
Median time (milliseconds):
10.4398727417
~~~

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="SELECT id FROM users WHERE name = 'Toni Brooks'" \
--repeat=1
~~~

~~~
Result:
['id']
['ae563e17-ad59-4307-a99e-191e682b4278']

Median time (milliseconds):
5.53798675537
~~~

이 두 문구는 합쳐서 15.96ms로 비교적 빠르게 실행되지만, 훨씬 더 효과적인 접근법은 `INSERT` 끝에 `RETURNING id`를 추가하는 것입니다.

{% include copy-clipboard.html %}
~~~ shell
$ ./tuning.py \
--host=<address of any node> \
--statement="INSERT INTO users VALUES (gen_random_uuid(), 'new york', 'Brian Brooks', '800 Camden Lane, Brooklyn, NY 11218', '98244843845134960') \
RETURNING id" \
--repeat=1
~~~

~~~
Result:
['id']
['3d16500e-cb2e-462e-9c83-db0965d6deaf']

Median time (milliseconds):
9.48596000671
~~~

단 9.48ms로 실행되는 이 접근법은 두 번의 클라이언트-서버 라운드트립을 하지 않고 한 번에 쓰기 및 읽기를 실행할 수 있어 더 빠릅니다. 하지만 앞에서 논의한 바와 같이 테이블에 대한 임대자가 쿼리가 실행되는 노드와 다른 노드에 있는 경우에는 추가 네트워크 홉과 대기 시간이 추가된다는 점에 유의하십시오.

<!-- - upsert instead of insert/update
- update using case expressions (instead of 2 separate updates)
- returning nothing
- insert with returning (auto gen ID) instead of select to get auto gen ID
- Maybe interleaved tables -->

## 다중 영역 배포

<!-- roachprod instructions for multi-region deployment
You created all instanced up front, so no need to add more now.
1. Start the nodes in us-west1-a: roachprod start -b "/usr/local/bin/cockroach" <yourname>-tuning:5-7
2. Start the nodes in us-west2-a: roachprod start -b "/usr/local/bin/cockroach" <yourname>-tuning:9-11
3. Install the Python client on instance 8:
   - SSH to instance 8: roachprod run <yourname>-tuning:8
   - Run commands in Step 5 above.
4. Install the Python client on instance 12:
   - SSH onto instance 12: roachprod run <yourname>-tuning:12
   - Run commands in Step 5 above.
5. Check rebalancing:
   - SSH to instance 4, 8, or 12.
   - Run `SHOW EXPERIMENTAL_RANGES` from Step 11 below.
6. Test performance:
   - Run the SQL commands in Step 12 below. You'll need to SSH to instance 8 or 12 as suggested.
7. Partition the data:
   - SSH to any node and run the SQL in Step 13 below.
8. Check rebalancing after partitioning:
   - SSH to instance 4, 8, or 12.
   - Run `SHOW EXPERIMENTAL_RANGES` from Step 14 below.
8. Test performance after partitioning:
   - Run the SQL commands in Step 15 below. You'll need to SSH to instance 8 or 12 as suggested.
-->

Movr가 두 미국 연안에서 모두 사용되려면, 클러스터는 각각 3개의 노드로 구성되고 지역 클라이언트 트래픽을 시뮬레이션하는 추가 인스턴스가 있는 두 개의 새로운 지역인 `us-west1-a` and `us-west2-a`로 확장되어야 합니다.

### 8단계. 인스턴스 추가 생성

1. [6개 인스턴스 추가 생성](https://cloud.google.com/compute/docs/instances/create-start-instance), `us-west1-a`영역(오리건)에 3개,`us-west2-a` 영역(로스앤젤레스)에 3개. 
각 인스턴스에 대하여:
    - `n1-standard-4`(4 vCPUs, 15 GB 메모리) 컴퓨터 타입 사용
    - Ubuntu 16.04 OS 이미지 사용
    - [로컬 SSD 생성 및 마운트](https://cloud.google.com/compute/docs/disks/local-ssd#create_local_ssd).
    - 앞서 생성한 웹 UI 방화벽 규칙을 적용하려면 **Management, disk, networking, SSH keys** 를 클릭하고, **Networking** 탭을 선택한 다음 **Network tags** 필드에 `cockroachdb` 를 입력하십시오.

2.각 `n1-standard-4` 인스턴스의 내부 IP 주소를 기록해 두십시오. 이 주소는 CockroachDB 노드를 시작할 때 필요합니다.

3. us-west1-a` 와 `us-west2-a` 영역에서 추가 인스턴스를 만드십시오. 이 인스턴스는 `n1-standard-1`처럼 더 작아도 됩니다.

### 9단계. 클러스터 확장

1. `us-west1-a` 영역의 `n1-standard-4` 인스턴스 중 하나에 대해 SSH

2. [CockroachDB 아카이브](https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz)를 리눅스용으로 다운로드하고, 바이너리를 추출하여 `PATH`에 복사:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget -qO- https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz \
    | tar  xvz
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo cp -i cockroach-{{ page.release_info.version }}.linux-amd64/cockroach /usr/local/bin
    ~~~

3. [`cockroach start`](start-a-node.html) 명령어 실행:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --advertise-host=<node internal address> \
    --join=<same as earlier> \
    --locality=cloud=gce,region=us-west1,zone=us-west1-a \
    --cache=.25 \
    --max-sql-memory=.25 \
    --background
    ~~~

4. `us-west1-a` zone.`us-west1-a` 영역의 다른 두 `n1-standard-4` 인스턴스에 대해 1 - 3번을 반복

5. `us-west2-a` 지역의 `n1-standard-4` 인스턴스 중 하나에 대해 SSH.

6. [CockroachDB 아카이브](https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz)를 리눅스용으로 다운로드하고, 바이너리를 추출하여 `PATH`에 복사:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget -qO- https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz \
    | tar  xvz
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ sudo cp -i cockroach-{{ page.release_info.version }}.linux-amd64/cockroach /usr/local/bin
    ~~~

7. [`cockroach start`](start-a-node.html) 명령어 실행:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --advertise-host=<node1 internal address> \
    --join=<same as earlier> \
    --locality=cloud=gce,region=us-west2,zone=us-west2-a \
    --cache=.25 \
    --max-sql-memory=.25 \
    --background
    ~~~

8. `us-west2-a` 영역의 다른 두 `n1-standard-4` 인스턴스에 대해 5 - 7번을 반복
 
### 10단계. Python 클라이언트 설치

각 새 영역에서 CockroachDB 노드를 실행하지 않는 인스턴스에 SSH 하고 위의 [5단계](#step-5-install-the-python-client)에 설명된 대로 Python 클라이언트를 설치하십시오.

### 11단계. 재조정 확인

각 노드를 GCE 영역에 설정된  `--locality`플래그로 시작했으므로 다음 몇 분 동안 CockroachDB는 데이터를 전체 영역에 걸쳐 균등하게 재조정할 것입니다.

이를 확인하려면 `<node address>:8080`의 임의의 노드에 있는 웹 UI에 액세스하여 **Node List**를 참조하십시오. 모든 노드의 범위 카운트가 어느 정도인지 볼 수 있습니다.

<img src="{{ 'images/v2.1/perf_tuning_multi_region_rebalancing.png' | relative_url }}" alt="Perf tuning rebalancing" style="border:1px solid #eee;max-width:100%" />

참고: 영역과 노드

Node IDs | Zone
---------|-----
1-3 | `us-east1-b` (South Carolina)
4-6 | `us-west1-a` (Oregon)
7-9 | `us-west2-a` (Los Angeles)

범위 수준에서 균등한 균형을 확인하려면 CockroachDB를 실행하지 않는 인스턴스 중 하나에 SSH를 실행하고 `SHOW EXPERIMENTAL_RANGES`문을 실행하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="SHOW EXPERIMENTAL_RANGES FROM TABLE vehicles;"
~~~

~~~
  start_key | end_key | range_id | replicas | lease_holder
+-----------+---------+----------+----------+--------------+
  NULL      | NULL    |       33 | {3,4,7}  |            7
(1 row)
~~~

이때 `vehicles` 데이터를 포함하는 단일 범위의 경우 각 영역마다 복제본 1개가 있고, 임대자는 `us-west2-a` 영역에 있는 것을 볼 수 있습니다.

### Step 12. 성능 테스트

일반적으로 위의 단일 영역 시나리오에 포함된 모든 튜닝 기법은 다중 영역에도 적용됩니다. 그러나, 다중 영역에서 많은 경우 데이터와 임대자가 미국 전역에 분산되기 때문에 이는 더 긴 지연시간의 원인이 됩니다.

#### 읽기

예를 들어, 우리가 뉴욕의 Movr 관리자라고 가정하고, 현재 사용 중인 모든 뉴욕 기반 자전거의 ID 및 정보를 얻고 싶다고 가정해 보십시오.

1. Python 클라이언트가 있는 `us-east1-b`에서 해당 인스턴스에 대한 SSH.

2. 데이터에 대한 쿼리:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./tuning.py \
    --host=<address of a node in us-east1-b> \
    --statement="SELECT id, ext FROM vehicles \
    WHERE city = 'new york' \
        AND type = 'bike' \
        AND status = 'in_use'" \
    --repeat=50 \
    --times
    ~~~

    ~~~
    Result:
    ['id', 'ext']
    ['0068ee24-2dfb-437d-9a5d-22bb742d519e', "{u'color': u'green', u'brand': u'Kona'}"]
    ['01b80764-283b-4232-8961-a8d6a4121a08', "{u'color': u'green', u'brand': u'Pinarello'}"]
    ['02a39628-a911-4450-b8c0-237865546f7f', "{u'color': u'black', u'brand': u'Schwinn'}"]
    ['02eb2a12-f465-4575-85f8-a4b77be14c54', "{u'color': u'black', u'brand': u'Pinarello'}"]
    ['02f2fcc3-fea6-4849-a3a0-dc60480fa6c2', "{u'color': u'red', u'brand': u'FujiCervelo'}"]
    ['034d42cf-741f-428c-bbbb-e31820c68588', "{u'color': u'yellow', u'brand': u'Santa Cruz'}"]
    ...

    Times (milliseconds):
    [933.8209629058838, 72.02410697937012, 72.45206832885742, 72.39294052124023, 72.8158950805664, 72.07584381103516, 72.21412658691406, 71.96712493896484, 71.75517082214355, 72.16811180114746, 71.78592681884766, 72.91603088378906, 71.91109657287598, 71.4719295501709, 72.40676879882812, 71.8080997467041, 71.84004783630371, 71.98500633239746, 72.40891456604004, 73.75001907348633, 71.45905494689941, 71.53081893920898, 71.46596908569336, 72.07608222961426, 71.94995880126953, 71.41804695129395, 71.29096984863281, 72.11899757385254, 71.63381576538086, 71.3050365447998, 71.83194160461426, 71.20394706726074, 70.9981918334961, 72.79205322265625, 72.63493537902832, 72.15285301208496, 71.8698501586914, 72.30591773986816, 71.53582572937012, 72.69001007080078, 72.03006744384766, 72.56317138671875, 71.61688804626465, 72.17121124267578, 70.20092010498047, 72.12018966674805, 73.34589958190918, 73.01592826843262, 71.49410247802734, 72.19099998474121]

    Median time (milliseconds):
    72.0270872116
    ~~~

앞에서 본 대로, `vehicles` 표에 대한 임대주는 `us-west2-a` (로스앤젤레스),에 있으므로 우리의 쿼리는 `us-east1-b`에 있는 게이트웨이 노드에서 출발해 서해안까지 갔다가 다시 돌아와야 클라이언트에게 데이터를 반환할 수 있습니다.

반대로, 우리가 현재 로스엔젤레스의 Movr 관리자로서, 현재 사용되고 있는 모든 로스엔젤레스 기반 자전거의 ID 및 정보를 얻고 싶다고 가정해 보십시오.

1. Python 클라이언트가 있는 `us-west2-a`에서 해당 인스턴스에 대한 SSH.

2. 데이터에 대한 쿼리:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./tuning.py \
    --host=<address of a node in us-west2-a> \
    --statement="SELECT id, ext FROM vehicles \
    WHERE city = 'los angeles' \
        AND type = 'bike' \
        AND status = 'in_use'" \
    --repeat=50 \
    --times
    ~~~

    ~~~
    Result:
    ['id', 'ext']
    ['00078349-94d4-43e6-92be-8b0d1ac7ee9f', "{u'color': u'blue', u'brand': u'Merida'}"]
    ['003f84c4-fa14-47b2-92d4-35a3dddd2d75', "{u'color': u'red', u'brand': u'Kona'}"]
    ['0107a133-7762-4392-b1d9-496eb30ee5f9', "{u'color': u'yellow', u'brand': u'Kona'}"]
    ['0144498b-4c4f-4036-8465-93a6bea502a3', "{u'color': u'blue', u'brand': u'Pinarello'}"]
    ['01476004-fb10-4201-9e56-aadeb427f98a', "{u'color': u'black', u'brand': u'Merida'}"]

    Times (milliseconds):
    [782.6759815216064, 8.564949035644531, 8.226156234741211, 7.949113845825195, 7.86590576171875, 7.842063903808594, 7.674932479858398, 7.555961608886719, 7.642984390258789, 8.024930953979492, 7.717132568359375, 8.46409797668457, 7.520914077758789, 7.6541900634765625, 7.458925247192383, 7.671833038330078, 7.740020751953125, 7.771015167236328, 7.598161697387695, 8.411169052124023, 7.408857345581055, 7.469892501831055, 7.524967193603516, 7.764101028442383, 7.750988006591797, 7.2460174560546875, 6.927967071533203, 7.822990417480469, 7.27391242980957, 7.730960845947266, 7.4710845947265625, 7.4310302734375, 7.33494758605957, 7.455110549926758, 7.021188735961914, 7.083892822265625, 7.812976837158203, 7.625102996826172, 7.447957992553711, 7.179021835327148, 7.504940032958984, 7.224082946777344, 7.257938385009766, 7.714986801147461, 7.4939727783203125, 7.6160430908203125, 7.578849792480469, 7.890939712524414, 7.546901702880859, 7.411956787109375]

    Median time (milliseconds):
    7.6071023941
    ~~~

`vehicles` 에 대한 임대자가 클라이언트의 요청과 동일한 영역에 있기 때문에 72.02ms가 걸린 뉴욕의 쿼리에 비해 7.60ms밖에 걸리지 않습니다.

#### 쓰기

데이터의 지리적 분포는 쓰기 성능에도 영향을 미칩니다. 예를 들어, 시애틀에 있는 100명의 사람들과 뉴욕에 있는 100명의 사람들이 새로운 Movr 계정을 만들고 싶다고 가정해 보십시오.

1. Python 클라이언트가 있는 `us-west1-a` 에서 해당 인스턴스에 대한 SSH.

2. 시애틀 기반 사용자 100명 생성:

    {% include copy-clipboard.html %}
    ~~~ shell
    ./tuning.py \
    --host=<address of a node in us-west1-a> \
    --statement="INSERT INTO users VALUES (gen_random_uuid(), 'seattle', 'Seatller', '111 East Street', '1736352379937347')" \
    --repeat=100 \
    --times
    ~~~

    ~~~
    Times (milliseconds):
    [277.4538993835449, 50.12702941894531, 47.75214195251465, 48.13408851623535, 47.872066497802734, 48.65407943725586, 47.78695106506348, 49.14689064025879, 52.770137786865234, 49.00097846984863, 48.68602752685547, 47.387123107910156, 47.36208915710449, 47.6841926574707, 46.49209976196289, 47.06096649169922, 46.753883361816406, 46.304941177368164, 48.90894889831543, 48.63715171813965, 48.37393760681152, 49.23295974731445, 50.13418197631836, 48.310041427612305, 48.57516288757324, 47.62911796569824, 47.77693748474121, 47.505855560302734, 47.89996147155762, 49.79205131530762, 50.76479911804199, 50.21500587463379, 48.73299598693848, 47.55592346191406, 47.35088348388672, 46.7071533203125, 43.00808906555176, 43.1060791015625, 46.02813720703125, 47.91092872619629, 68.71294975280762, 49.241065979003906, 48.9039421081543, 47.82295227050781, 48.26998710632324, 47.631025314331055, 64.51892852783203, 48.12812805175781, 67.33417510986328, 48.603057861328125, 50.31013488769531, 51.02396011352539, 51.45716667175293, 50.85396766662598, 49.07512664794922, 47.49894142150879, 44.67201232910156, 43.827056884765625, 44.412851333618164, 46.69189453125, 49.55601692199707, 49.16882514953613, 49.88598823547363, 49.31306838989258, 46.875, 46.69594764709473, 48.31886291503906, 48.378944396972656, 49.0570068359375, 49.417972564697266, 48.22111129760742, 50.662994384765625, 50.58097839355469, 75.44088363647461, 51.05400085449219, 50.85110664367676, 48.187971115112305, 56.7781925201416, 42.47403144836426, 46.2191104888916, 53.96890640258789, 46.697139739990234, 48.99096488952637, 49.1330623626709, 46.34690284729004, 47.09315299987793, 46.39410972595215, 46.51689529418945, 47.58000373840332, 47.924041748046875, 48.426151275634766, 50.22597312927246, 50.1859188079834, 50.37498474121094, 49.861907958984375, 51.477909088134766, 73.09293746948242, 48.779964447021484, 45.13692855834961, 42.2968864440918]

    Median time (milliseconds):
    48.4025478363
    ~~~

3. 클라이언트가 있는 `us-east1-b`에서 해당 인스턴스에 대한 SSH.

4. 뉴욕 기반 사용자 100명 생성:

    {% include copy-clipboard.html %}
    ~~~ shell
    ./tuning.py \
    --host=<address of a node in us-east1-b> \
    --statement="INSERT INTO users VALUES (gen_random_uuid(), 'new york', 'New Yorker', '111 West Street', '9822222379937347')" \
    --repeat=100 \
    --times
    ~~~

    ~~~
    Times (milliseconds):
    [131.05082511901855, 116.88899993896484, 115.15498161315918, 117.095947265625, 121.04082107543945, 115.8750057220459, 113.80696296691895, 113.05880546569824, 118.41201782226562, 125.30899047851562, 117.5389289855957, 115.23890495300293, 116.84799194335938, 120.0411319732666, 115.62800407409668, 115.08989334106445, 113.37089538574219, 115.15498161315918, 115.96989631652832, 133.1961154937744, 114.25995826721191, 118.09396743774414, 122.24102020263672, 116.14608764648438, 114.80998992919922, 131.9139003753662, 114.54391479492188, 115.15307426452637, 116.7759895324707, 135.10799407958984, 117.18511581420898, 120.15485763549805, 118.0570125579834, 114.52388763427734, 115.28396606445312, 130.00011444091797, 126.45292282104492, 142.69423484802246, 117.60401725769043, 134.08493995666504, 117.47002601623535, 115.75007438659668, 117.98381805419922, 115.83089828491211, 114.88890647888184, 113.23404312133789, 121.1700439453125, 117.84791946411133, 115.35286903381348, 115.0820255279541, 116.99700355529785, 116.67394638061523, 116.1041259765625, 114.67289924621582, 112.98894882202148, 117.1119213104248, 119.78602409362793, 114.57300186157227, 129.58717346191406, 118.37983131408691, 126.68204307556152, 118.30306053161621, 113.27195167541504, 114.22920227050781, 115.80777168273926, 116.81294441223145, 114.76683616638184, 115.1430606842041, 117.29192733764648, 118.24417114257812, 116.56999588012695, 113.8620376586914, 114.88819122314453, 120.80597877502441, 132.39002227783203, 131.00910186767578, 114.56179618835449, 117.03896522521973, 117.72680282592773, 115.6010627746582, 115.27681350708008, 114.52317237854004, 114.87483978271484, 117.78903007507324, 116.65701866149902, 122.6949691772461, 117.65193939208984, 120.5449104309082, 115.61179161071777, 117.54202842712402, 114.70890045166016, 113.58809471130371, 129.7171115875244, 117.57993698120117, 117.1119213104248, 117.64001846313477, 140.66505432128906, 136.41691207885742, 116.24789237976074, 115.19908905029297]

    Median time (milliseconds):
    116.868495941
    ~~~

시애틀에서 사용자를 만드는 데 48.40ms, 뉴욕에서 사용자를 만드는 데 116.86ms가 소요되었습니다. 이러한 차이를 더 잘 이해하기 위해 `users` 테이블에 대한 데이터 분포를 살펴봅시다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="SHOW EXPERIMENTAL_RANGES FROM TABLE users;"
~~~

~~~
  start_key | end_key | range_id | replicas | lease_holder
+-----------+---------+----------+----------+--------------+
  NULL      | NULL    |       49 | {2,6,8}  |            6
(1 row)
~~~

데이터를 포함하는 단일 범위의 경우, `us-west1-a` 영역에 있는 임대자와 함께 각 영역에 복제본 1개가 있습니다. 
이는 다음을 의미합니다:

- 시애틀에서 사용자를 생성할 때, 해당 요청은 임대자에게 도달하기 위해 영역을 벗어날 필요가 없습니다. 그러나 쓰기는 복제본 그룹의 합의를 필요로 하기 때문에, 쓰기는 `us-west1-b` (로스앤젤레스) 또는 `us-east1-b` (뉴욕)의 복제본으로부터의 확인을 기다렸다가 커밋한 후 클라이언트에게 확인을 반환해야 합니다.
- 뉴욕에서 유저 생성시 네트워크 홉이 많아져 대기시간이 늘어납니다. 우선 그 요청은 대륙을 건너서 임대주가 있는 `us-west1-a`로 가야 합니다. 그리고 커밋하기 전에 `us-west1-b` (로스앤젤레스) 또는 `us-east1-b` (뉴욕)의 복제본으로부터 확인을 기다렸다가 다시 동쪽에 있는 클라이언트에게 확인을 반환해야 합니다.

### 13단계. 도시별 데이터 분할

 이 서비스의 경우, 읽기 및 쓰기 대기 시간을 개선하는 가장 효과적인 기법은 도시별 데이터를 [geo-partition(지리적 분할)](partitioning.html)하는 것입니다. 본질적으로, 이것은 데이터가 범위에 매핑되는 방법을 바꾸는 것을 의미합니다. 특정 범위 또는 범위 집합에 매핑되는 전체 테이블과 그 색인 대신, 표의 모든 행과 인덱스의 모든 행은 범위 또는 범위 세트에 매핑됩니다. 범위를 정의한 후에는 [복제 영역](configure-replication-zones.html) 기능을 사용하여 파티션을 특정 위치에 고정함으로써 특정 도시의 사용자의 읽기 및 쓰기 요청이 해당 지역을 떠나지 않도록 할 수 있습니다.

1. 파티셔닝은 엔터프라이즈 기능이므로 [30일 평가판 license에 등록](https://www.cockroachlabs.com/get-cockroachdb/)부터 시작하십시오.


2. 평가판 라이센스를 받으면 클러스터의 임의의 노드에 SSH 하고 [라이센스 적용](enterprise-licensing.html#set-the-trial-or-enterprise-license-key)을 수행하십시오.


    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql \
    --insecure \
    --host=<address of any node> \
    --execute="SET CLUSTER SETTING cluster.organization = '<your org name>';"
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql \
    --insecure \
    --host=<address of any node> \
    --execute="SET CLUSTER SETTING enterprise.license = '<your license>';"
    ~~~

3. 모든 테이블에 대한 파티션 및 그에 대한 보조인덱스를 정의하십시오.

    `users` 테이블 부터 시작:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql \
    --insecure \
    --database=movr \
    --host=<address of any node> \
    --execute="ALTER TABLE users \
    PARTITION BY LIST (city) ( \
        PARTITION new_york VALUES IN ('new york'), \
        PARTITION boston VALUES IN ('boston'), \
        PARTITION washington_dc VALUES IN ('washington dc'), \
        PARTITION seattle VALUES IN ('seattle'), \
        PARTITION san_francisco VALUES IN ('san francisco'), \
        PARTITION los_angeles VALUES IN ('los angeles') \
    );"
    ~~~

    이제 `vehicles` 테이블에 대한 파티션과 그에 대한 보조 인덱스를 정의하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql \
    --insecure \
    --database=movr \
    --host=<address of any node> \
    --execute="ALTER TABLE vehicles \
    PARTITION BY LIST (city) ( \
        PARTITION new_york VALUES IN ('new york'), \
        PARTITION boston VALUES IN ('boston'), \
        PARTITION washington_dc VALUES IN ('washington dc'), \
        PARTITION seattle VALUES IN ('seattle'), \
        PARTITION san_francisco VALUES IN ('san francisco'), \
        PARTITION los_angeles VALUES IN ('los angeles') \
    );"
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql \
    --insecure \
    --database=movr \
    --host=<address of any node> \
    --execute="ALTER INDEX vehicles_auto_index_fk_city_ref_users \
    PARTITION BY LIST (city) ( \
        PARTITION new_york_idx VALUES IN ('new york'), \
        PARTITION boston_idx VALUES IN ('boston'), \
        PARTITION washington_dc_idx VALUES IN ('washington dc'), \
        PARTITION seattle_idx VALUES IN ('seattle'), \
        PARTITION san_francisco_idx VALUES IN ('san francisco'), \
        PARTITION los_angeles_idx VALUES IN ('los angeles') \
    );"
    ~~~

    다음으로 `rides` 테이블에 대한 파티션과 그에 대한 보조 인덱스를 정의하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql \
    --insecure \
    --database=movr \
    --host=<address of any node> \
    --execute="ALTER TABLE rides \
    PARTITION BY LIST (city) ( \
        PARTITION new_york VALUES IN ('new york'), \
        PARTITION boston VALUES IN ('boston'), \
        PARTITION washington_dc VALUES IN ('washington dc'), \
        PARTITION seattle VALUES IN ('seattle'), \
        PARTITION san_francisco VALUES IN ('san francisco'), \
        PARTITION los_angeles VALUES IN ('los angeles') \
    );"
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql \
    --insecure \
    --database=movr \
    --host=<address of any node> \
    --execute="ALTER INDEX rides_auto_index_fk_city_ref_users \
    PARTITION BY LIST (city) ( \
        PARTITION new_york_idx1 VALUES IN ('new york'), \
        PARTITION boston_idx1 VALUES IN ('boston'), \
        PARTITION washington_dc_idx1 VALUES IN ('washington dc'), \
        PARTITION seattle_idx1 VALUES IN ('seattle'), \
        PARTITION san_francisco_idx1 VALUES IN ('san francisco'), \
        PARTITION los_angeles_idx1 VALUES IN ('los angeles') \
    );"
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql \
    --insecure \
    --database=movr \
    --host=<address of any node> \
    --execute="ALTER INDEX rides_auto_index_fk_vehicle_city_ref_vehicles \
    PARTITION BY LIST (vehicle_city) ( \
        PARTITION new_york_idx2 VALUES IN ('new york'), \
        PARTITION boston_idx2 VALUES IN ('boston'), \
        PARTITION washington_dc_idx2 VALUES IN ('washington dc'), \
        PARTITION seattle_idx2 VALUES IN ('seattle'), \
        PARTITION san_francisco_idx2 VALUES IN ('san francisco'), \
        PARTITION los_angeles_idx2 VALUES IN ('los angeles') \
    );"
    ~~~

    마지막으로, `rides`에 있는 사용되지 않은 인덱스를 분할하지 않고 삭제하십시오.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql \
    --insecure \
    --database=movr \
    --host=<address of any node> \
    --execute="DROP INDEX rides_start_time_idx;"
    ~~~

    {{site.data.alerts.callout_info}}
    `rides` 테이블은 100만 행으로 되어 있어 이 인덱스를 떨어뜨리는 데는 몇 분 정도가 소요됩니다.
    {{site.data.alerts.end}}

7. 이제 [복제 영역 만들기](configure-replication-zones.html#create-a-replication-zone-for-a-table-or-secondary-index-partition)를 통해 노드 인접성을 기반으로 특정 노드에 도시 데이터를 저장해야 합니다.

    City | Locality
    -----|---------
    New York | `zone=us-east1-b`
    Boston | `zone=us-east1-b`
    Washington DC | `zone=us-east1-b`
    Seattle | `zone=us-west1-a`
    San Francisco | `zone=us-west2-a`
    Los Angelese | `zone=us-west2-a`

    {{site.data.alerts.callout_info}}
    우리의 노드는 3개의 특정 GCE 구역에 위치하고 있기 때문에, 우리는 노드 위치의 `zone=` 부분만을 사용합니다. 지역별로 여러 구역을 사용하는 경우에는 노드 인접성의 `region=`부분을 대신 사용할 수 있습니다.
    {{site.data.alerts.end}}

    Start with the `users` table partitions:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION new_york OF TABLE movr.users CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION boston OF TABLE movr.users CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION washington_dc OF TABLE movr.users CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION seattle OF TABLE movr.users CONFIGURE ZONE USING constraints='[+zone=us-west1-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION san_francisco OF TABLE movr.users CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION los_angeles OF TABLE movr.users CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    Move on to the `vehicles` table and secondary index partitions:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION new_york OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION new_york_idx OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION boston OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION boston_idx OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION washington_dc OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION washington_dc_idx OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION seattle OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-west1-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION seattle_idx OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-west1-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION san_francisco OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION san_francisco_idx OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION los_angeles OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION los_angeles_idx OF TABLE movr.vehicles CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    Finish with the `rides` table and secondary index partitions:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION new_york OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION new_york_idx1 OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION new_york_idx2 OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION boston OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION boston_idx1 OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION boston_idx2 OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION washington_dc OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION washington_dc_idx OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION washington_dc_idx2 OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-east1-b]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION seattle OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-west1-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION seattle_idx1 OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-west1-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION seattle_idx2 OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-west1-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION san_francisco OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION san_francisco_idx1 OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION san_francisco_idx2 OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION los_angeles OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION los_angeles_idx1 OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --execute="ALTER PARTITION los_angeles_idx2 OF TABLE movr.rides CONFIGURE ZONE USING constraints='[+zone=us-west2-a]';" \
    --insecure \
    --host=<address of any node>
    ~~~

### Step 14. 분할 후 재조정 확인

다음 몇 분 동안, CockroachDB는 사용자가 정의한 제약조건에 따라 모든 파티션의 균형을 재조정할 것입니다.

이를 확인하려면 " `<node address>:8080`의 모든 노드에 있는 웹 UI에 접속하여 **Node List** 를 참조하십시오. 모든 노드에서 범위 카운트가 여전히 거의 균등하지만, 분할 전보다 훨씬 더 높은 것을 확인할 수 있습니다.

<img src="{{ 'images/v2.1/perf_tuning_multi_region_rebalancing_after_partitioning.png' | relative_url }}" alt="Perf tuning rebalancing" style="border:1px solid #eee;max-width:100%" />

보다 세분화된 수준에서 확인하기 위해 CockroachDB를 실행하지 않는 인스턴스 중 하나에 SSH를 수행하고 `vehicles` 테이블에 `SHOW EXPERIMENTAL_RANGES` 문을 실행하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--insecure \
--host=<address of any node> \
--database=movr \
--execute="SELECT * FROM \
[SHOW EXPERIMENTAL_RANGES FROM TABLE vehicles] \
WHERE \"start_key\" IS NOT NULL \
    AND \"start_key\" NOT LIKE '%Prefix%';"
~~~

~~~
     start_key     |          end_key           | range_id | replicas | lease_holder
+------------------+----------------------------+----------+----------+--------------+
  /"boston"        | /"boston"/PrefixEnd        |      105 | {1,2,3}  |            3
  /"los angeles"   | /"los angeles"/PrefixEnd   |      121 | {7,8,9}  |            8
  /"new york"      | /"new york"/PrefixEnd      |      101 | {1,2,3}  |            3
  /"san francisco" | /"san francisco"/PrefixEnd |      117 | {7,8,9}  |            8
  /"seattle"       | /"seattle"/PrefixEnd       |      113 | {4,5,6}  |            5
  /"washington dc" | /"washington dc"/PrefixEnd |      109 | {1,2,3}  |            1
(6 rows)
~~~

참고: 노드가 영역에 매핑되는 방식

Node IDs | Zone
---------|-----
1-3 | `us-east1-b` (South Carolina)
4-6 | `us-west1-a` (Oregon)
7-9 | `us-west2-a` (Los Angeles)

분할 후 뉴욕, 보스턴, 워싱턴 DC의 복제본은 `us-east1-b`, 의 노드 1-3에 위치하며, 시애틀의 복제본은 `us-west1-a` 의 노드 4-6에 위치하며, 샌프란시스코와 로스엔젤레스 복제본은 `us-west2-a`의 노드 7-9에 위치하는 것을 볼 수 있습니다.

### Step 15. 분할 후 성능 테스트

특정 도시를 위한 모든 복제본이 현재 도시에서 가장 가까운 노드에 위치하기 때문에 분할을 한 후, 읽기 및 쓰기 속도가 훨씬 빨라질 것입니다. 

이를 확인하려면  [step 12](#step-12-test-performance).에서 분할하기 전에 실행한 몇 가지 읽기 및 쓰기 쿼리를 반복하십시오.

#### 읽기

우리가 뉴욕의 Movr 관리자로서, 현재 사용 중인 모든 뉴욕 기반 자전거의 ID와 정보를 얻고 싶다고 다시 가정해 보십시오.

1. Python 클라이언트와 함께 `us-east1-b` 에 있는 인스턴스에 대한 SSH.

2. 데이터에 대한 쿼리:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./tuning.py \
    --host=<address of a node in us-east1-b> \
    --statement="SELECT id, ext FROM vehicles \
    WHERE city = 'new york' \
        AND type = 'bike' \
        AND status = 'in_use'" \
    --repeat=50 \
    --times
    ~~~

    ~~~
    Result:
    ['id', 'ext']
    ['0068ee24-2dfb-437d-9a5d-22bb742d519e', "{u'color': u'green', u'brand': u'Kona'}"]
    ['01b80764-283b-4232-8961-a8d6a4121a08', "{u'color': u'green', u'brand': u'Pinarello'}"]
    ['02a39628-a911-4450-b8c0-237865546f7f', "{u'color': u'black', u'brand': u'Schwinn'}"]
    ['02eb2a12-f465-4575-85f8-a4b77be14c54', "{u'color': u'black', u'brand': u'Pinarello'}"]
    ['02f2fcc3-fea6-4849-a3a0-dc60480fa6c2', "{u'color': u'red', u'brand': u'FujiCervelo'}"]
    ['034d42cf-741f-428c-bbbb-e31820c68588', "{u'color': u'yellow', u'brand': u'Santa Cruz'}"]
    ...

    Times (milliseconds):
    [20.065784454345703, 7.866144180297852, 8.362054824829102, 9.08803939819336, 7.925987243652344, 7.543087005615234, 7.786035537719727, 8.227825164794922, 7.907867431640625, 7.654905319213867, 7.793903350830078, 7.627964019775391, 7.833957672119141, 7.858037948608398, 7.474184036254883, 9.459972381591797, 7.726192474365234, 7.194995880126953, 7.364034652709961, 7.25102424621582, 7.650852203369141, 7.663965225219727, 9.334087371826172, 7.810115814208984, 7.543087005615234, 7.134914398193359, 7.922887802124023, 7.220029830932617, 7.606029510498047, 7.208108901977539, 7.333993911743164, 7.464170455932617, 7.679939270019531, 7.436990737915039, 7.62486457824707, 7.235050201416016, 7.420063018798828, 7.795095443725586, 7.39598274230957, 7.546901702880859, 7.582187652587891, 7.9669952392578125, 7.418155670166016, 7.539033889770508, 7.805109024047852, 7.086992263793945, 7.069826126098633, 7.833957672119141, 7.43412971496582, 7.035017013549805]

    Median time (milliseconds):
    7.62641429901
    ~~~

분할 전에 이 쿼리는 72.02ms의 평균 시간이 걸렸으나, 분할 후 쿼리는 7.62ms의 평균 시간만 걸리는 것을 볼 수 있습니다.

#### 쓰기

자, 이제 뉴욕의 100명, 시애틀의 100명, 그리고 뉴욕의 100명이 새로운 Movr 계정을 만들고 싶어한다고 다시 가정해봅시다.

1. Python 클라이언트와 함께 `us-west1-a` 의 인스턴스에 대한 SSH.

2. 시애틀 기반 사용자 100명 생성:

    {% include copy-clipboard.html %}
    ~~~ shell
    ./tuning.py \
    --host=<address of a node in us-west1-a> \
    --statement="INSERT INTO users VALUES (gen_random_uuid(), 'seattle', 'Seatller', '111 East Street', '1736352379937347')" \
    --repeat=100 \
    --times
    ~~~

    ~~~
    Times (milliseconds):
    [41.8248176574707, 9.701967239379883, 8.725166320800781, 9.058952331542969, 7.819175720214844, 6.247997283935547, 10.265827178955078, 7.627964019775391, 9.120941162109375, 7.977008819580078, 9.247064590454102, 8.929967880249023, 9.610176086425781, 14.40286636352539, 8.588075637817383, 8.67319107055664, 9.417057037353516, 7.652044296264648, 8.917093276977539, 9.135961532592773, 8.604049682617188, 9.220123291015625, 7.578134536743164, 9.096860885620117, 8.942842483520508, 8.63790512084961, 7.722139358520508, 13.59701156616211, 9.176015853881836, 11.484146118164062, 9.212017059326172, 7.563114166259766, 8.793115615844727, 8.80289077758789, 7.827043533325195, 7.6389312744140625, 17.47584342956543, 9.436845779418945, 7.63392448425293, 8.594989776611328, 9.002208709716797, 8.93402099609375, 8.71896743774414, 8.76307487487793, 8.156061172485352, 8.729934692382812, 8.738040924072266, 8.25190544128418, 8.971929550170898, 7.460832595825195, 8.889198303222656, 8.45789909362793, 8.761167526245117, 10.223865509033203, 8.892059326171875, 8.961915969848633, 8.968114852905273, 7.750988006591797, 7.761955261230469, 9.199142456054688, 9.02700424194336, 9.509086608886719, 9.428977966308594, 7.902860641479492, 8.940935134887695, 8.615970611572266, 8.75401496887207, 7.906913757324219, 8.179187774658203, 11.447906494140625, 8.71419906616211, 9.202003479003906, 9.263038635253906, 9.089946746826172, 8.92496109008789, 10.32114028930664, 7.913827896118164, 9.464025497436523, 10.612010955810547, 8.78596305847168, 8.878946304321289, 7.575035095214844, 10.657072067260742, 8.777856826782227, 8.649110794067383, 9.012937545776367, 8.931875228881836, 9.31406021118164, 9.396076202392578, 8.908987045288086, 8.002996444702148, 9.089946746826172, 7.5588226318359375, 8.918046951293945, 12.117862701416016, 7.266998291015625, 8.074045181274414, 8.955001831054688, 8.868932723999023, 8.755922317504883]

    Median time (milliseconds):
    8.90052318573
    ~~~

   분할 전에 이 쿼리는 48.40ms의 평균 시간이 걸렸으나, 분할 후 쿼리는 8.90ms의 평균 시간만 걸리는 것을 볼 수 있습니다.

3. 클라이언트와 함께 `us-east1-b` 에서 인스턴스에 대한 SSH.

4. 신규 NY 기반 사용자 100명 생성:

    {% include copy-clipboard.html %}
    ~~~ shell
    ./tuning.py \
    --host=<address of a node in us-east1-b> \
    --statement="INSERT INTO users VALUES (gen_random_uuid(), 'new york', 'New Yorker', '111 West Street', '9822222379937347')" \
    --repeat=100 \
    --times
    ~~~

    ~~~
    Times (milliseconds):
    [276.3068675994873, 9.830951690673828, 8.772134780883789, 9.304046630859375, 8.24880599975586, 7.959842681884766, 7.848978042602539, 7.879018783569336, 7.754087448120117, 10.724067687988281, 13.960123062133789, 9.825944900512695, 9.60993766784668, 9.273052215576172, 9.41920280456543, 8.040904998779297, 16.484975814819336, 10.178089141845703, 8.322000503540039, 9.468793869018555, 8.002042770385742, 9.185075759887695, 9.54294204711914, 9.387016296386719, 9.676933288574219, 13.051986694335938, 9.506940841674805, 12.327909469604492, 10.377168655395508, 15.023946762084961, 9.985923767089844, 7.853031158447266, 9.43303108215332, 9.164094924926758, 10.941028594970703, 9.37199592590332, 12.359857559204102, 8.975028991699219, 7.728099822998047, 8.310079574584961, 9.792089462280273, 9.448051452636719, 8.057117462158203, 9.37795639038086, 9.753942489624023, 9.576082229614258, 8.192062377929688, 9.392023086547852, 7.97581672668457, 8.165121078491211, 9.660959243774414, 8.270978927612305, 9.901046752929688, 8.085966110229492, 10.581016540527344, 9.831905364990234, 7.883787155151367, 8.077859878540039, 8.161067962646484, 10.02812385559082, 7.9898834228515625, 9.840965270996094, 9.452104568481445, 9.747028350830078, 9.003162384033203, 9.206056594848633, 9.274005889892578, 7.8449249267578125, 8.827924728393555, 9.322881698608398, 12.08186149597168, 8.76307487487793, 8.353948593139648, 8.182048797607422, 7.736921310424805, 9.31406021118164, 9.263992309570312, 9.282112121582031, 7.823944091796875, 9.11712646484375, 8.099079132080078, 9.156942367553711, 8.363962173461914, 10.974884033203125, 8.729934692382812, 9.2620849609375, 9.27591323852539, 8.272886276245117, 8.25190544128418, 8.093118667602539, 9.259939193725586, 8.413076400756836, 8.198976516723633, 9.95182991027832, 8.024930953979492, 8.895158767700195, 8.243083953857422, 9.076833724975586, 9.994029998779297, 10.149955749511719]

    Median time (milliseconds):
    9.26303863525
    ~~~

    분할 전에 이 쿼리는 116.86ms의 평균 시간이 걸렸으나, 분할 후 쿼리는 9.26ms의 평균 시간이 걸리는 것을 볼 수 있습니다.

## 더 알아보기

- [SQL 성능 모범 사례](performance-best-practices-overview.html)
- [성능 벤치마킹](performance-benchmarking-with-tpc-c.html)
- [생산 ](recommended-production-settings.html)
