---
title: Define Table Partitions
summary: Partitioning is an enterprise feature that gives you row-level control of how and where your data is stored.
toc: true
---

CockroachDB를 사용하면 테이블 파티션을 정의할 수 있으므로, 데이터를 저장하는 방법과 위치를 행 수준에서 제어할 수 있습니다. 파티셔닝을 사용하면 대기 시간과 비용을 줄일 수 있으며 데이터에 대한 규정 요구 사항을 충족하는 데 도움이 됩니다.

{{site.data.alerts.callout_info}}
테이블 파티셔닝은 [엔터프라이즈 전용](enterprise-licensing.html) 기능입니다.
{{site.data.alerts.end}}

## 테이블 파티셔닝을 사용하는 이유

테이블 파티셔닝을 통해 대기 시간과 비용을 줄일 수 있습니다: 

- **지리적-파티셔닝**에서는 사용자 데이터를 사용자 가까이에 두어 데이터가 이동해야하는 거리를 줄여주므로 **대기 시간 감소**가 가능합니다. 테이블을 지리적으로 분할하려면, 테이블를 작성하는 동안 위치-기반 분할 영역을 정의하고, 위치별 영역 구성을 생성하고, 해당 분할 영역에 영역 구성을 적용하십시오.
- **아카이브-파티셔닝**를 사용하면 자주 사용하지 않는 데이터를 더 느리고 저렴한 스토리지에 저장할 수 있으므로 **비용이 절감**됩니다. 테이블을 아카이브-분할하려면, 테이블을 생성하는 동안 빈도-기반 파티션을 정의하고, 적절한 저장 장치 제약 조건을 사용하여 주파수-별 영역 구성을 만들고, 해당 파티션에 영역 구성을 적용하십시오.



## 작동 원리

테이블 파티셔닝은 CockroachDB 기능의 조합을 포함합니다:

- [노드 속성](#node-attributes)
- [엔터프라이즈 라이센스](#enterprise-license)
- [테이블 생성](#table-creation)
- [복제 영역](#replication-zones)

### 노드 속성

특정 위치 (예: 지리적-파티셔닝) 또는 특정 속성이 있는 머신(예: 아카이브-파티셔닝)에 파티션을 저장하려면, 클러스터의 노드가 관련 플래그와 함께 [시작](start-a-node.html)되어야 합니다:

- `--locality` 플래그를 사용하여 노드의 위치를 설명하는 키-값 쌍을 할당하십시오, 예를 들어, `--locality=region=east,datacenter=us-east-1`
- 특수한 하드웨어나 코어의 수를 포함할 수 있는 노드 용량을 지정하려면 `--attrs` 플래그를 사용하십시오, 예를 들어, `--attrs=ram:64gb`
- `--store` 플래그의 `attrs` 필드를 사용하여 디스크 유형 또는 기능을 지정하십시오, 예를 들어, `--store=path=/mnt/ssd01,attrs=ssd`.

이러한 플래그에 대한 자세한 내용은, [`cockroach start`](start-a-node.html) 설명서를 참조하십시오.

### 엔터프라이즈 라이센스

테이블 파티셔닝 기능을 사용하려면 유효한 엔터프라이즈 라이센스가 있어야 합니다. 평가판 또는 전체 엔터프라이즈 라이센스 요청 및 설정에 대한 자세한 내용은, [엔터프라이즈 라이센싱](enterprise-licensing.html)을 보십시오. 

**만료된 라이센스**에서는 다음 기능이 작동하지 않습니다:

- 새 테이블 파티션 생성 또는 파티션에 대한 새 영역 구성 추가
- 테이블 또는 인덱스에서 파티셔닝 계획 변경
- 파티션의 영역 구성 변경

그러나, 다음 기능은 만료된 엔터프라이즈 라이센스로도 계속 작동합니다:

- 분할된 테이블 쿼리 (예: `SELECT foo PARTITION`)
- 분할된 테이블에 데이터 삽입 또는 업데이트
- 분할된 테이블 삭제
- 분할된 테이블 분할 해제
- 분할된 테이블에 비-파티셔닝 변경 생성 (예: 열/인덱스/외래 키/검사 제약 조건 추가)

### 테이블 생성

테이블의 하나 이상의 열에 대해 파티션 및 하위 파티션을 정의할 수 있습니다. [테이블 생성](create-table.html) 중에, 다음 두 가지 방법 중 하나로 각 파티션에 속한 값을 선언합니다:

- **목록 파티셔닝**: 각 파티션에 대해 가능한 모든 값을 열거하십시오. 가능한 값의 수가 적은 경우 목록 파티셔닝이 좋은 선택입니다. 목록 파티셔닝은 지리적-파티셔닝에 매우 적합합니다.
- **범위 파티셔닝**: 상한 및 하한을 지정하여 각 파티션에 인접한 값 범위를 지정하십시오. 범위 파티셔닝은 가능한 값의 수가 너무 커서 명시적으로 나열할 수 없는 경우 좋은 선택입니다. 범위 파티셔닝은 아카이브-파티셔닝에 매우 적합합니다.

#### 목록 별 파티션

[`PARTITION BY LIST`](partition-by.html)를 사용하면 하나 이상의 튜플을 파티션에 매핑할 수 있습니다.

리스트로 테이블을 분할하려면, 테이블을 생성하는 동안 [`PARTITION BY LIST`](partition-by.html) 구문을 사용하십시오. 목록 파티션을 정의하는 동안, 정의 된 파티션에 대한 요구 사항과 일치하는 행이 없을 경우, 포괄적인 역할을 하는 `DEFAULT` 파티션을 설정할 수도 있습니다.

자세한 내용은 아래의 [목록별 파티션](#partition-by-list) 예제를 참조하십시오.

#### 범위 별 파티션

[`PARTITION BY RANGE`](partition-by.html)은 튜플의 범위를 파티션에 매핑합니다.

범위로 테이블 파티션을 정의하려면, 테이블을 생성하는 동안 [`PARTITION BY RANGE`](partition-by.html) 구문을 사용하십시오.  범위 파티션을 정의하는 동안, CockroachDB-정의된 `MINVALUE` 및 `MAXVALUE` 매개 변수를 사용하여 범위의 하한 및 상한을 각각 정의할 수 있습니다.

{{site.data.alerts.callout_info}}범위 파티션의 하한은 포함되지만, 상한은 배타적입니다. 범위 파티션의 경우, <code>NULL</code>은 키 인코딩 순서 및 <code>ORDER BY</code> 동작과 일치하는 다른 데이터보다 작은 것으로 간주됩니다.{{site.data.alerts.end}}

파티션 값은 모든 SQL 표현식이 될 수 있지만, 한 번만 평가됩니다. 2017-01-30 에 `<(now () - '1d')`값으로 파티션을 생성하면, 2017-01-29보다 작은 모든 값을 포함하게 됩니다. 다음 날에는 업데이트되지 않으며, 계속해서 2017-01-29보다 작은 값을 포함합니다.

자세한 내용은 아래의 [범위별 파티션](#partition-by-range) 예제를 참조하십시오.

#### 기본 키를 사용하여 파티션하기

파티셔닝에 필요한 기본 키는 기존의 기본 키와 다릅니다. 파티셔닝의 기본 키를 정의하려면, 기본 키의 고유 식별자 앞에 분할할 모든 열을 붙이고 하위 분할 영역을 중첩하려는 순서대로 테이블을 하위 분할하십시오.

예를 들어, 전 세계의 모든 코스 학생들을 대상으로 한 테이블이 있는 글로벌 온라인 학습 포털의 데이터베이스를 생각해보십시오. 학생 국가를 기반으로 테이블을 지역-파티션으로 지정하려면, 기본 키를 다음과 같이 정의해야 합니다:

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE students (
    id INT DEFAULT unique_rowid(),
    name STRING,
    email STRING,
    country STRING,
    expected_graduation_date DATE,   
    PRIMARY KEY (country, id));
~~~

**기본 키 고려 사항**

- v2.0의 경우, 테이블을 생성한 후에는 기본 키를 변경할 수 없습니다. 기본 키에 해당 열을 포함시켜 이후의 모든 하위 파티션에 대해 공급하십시오. 온라인 학습 포털의 예에서, 미래에 `expected_graduation_date`를 기반으로 하위 분할을 할 수 있다고 생각되면, 기본 키를 `(country, expected_graduation_date, id)`로 정의하십시오. v2.1에서는 기본 키를 변경할 수 있습니다.
- 기본 키에 열이 정의되는 순서가 중요합니다. 파티션과 하위 파티션은 이 순서대로 따라야 합니다. 온라인 학습 포털의 예에서, 기본 키를 `(country, expected_graduation_date, id)`로 정의하면, 기본 파티션은`country`이고, 하위 파티션은 `expected_graduation_date`입니다. `country`를 건너뛰고 `expected_graduation_date`으로 분할할 수는 없다.

#### 보조 인덱스를 사용하여 파티션하기

위에서 설명한 기본 키에는 두 가지 단점이 있습니다:

- 식별자 열이 전역적으로 고유하다는 것을 강요하지 않습니다.
- 식별자에 대한 빠른 검색을 제공하지 않습니다.

고유성 또는 빠른 검색을 보장하려면, 식별자에 파티션되지 않은 고유한 보조 인덱스를 생성합니다.

인덱스도 파티션될 수 있지만, 필수는 아닙니다. 각 파티션은 해당 테이블 및 해당 인덱스의 모든 파티션에서 고유한 이름을 가져야 합니다. 예를 들어, 다음 `CREATE INDEX` 시나리오는 기본 키의 파티션 이름을 다시 사용하기 때문에 실패합니다:

{% include copy-clipboard.html %}
~~~ sql
CREATE TABLE foo (a STRING PRIMARY KEY, b STRING) PARTITION BY LIST (a) (
    bar VALUES IN ('bar'),
    default VALUES IN (DEFAULT)
);
~~~

{% include copy-clipboard.html %}
~~~ sql
CREATE INDEX foo_b_idx ON foo (b) PARTITION BY LIST (b) (
    baz VALUES IN ('baz'),
    default VALUES IN (DEFAULT)
);
~~~

충돌을 피하기 위해 인덱스 이름을 사용하는 명명 체계를 사용해보십시오. 예를 들어, 위의 파티션은 `primary_idx_bar`,`primary_idx_default`,`b_idx_baz`,`b_idx_default`로 명명될 수 있습니다.

#### 인터리브된 테이블에 파티션 정의

[인터리브된 테이블](interleave-in-parent.html)의 경우, 파티션은 인터리브 계층 구조의 루트 테이블에서만 정의될 수 있으며 자식은 부모와 같은 인터리브로 나타납니다.

### 복제 영역

자체적으로, 파티션은 비활성이며 정의된 파티션의 기준을 충족시키는 테이블의 행에 레이블을 적용하기만 하면 됩니다. 파티션에 기능을 적용하려면, [복제 영역](configure-replication-zones.html)을 생성하여 해당 파티션에 적용해야 합니다.

CockroachDB는 사용 가능한 가장 세부적인 영역 구성을 사용합니다. 파티션을 대상으로 하는 영역 구성은 테이블 또는 인덱스를 대상으로 하는 영역 구성보다 세분화 된 것으로 간주되며, 테이블 또는 인덱스는 데이터베이스를 대상으로 하는 구성보다 세부적인 것으로 간주됩니다.

## 예제

### 목록별로 테이블 파티션 정의

글로벌 온라인 학습 포털인 RoachLearn을 고려해보십시오, 이 데이터베이스는 전세계 학생들의 테이블을 포함하고 있습니다. 미국에 하나, 호주에 하나, 두 개의 데이터 센터가 있다고 가정해 보겠습니다. 대기 시간을 줄이기 위해, 학생들의 데이터를 자신의 위치에 더 가깝게 유지하려고 합니다:

- 우리는 미국과 캐나다에 위치한 학생들의 데이터를 미국 데이터 센터에 보관하고자 합니다.
- 우리는 호주와 뉴질랜드에 있는 학생들의 데이터를 호주 데이터 센터에 보관하고자 합니다.

#### 1단계. 분할 방법 식별

우리는 학생들의 데이터를 자신의 위치에 더 가깝게 유지하기 위해 표를 지리적으로 분할하려고 합니다. 우리는 `country`를 분할하고 `PARTITION BY LIST` 구문을 사용하여 이를 달성할 수 있습니다.

#### 2단계. `--locality` 플래그로 지정된 데이터 센터 위치로 각 노드를 시작

{% include copy-clipboard.html %}
~~~ shell
# Start the node in the US datacenter:
$ cockroach start \
--insecure \
--locality=region=us1  \
--advertise-addr=<node1 hostname> \
--join=<node1 hostname>,<node2 hostname>
~~~

{% include copy-clipboard.html %}
~~~ shell
# Start the node in the AUS datacenter:
$ cockroach start \
--insecure \
--locality=region=aus1 \
--advertise-addr=<node2 hostname> \
--join=<node1 hostname>,<node2 hostname>
~~~

{% include copy-clipboard.html %}
~~~ shell
# Initialize the cluster:
$ cockroach init \
--insecure \
--host=<address of any node>
~~~

#### 3단계. 엔터프라이즈 라이센스 설정

엔터프라이즈 라이센스를 설정하려면, [평가판 또는 엔터프라이즈 라이센스 키 설정](enterprise-licensing.html#set-the-trial-or-enterprise-license-key)을 보십시오. 

#### 4단계. 적절한 파티션이 있는 테이블 생성

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE students_by_list (
    id INT DEFAULT unique_rowid(),
    name STRING,
    email STRING,
    country STRING,
    expected_graduation_date DATE,   
    PRIMARY KEY (country, id))
    PARTITION BY LIST (country)
      (PARTITION north_america VALUES IN ('CA','US'),
      PARTITION australia VALUES IN ('AU','NZ'),
      PARTITION DEFAULT VALUES IN (default));
~~~

#### 5단계. 해당 영역 구성 생성 및 적용

영역 구성을 생성하고 해당 파티션에 적용하려면, [`ALTER PARTITION ... CONFIGURE ZONE`](configure-zone.html) 명령문을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> ALTER PARTITION north_america OF TABLE roachlearn.students_by_list \
    CONFIGURE ZONE USING constraints='[+datacenter=us1]';
~~~

{% include copy-clipboard.html %}
~~~ sql
> ALTER PARTITION australia OF TABLE roachlearn.students_by_list \
    CONFIGURE ZONE USING constraints='[+datacenter=au1]';
~~~

#### 6단계. 테이블 파티션 확인

{% include copy-clipboard.html %}
~~~ sql
> SHOW EXPERIMENTAL_RANGES FROM TABLE students_by_list;
~~~

다음과 같은 결과가 나옵니다:

~~~
+-----------------+-----------------+----------+----------+--------------+
| start_key       | end_key         | range_id | replicas | lease_holder |
+-----------------+-----------------+----------+----------+--------------+
| NULL            | /"AU"           |      251 | {1,2,3}  |            1 |
| /"AU"           | /"AU"/PrefixEnd |      257 | {1,2,3}  |            1 |
| /"AU"/PrefixEnd | /"CA"           |      258 | {1,2,3}  |            1 |
| /"CA"           | /"CA"/PrefixEnd |      252 | {1,2,3}  |            1 |
| /"CA"/PrefixEnd | /"NZ"           |      253 | {1,2,3}  |            1 |
| /"NZ"           | /"NZ"/PrefixEnd |      256 | {1,2,3}  |            1 |
| /"NZ"/PrefixEnd | /"US"           |      259 | {1,2,3}  |            1 |
| /"US"           | /"US"/PrefixEnd |      254 | {1,2,3}  |            1 |
| /"US"/PrefixEnd | NULL            |      255 | {1,2,3}  |            1 |
+-----------------+-----------------+----------+----------+--------------+
(9 rows)

Time: 7.209032ms
~~~

### 범위 별로 테이블 파티션 정의

현재 학생의 데이터를 빠르고 값 비싼 저장 장치 (예 : SSD)에 저장하고 졸업 한 학생의 데이터를 더 느리고 저렴한 저장 장치 (예 : HDD)에 저장하려고 한다고 가정합니다.

#### 1단계. 분할 방법 식별

우리는 더 빠른 장치에 새로운 데이터를 저장하고 더 느린 장치에 이전 데이터를 보관할 수 있도록 테이블을 아카이브-분할하고자 합니다. 테이블을 날짜별로 분할하고 `PARTITION BY RANGE` 구문을 사용하여 이 작업을 수행 할 수 있습니다.

#### 2단계. 엔터프라이즈 라이센스 설정

엔터프라이즈 라이센스를 설정하려면, [평가판 또는 엔터프라이즈 라이센스 키 설정](enterprise-licensing.html#set-the-trial-or-enterprise-license-key)을 보십시오. 

#### 3단계. `--store` 플래그로 지정된 적절한 저장소 디바이스로 각 노드를 시작

{% include copy-clipboard.html %}
~~~ shell
# Start the first node:
$ cockroach start --insecure \
--store=path=/mnt/1,attrs=ssd \
--advertise-addr=<node1 hostname> \
--join=<node1 hostname>,<node2 hostname>
~~~

{% include copy-clipboard.html %}
~~~ shell
# Start the first node:
$ cockroach start --insecure \
--store=path=/mnt/2,attrs=hdd \
--advertise-addr=<node2 hostname> \
--join=<node1 hostname>,<node2 hostname>
~~~

{% include copy-clipboard.html %}
~~~ shell
# Initialize the cluster:
$ cockroach init \
--insecure \
--host=<address of any node>
~~~

#### 4단계. 적절한 파티션이 있는 테이블 생성

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE students_by_range (
   id INT DEFAULT unique_rowid(),
   name STRING,
   email STRING,                                                                                           
   country STRING,
   expected_graduation_date DATE,                                                                                      
   PRIMARY KEY (expected_graduation_date, id))
   PARTITION BY RANGE (expected_graduation_date)
      (PARTITION graduated VALUES FROM (MINVALUE) TO ('2017-08-15'),
      PARTITION current VALUES FROM ('2017-08-15') TO (MAXVALUE));
~~~

#### 5단계. 해당 영역 구성 생성 및 적용

영역 설정을 생성하고 해당 파티션에 적용하려면, [`ALTER PARTITION ... CONFIGURE ZONE`](configure-zone.html) 명령문을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> ALTER PARTITION current OF TABLE roachlearn.students_by_range \
    CONFIGURE ZONE USING constraints='[+ssd]';
~~~

{% include copy-clipboard.html %}
~~~ sql
> ALTER PARTITION graduated OF TABLE roachlearn.students_by_range \
    CONFIGURE ZONE USING constraints='[+hdd]';
~~~

#### 6단계. 테이블 파티션 확인

{% include copy-clipboard.html %}
~~~ sql
> SHOW EXPERIMENTAL_RANGES FROM TABLE students_by_range;
~~~

다음과 같은 결과가 나옵니다:

~~~
+-----------+---------+----------+----------+--------------+
| start_key | end_key | range_id | replicas | lease_holder |
+-----------+---------+----------+----------+--------------+
| NULL      | /17393  |      244 | {1,2,3}  |            1 |
| /17393    | NULL    |      242 | {1,2,3}  |            1 |
+-----------+---------+----------+----------+--------------+
(2 rows)

Time: 5.850903ms
~~~

### 테이블에 하위 파티션 정의

목록 파티션 자체를 분할하여, 하위 파티션을 구성할 수 있습니다. 하위 파티셔닝 레벨의 수에는 제한이 없습니다; 즉, 목록 파티션은 무한대로 중첩될 수 있습니다.

{{site.data.alerts.callout_info}}범위 파티션은 하위 파티션으로 분할할 수 없습니다.{{site.data.alerts.end}}

RoachLearn의 시나리오로 돌아가서, 우리가 다음을 모두 수행하기를 원한다고 가정해 보십시오:

- 학생들의 데이터를 자신의 위치와 가깝게 유지하십시오.
- 현재 학생들의 데이터를 보다 빠른 저장 장치에 저장하십시오.
- 졸업한 학생들의 데이터를 더 느리고 저렴한 저장 장치 (예 : HDD)에 저장하십시오.

#### 1단계. 분할 방법 식별

우리는 아카이브-파티션 테이블뿐만 아니라 지리적-파티션을 원합니다. 먼저 테이블을 위치별로 분할한 다음 날짜별로 분할하여 이 작업을 수행할 수 있습니다.

#### 2단계. `--store` 플래그로 지정된 적절한 저장소 디바이스로 각 노드를 시작

US 데이터 센터에서 노드 시작:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--advertise-addr=<node1 hostname> \
--locality=datacenter=us1 \
--store=path=/mnt/1,attrs=ssd \
--store=path=/mnt/2,attrs=hdd \
--join=<node1 hostname>,<node2 hostname>
~~~

AUS 데이터 센터에서 노드 시작:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--advertise-addr=<node2 hostname> \
--locality=datacenter=aus1 \
--store=path=/mnt/3,attrs=ssd \
--store=path=/mnt/4,attrs=hdd \
--join=<node1 hostname>,<node2 hostname>
~~~

클러스터 초기화:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach init --insecure --host=<address of any node>
~~~

#### 3단계. 엔터프라이즈 라이센스 설정

엔터프라이즈 라이센스를 설정하려면, [평가판 또는 엔터프라이즈 라이센스 키 설정](enterprise-licensing.html#set-the-trial-or-enterprise-license-key)을 보십시오. 

#### 4단계. 적절한 파티션이 있는 테이블 생성

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE students (
    id INT DEFAULT unique_rowid(),
    name STRING,
    email STRING,
    country STRING,
    expected_graduation_date DATE,
    PRIMARY KEY (country, expected_graduation_date, id))
    PARTITION BY LIST (country)(
        PARTITION australia VALUES IN ('AU','NZ') PARTITION BY RANGE (expected_graduation_date)(PARTITION graduated_au VALUES FROM (MINVALUE) TO ('2017-08-15'), PARTITION current_au VALUES FROM ('2017-08-15') TO (MAXVALUE)),
        PARTITION north_america VALUES IN ('US','CA') PARTITION BY RANGE (expected_graduation_date)(PARTITION graduated_us VALUES FROM (MINVALUE) TO ('2017-08-15'), PARTITION current_us VALUES FROM ('2017-08-15') TO (MAXVALUE))
    );
~~~

하위 분할 이름은 테이블 내에서 고유해야 합니다. 이 예에서는, `graduated`와 `current`가 구분된 파티션의 하위-파티션이지만, 여전히 고유한 이름을 지정해야 합니다. 따라서, `graduated_au`,`graduated_us`,`current_au`와`current_us`라는 이름이 있습니다.

#### 5단계. 해당 영역 구성 생성 및 적용

영역 구성을 생성하고 해당 파티션에 적용하려면, [`ALTER PARTITION ... CONFIGURE ZONE`](configure-zone.html) 명령문을 사용하십시오:

{% include copy-clipboard.html %}
~~~ sql
> ALTER PARTITION current_us OF TABLE roachlearn.students \
    CONFIGURE ZONE USING constraints='[+ssd,+datacenter=us1]';
~~~

{% include copy-clipboard.html %}
~~~ sql
> ALTER PARTITION graduated_us OF TABLE roachlearn.students CONFIGURE ZONE \
    USING constraints='[+hdd,+datacenter=us1]';
~~~

{% include copy-clipboard.html %}
~~~ sql
> ALTER PARTITION current_au OF TABLE roachlearn.students \
    CONFIGURE ZONE USING constraints='[+ssd,+datacenter=au1]';
~~~

{% include copy-clipboard.html %}
~~~ sql
> ALTER PARTITION graduated_au OF TABLE roachlearn.students CONFIGURE ZONE \
    USING constraints='[+hdd,+datacenter=au1]';
~~~

#### 6단계. 테이블 파티션 확인

{% include copy-clipboard.html %}
~~~ sql
> SHOW EXPERIMENTAL_RANGES FROM TABLE students;
~~~

다음과 같은 결과가 나옵니다:

~~~
+-----------------+-----------------+----------+----------+--------------+
| start_key       | end_key         | range_id | replicas | lease_holder |
+-----------------+-----------------+----------+----------+--------------+
| NULL            | /"AU"           |      260 | {1,2,3}  |            1 |
| /"AU"           | /"AU"/17393     |      268 | {1,2,3}  |            1 |
| /"AU"/17393     | /"AU"/PrefixEnd |      266 | {1,2,3}  |            1 |
| /"AU"/PrefixEnd | /"CA"           |      267 | {1,2,3}  |            1 |
| /"CA"           | /"CA"/17393     |      265 | {1,2,3}  |            1 |
| /"CA"/17393     | /"CA"/PrefixEnd |      261 | {1,2,3}  |            1 |
| /"CA"/PrefixEnd | /"NZ"           |      262 | {1,2,3}  |            3 |
| /"NZ"           | /"NZ"/17393     |      284 | {1,2,3}  |            3 |
| /"NZ"/17393     | /"NZ"/PrefixEnd |      282 | {1,2,3}  |            3 |
| /"NZ"/PrefixEnd | /"US"           |      283 | {1,2,3}  |            3 |
| /"US"           | /"US"/17393     |      281 | {1,2,3}  |            3 |
| /"US"/17393     | /"US"/PrefixEnd |      263 | {1,2,3}  |            1 |
| /"US"/PrefixEnd | NULL            |      264 | {1,2,3}  |            1 |
+-----------------+-----------------+----------+----------+--------------+
(13 rows)

Time: 11.586626ms
~~~

### 테이블 다시 분할

RoachLearn의 학생들의 분할된 테이블을 고려하십시오. 테이블이 현재의 학생들을 빠르고 비싼 저장 장치 (예 : SSD)에 저장하고 졸업한 학생의 데이터를 더 느리고 저렴한 저장 장치 (예 : HDD)에 저장하기 위해 범위를 분할했다고 가정합니다. 이제 우리는 학생들이 현재 `2018-08-15`로 간주될 날짜를 변경하고 싶다고 가정합니다. [`ALTER TABLE`](alter-table.html) 명령의 [`PARTITION BY`](partition-by.html) 부속 명령을 사용하여 이 작업을 수행할 수 있습니다.

{% include copy-clipboard.html %}
~~~ sql
> ALTER TABLE students_by_range PARTITION BY RANGE (expected_graduation_date) (
    PARTITION graduated VALUES FROM (MINVALUE) TO ('2018-08-15'),
    PARTITION current VALUES FROM ('2018-08-15') TO (MAXVALUE));
~~~

### 테이블 분할 해제

[`PARTITION BY NOTHING`](partition-by.html) 구문을 사용하여 테이블의 파티션을 제거할 수 있습니다:

{% include copy-clipboard.html %}
~~~ sql
> ALTER TABLE students PARTITION BY NOTHING;
~~~

## 지역성-탄력성 상충 관계

읽기/쓰기를 빠르게 하는 것과 생존 실패 사이에는 상충 관계가 있습니다. 호주 학생들을 위한 `roachlearn.students`의 3개의 복제본이 있는 파티션을 생각해보십시오.

- 호주 데이터 센터에 고정된 복제본이 하나 뿐인 경우, 읽기가 빠르지만 (리스가 워크로드를 따라 가면서) 읽기가 느려질 수 있습니다.
- 두 개의 복제본이 호주 데이터 센터에 고정되어 있으면, 읽기와 쓰기가 빨라집니다 (교차 대륙 링크의 대역폭이 충분하고 세 번째 복제본이 뒤떨어지지 않는 한). 두 개의 복제본이 동일한 데이터 센터에 있으면, 하나의 데이터 센터가 손실되어 데이터를 사용할 수 없게 될 수 있으므로, 일부 배포에서는 두 개의 별도 호주 데이터 센터가 필요할 수 있습니다.
- 3개의 복제본이 모두 호주 데이터 센터에 있으면, 3개의 호주 데이터 센터가 데이터 센터 손실에 복원력이 있어야 합니다.

## CockroachDB의 파티셔닝이 다른 데이터베이스와 어떻게 다른가

다른 데이터베이스는 세 가지 추가 사용 경우에 대해 파티셔닝을 사용합니다: 보조 인덱스, 샤딩 및 대량 로드/삭제. CockroachDB는 파티셔닝을 사용하지 않고 다음과 같은 방법으로 이러한 사용-경우를 해결합니다.

- **보조 인덱스로 변경:** CockroachDB는 온라인 계획 변경을 통해 이러한 변경 사항을 해결합니다. 온라인 계획 변경은 제로-다운타임을 필요로하고 일관성 문제의 가능성을 없애기 때문에 파티셔닝보다 우수한 기능입니다.
- **샤딩:** CockroachDB는 분산 데이터베이스 아키텍처의 일부로 데이터를 자동으로 파쇄합니다.
- **대량 로드 및 삭제:** CockroachDB에는 현재 이 사용 경우를 지원하는 기능이 없습니다.

## 알려진 제한 사항

{% include {{ page.version.version }}/known-limitations/partitioning-with-placeholders.md %}

## 더 보기

- [`CREATE TABLE`](create-table.html)
- [`ALTER TABLE`](alter-table.html)
- [계산된 열](computed-columns.html)
