---
title: Follow-the-Workload
summary: CockroachDB can dynamically optimize read latency for the location from which most of the workload is originating.
toc: true
---

"팔로우 워크로드"는 CockroachDB가 대부분의 작업 부하가 발생하는 위치에 대한 읽기 대기 시간을 동적으로 최적화하는 기능을 나타냅니다.
이 페이지는 "팔로우 워크로드"가 어떻게 작동하는지를 설명하고 로컬 클러스터를 사용한 간단한 설명을 보여줍니다.


## 개요

### 기본 용어

"팔로우 워크로드" 작동 방식을 이해하려면 몇 가지 기본 용어로 시작하는 것이 중요합니다.

용어 | 설명
-----|------------
**Range** | CockroachDB는 모든 사용자 데이터와 거의 모든 시스템 데이터를 키 값 쌍의 정렬된 거대한 맵에 저장합니다. 이 키스페이스는 모든 범위가 항상 단일 범위에서 발견 될 수 있도록 키스페이스의 연속적인 덩어리인 "범위"로 구분됩니다.
**Range Replica** | CockroachDB는 각 범위를 복제(기본적으로 3번)하고 각 복제본을 다른 노드에 저장합니다.
**Range Lease** | 각 범위에 대해 복제본 중 하나가 "레인지 리스"를 보유합니다. 이 복제본은 "리스 홀더"라고 하며 범위에 대한 모든 읽기 및 쓰기 요청을 받아서 조정합니다.


### 작동 원리

"팔로우 워크로드"는 **레인지 리스**에서 읽기 요청을 처리하는 방식을 기반으로 합니다. 읽기 요청은 레인지 리스(리스홀더)를 보유한 범위 복제본에 액세스하고 다른 범위 복제본과 조정할 필요없이 클라이언트에 결과를 보내는 래프트 프로토콜을 우회합니다. 모든 기록 요청이 리스홀더에게 전달된다는 사실 때문에 리스홀더가 최신 상태로 보장되기에 래프트 우회 및 네트워크 왕복 이동이 가능합니다.

이렇게하면 읽기 속도가 빨라지지만, 레인지 리스가 요청의 원점 근처에 있을 것이라는 보장은 없습니다. 예를 들어, 요청이 미국 동부에서 오는 경우, 해당 레인지 리스가 미국 동부의 한 노드에 있는 경우에 요청은 미국 서부의 게이트웨이 노드로 들어간 다음 미국 동부의 레인지 리스가 있는 노드로 전송됩니다. 

그러나 클러스터가 [`--locality`](start-a-node.html#locality) 플래그로 각 노드를 시작하여 더 나은 읽기 성능을 위해 레인지 리스를 적극적으로 이동하게할 수 있습니다. 이 플래그가 지정되면, 클러스터는 각 노드의 위치를 알고 있으므로, 노드간에 대기 시간이 길면 클러스터는 활성 레인지 리스를 대다수의 워크로드의 근원에 가까운 노드로 이동합니다. 이는 하루 종일 이동하는 워크로드가 있는 응용 프로그램 (예 : 대부분의 트래픽이 오전에 미국 동부에, 저녁에 미국 서부에 있는 경우)에 유용합니다.
 
{{site.data.alerts.callout_success}}"팔로우 워크로드"를 사용하려면 아래의 튜토리얼에 표시된대로 <code>--locality</code> 플래그를 사용하여 클러스터의 각 노드를 시작하기만 하면 됩니다. 추가 사용자 액션이 요구되지 않습니다.{{site.data.alerts.end}}



### 예시

이 예시에서는 많은 읽기 요청이 노드 1로 가고, 요청이 범위 3의 데이터에 대한 것이라고 가정해 보겠습니다. 범위 3의 리스가 노드 3에 있기 때문에 요청은 노드 3으로 전송되고 결과는 노드 1에 반환됩니다. 노드 1은 클라이언트에 응답합니다.


<img src="{{ 'images/v2.1/follow-workload-1.png' | relative_url }}" alt="팔로우 워크로드 예시" style="max-width:100%" />


그러나 노드가 [`--locality`](start-a-node.html#locality) 플래그로 시작된 경우, 잠시 후, 클러스터는 범위 3의 리스를 워크로드의 근원에 가까운 노드 1에 옮겨 놓음으로써 네트워크 왕복 트립을 줄이고 읽기 속도를 높일 수 있습니다.


<img src="{{ 'images/v2.1/follow-workload-2.png' | relative_url }}" alt="팔로우 " style="max-width:100%" />

## 튜토리얼

### 1단계. 필수 구성 요소 설치

이 튜토리얼에서는, 로컬 워크 스테이션의 네트워크 대기 시간을 시뮬레이션하는 `comcast` 네트워크 도구인 CockroachDB와 클라이언트 작업 부하를 시뮬레이트하는 `kv`로드 생성기를 사용합니다. 시작하기 전에 다음 응용 프로그램이 설치되어 있는지 확인하십시오: 


- 최신 버전의 [CockroachDB](install-cockroachdb.html)를 설치하십시오.
- [Go](https://golang.org/doc/install) 버전 1.9 이상을 설치하십시오. Mac을 사용하고 Homebrew를 사용한다면 `brew install go`를 사용하십시오. `go version`을 실행하여 로컬 버전을 확인할 수 있습니다.
- [`comcast`](https://github.com/tylertreat/comcast) 네트워크 시뮬레이션 도구를 설치하십시오: `go get github.com/tylertreat/comcast`
- [`kv`](https://github.com/cockroachdb/loadgen/tree/master/kv) 로드 생성기를 설치하십시오: `go get github.com/cockroachdb/loadgen/kv`
또한 클러스터의 데이터 파일과 로그를 추적하려면 새 디렉토리 (예 :`mkdir follow-workload`)를 만들고 해당 디렉토리의 모든 노드를 시작해야 할 수 있습니다.


### 2단계. 네트워크 대기 시간 시뮬레이션 시작

"팔로우 워크로드"는 CockroachDB 클러스터의 노드 사이에 대기 시간이 많은 경우에만 시작됩니다. 이 튜토리얼에서는 로컬 워크 스테이션에서 3개의 노드를 실행하고 각 노드는 US의 다른 영역에 있는 것으로 가장합니다. 노드 사이의 대기 시간을 시뮬레이션하려면, 앞에서 설치한 `comcast` 도구를 사용하십시오.

새로운 터미널에서 다음과 같이 `comcast`를 시작하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ comcast --device lo0 --latency 100
~~~

`--device` 플래그는 Mac에서는 `lo0`, 리눅스에서는 `lo`를 사용합니다. 둘 다 작동하지 않으면, `ifconfig` 명령을 실행하고 출력에서`127.0.0.1`을 담당하는 인터페이스를 찾으십시오. 


이 명령은 로컬 워크 스테이션의 루프백 인터페이스에서 모든 요청에 대해 0.1초 지연을 발생시킵니다.
이는 컴퓨터에서 컴퓨터로의 연결에만 영향을 미치며 인터넷과의 연결에는 영향을 미치지 않습니다.


### 3단계. 클러스터 시작하기

미국의 다른 지역에 있는 각 노드를 가장하는 플래그[`--locality`](start-a-node.html#locality)를 사용하여 로컬 워크 스테이션에서 3개의 노드를 시작하는 [`cockroach start`](start-a-node.html) 명령을 사용합니다.


1. 새 터미널에서 "미국 서부"노드를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --locality=region=us-west \
    --store=follow1 \
    --listen-addr=localhost:26257 \
    --http-addr=localhost:8080 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~

2. 새 터미널에서 "미국 중서부"노드를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --locality=region=us-midwest \
    --store=follow2 \
    --listen-addr=localhost:26258 \
    --http-addr=localhost:8081 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~

3. 새 터미널에서 "미국 동부"노드를 시작하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --locality=region=us-east \
    --store=follow3 \
    --listen-addr=localhost:26259 \
    --http-addr=localhost:8082 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~

### 4단계. 클러스터 초기화하기

새 터미널에서 [`cockroach init`](initialize-a-cluster.html) 명령을 사용하여 클러스터의 1회 초기화를 수행하십시오:


{% include copy-clipboard.html %}
~~~ shell
$ cockroach init \
--insecure \
--host=localhost:26257
~~~

### 5단계. 미국 동부의 교통 시뮬레이션

이제 클러스터가 활성화되었으므로, 앞에서 설치한 `kv`로드 생성기를 사용하여 "미국 동부"에있는 노드에 대한 여러 클라이언트 연결을 시뮬레이션합니다.

1. 새로운 터미널에서, `kv`를 시작하여 `us-east` 지역을 가진 노드의 포트인 `26259` 포트를 가리킵니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kv -duration 1m -concurrency 32 -read-percent 100 -max-rate 100 'postgresql://root@localhost:26259?sslmode=disable'
    ~~~


    이 명령은 32개의 동시 읽기 전용 워크로드를 1분 동안 시작하지만 전체 `kv` 프로세스를 초당 100개의 작업으로 제한합니다. (단일 시스템에서 모든 것을 실행하므로)
    
`kv`가 실행 중일 때 터미널에 몇 가지 통계를 출력합니다:


    ~~~
    _elapsed___errors__ops/sec(inst)___ops/sec(cum)__p95(ms)__p99(ms)_pMax(ms)
          1s        0           23.0           23.0    838.9    838.9    838.9
          2s        0          111.0           66.9    805.3    838.9    838.9
          3s        0          100.0           78.0    209.7    209.7    209.7
          4s        0           99.9           83.5    209.7    209.7    209.7
          5s        0          100.0           86.8    209.7    209.7    209.7
    ...
    ~~~

    {{site.data.alerts.callout_info}}<code>comcast</code> 도구로 인한 각 방향의 0.1초 지연 (왕복 200ms)은 지연은 <code>comcast</code> 프로세스에서 <code>cockroach</code> 프로세스로 가는 트래픽에도 적용되기 때문에 대기 시간은 0.2초가 넘습니다.{{site.data.alerts.end}}

2. 로드 생성기가 완료 될 때까지 실행하십시오.

### 6단계. 레인지 리스 위치 확인

로드 생성기는 기본 키-값 범위에 매핑되는 `kv` 테이블을 생성했습니다. 레인지의 리스가 다음과 같이 "미국 동부"의 노드로 이동했는지 확인하십시오.


1. 새로운 터미널에서 어떤 노드에 대해서나 [`cockroach 노드 상태`](view-node-details.html) 명령을 실행하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach node status --insecure --host=localhost:26257
    ~~~

    ~~~
    id |     address     |                build                 |            started_at            |            updated_at            | is_available | is_live  
+----+-----------------+--------------------------------------+----------------------------------+----------------------------------+--------------+---------+
     1 | localhost:26257 | v2.1.0-beta.20180924-176-ge486be24c9 | 2018-09-27 15:16:33.922153+00:00 | 2018-09-27 15:20:01.67638+00:00  | true         | true
     2 | localhost:26258 | v2.1.0-beta.20180924-176-ge486be24c9 | 2018-09-27 15:16:33.894595+00:00 | 2018-09-27 15:20:01.676513+00:00 | true         | true           
     3 | localhost:26259 | v2.1.0-beta.20180924-176-ge486be24c9 | 2018-09-27 15:16:39.067892+00:00 | 2018-09-27 15:19:57.638609+00:00 | true         | true     
(3 rows)
    ~~~

2. 응답으로, 포트 `26259`에서 실행중인 노드의 ID를 기록하십시오.

3. 동일한 터미널에서, [빌트인 SQL 쉘](use-the-built-in-sql-client.html)을 모든 노드에 연결하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure --host=localhost:26257
    ~~~

4. `kv` 테이블에 대한 레인지 리스를 확인하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW EXPERIMENTAL_RANGES FROM TABLE test.kv;
    ~~~

    ~~~
    +-----------+---------+----------+----------+--------------+
    | start_key | end_key | range_id | replicas | lease_holder |
    +-----------+---------+----------+----------+--------------+    
    | NULL      | NULL    | 16       | {1,2,3}  |            3 |
    +-----------+---------+----------+----------|--------------+
    (1 row)
    ~~~

    `레플리카`와 `리스 홀더`은 노드 ID를 나타냅니다. 보시다시피,`kv` 테이블의 데이터를 보유하고있는 범위에 대한 리스는 노드 3에 있으며, 이는 포트 `26259`에 있는 노드와 동일한 ID입니다.


### 7단계. 미국 서부의 교통 시뮬레이션

1. 같은 터미널에서 `kv`를 시작하여 `미국 서부` 지역을 가진 노드의 포트인 `26257` 포트를 가리킵니다:

 
    {% include copy-clipboard.html %}
    ~~~ shell
    $ kv -duration 7m -concurrency 32 -read-percent 100 -max-rate 100 'postgresql://root@localhost:26257?sslmode=disable'
    ~~~

   이번에는 명령이 1분이 아닌 7분 동안 조금 더 오래 실행됩니다. 이것은 시스템이 여전히 다른 지역에 대한 이전 요청을 "기억"하기 때문에 필요합니다.

2. 로드 생성기가 완료될 때까지 실행하십시오.


### 8단계. 레인지 리스 위치 확인

레인지의 리스가 다음과 같이 "미국 서부"의 노드로 이동했는지 확인하십시오.


1. 같은 터미널에서, 모든 노드에 대해 [`cockroach 노드 상태`](view-node-details.html) 명령을 실행합니다:


    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach node status --insecure --host=localhost:26257
    ~~~

    ~~~
    id |      address      |                 build                 |            started_at            |            updated_at            | is_available | is_live  
+----+-------------------+---------------------------------------+----------------------------------+----------------------------------+--------------+---------+
     1 | localhost:26257   | v2.1.0-beta.20180924-176-ge486be24c9  | 2018-09-27 15:16:33.922153+00:00 | 2018-09-27 15:37:03.20778+00:00  | true         | true        
     2 | localhost:26258   | v2.1.0-beta.20180924-176-ge486be24c9  | 2018-09-27 15:16:33.894595+00:00 | 2018-09-27 15:37:03.208422+00:00 | true         | true     
     3 | localhost:26259   | v2.1.0-beta.20180924-176-ge486be24c9  | 2018-09-27 15:16:39.067892+00:00 | 2018-09-27 15:37:03.663409+00:00 | true         | true     
(4 rows)
    ~~~

2. 응답으로, 포트 `26257`에서 실행중인 노드의 ID를 기록하십시오.


3. [빌트인 SQL 쉘](use-the-built-sql-client.html)을 모든 노드에 연결하십시오:


    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure --host=localhost:26257
    ~~~

4. `kv` 테이블에 대한 레인지 리스를 확인하십시오 :


    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW EXPERIMENTAL_RANGES FROM TABLE test.kv;
    ~~~

    ~~~
    +-----------+---------+----------+----------+--------------+
    | start_key | end_key | range_id | replicas | lease_holder |
    +-----------+---------+----------+----------+--------------+    
    | NULL      | NULL    | 16       | {1,2,3}  |            1 |
    +-----------+---------+----------+----------|--------------+
    (1 row)
    ~~~

    보시다시피,`kv` 테이블의 데이터를 보유하고 있는 레인지의 리스는 이제 노드 1에 있으며, 이는 포트 `26257`에있는 노드와 동일한 ID입니다.

### 9단계. 클러스터 중지하기

클러스터 작업이 끝나면, 각 노드의 터미널에서 **CTRL-C**를 누릅니다.

{{site.data.alerts.callout_success}}마지막 노드의 경우 종료 프로세스가 오래 걸리며 (약 1분) 노드가 강제 종료됩니다. 이는 여전히 온라인 상태인 노드가 하나뿐이기 때문에, 대부분의 복제본을 사용할 수 없으므로(3분의 2), 클러스터가 작동하지 않기 때문입니다. 프로세스 속도를 높이려면 <strong>CTRL-C</strong>키를 두 번 누릅니다.{{site.data.alerts.end}}

클러스터를 다시 시작하지 않으려는 경우, 노드의 데이터 저장소를 제거하고 싶을 것입니다:

{% include copy-clipboard.html %}
~~~ shell
$ rm -rf follow1 follow2 follow3
~~~

### Step 10. 네트워크 대기 시간 시뮬레이션 중지

이 튜토리얼을 끝내고 나면, 로컬 워크 스테이션의 모든 요청에 대해 0.1초의 지연을 원하지 않으므로, `comcast`도구를 중지하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ comcast --device lo0 --stop
~~~

## 더 알아보기

다른 핵심 CockroachDB 혜택 및 기능을 탐색하십시오:


{% include {{ page.version.version }}/misc/explore-benefits-see-also.md %}
