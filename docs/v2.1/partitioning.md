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

- **지리적-파티셔닝** allows you to keep user data close to the user, which reduces the distance that the data needs to travel, thereby **대기 시간 감소**. To geo-partition a table, define location-based partitions while creating a table, create location-specific zone configurations, and apply the zone configurations to the corresponding partitions.
- **아카이브-파티셔닝** allows you to store infrequently-accessed data on slower and cheaper storage, thereby **reducing costs**. To archival-partition a table, define frequency-based partitions while creating a table, create frequency-specific zone configurations with appropriate storage devices constraints, and apply the zone configurations to the corresponding partitions.

## 작동 원리

테이블 파티셔닝은 CockroachDB 기능의 조합을 포함합니다:

- [노드 속성](#node-attributes)
- [엔터프라이즈 라이센스](#enterprise-license)
- [테이블 생성](#table-creation)
- [복제 영역](#replication-zones)

### 노드 속성

To store partitions in specific locations (e.g., geo-partitioning), or on machines with specific attributes (e.g., archival-partitioning), the nodes of your cluster must be [started](start-a-node.html) with the relevant flags:

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
- Making non-partitioning changes to a partitioned table 분할된 테이블에 비-파티셔닝 변경 생성 (예: 열/인덱스/외래 키/검사 제약 조건 추가)

### 테이블 생성

You can define partitions and subpartitions over one or more columns of a table. During [table creation](create-table.html), you declare which values belong to each partition in one of two ways:

- **목록 파티셔닝**: Enumerate all possible values for each partition. List partitioning is a good choice when the number of possible values is small. List partitioning is well-suited for geo-partitioning.
- **범위 파티셔닝**: Specify a contiguous range of values for each partition by specifying lower and upper bounds. Range partitioning is a good choice when the number of possible values is too large to explicitly list out. Range partitioning is well-suited for archival-partitioning.

#### 목록 별 파티션

[`PARTITION BY LIST`](partition-by.html)를 사용하면 하나 이상의 튜플을 파티션에 매핑할 수 있습니다.

To partition a table by list, use the [`PARTITION BY LIST`](partition-by.html) syntax while creating the table. While defining a list partition, you can also set the `DEFAULT` partition that acts as a catch-all if none of the rows match the requirements for the defined partitions.

자세한 내용은 아래의 [목록별 파티션](#partition-by-list) 예제를 참조하십시오.

#### 범위 별 파티션

[`PARTITION BY RANGE`](partition-by.html)은 튜플의 범위를 파티션에 매핑합니다.

범위로 테이블 파티션을 정의하려면, 테이블을 생성하는 동안 [`PARTITION BY RANGE`](partition-by.html) 구문을 사용하십시오.  범위 파티션을 정의하는 동안, CockroachDB-정의된 `MINVALUE` 및 `MAXVALUE` 매개 변수를 사용하여 범위의 하한 및 상한을 각각 정의할 수 있습니다.

{{site.data.alerts.callout_info}}범위 파티션의 하한은 포함되지만, 상한은 배타적입니다. For range partitions, <code>NULL</code> is considered less than any other data, which is consistent with our key encoding ordering and <code>ORDER BY</code> behavior.{{site.data.alerts.end}}

파티션 값은 모든 SQL 표현식이 될 수 있지만, 한 번만 평가됩니다. 2017-01-30 에 `<(now () - '1d')`값으로 파티션을 생성하면, 2017-01-29보다 작은 모든 값을 포함하게 됩니다. 다음 날에는 업데이트되지 않으며, 계속해서 2017-01-29보다 작은 값을 포함합니다.

자세한 내용은 아래의 [범위별 파티션](#partition-by-range) 예제를 참조하십시오.

#### 기본 키를 사용하여 파티션하기

파티셔닝에 필요한 기본 키는 기존의 기본 키와 다릅니다. To define the primary key for partitioning, prefix the unique identifier(s) in the primary key with all columns you want to partition and subpartition the table on, in the order in which you want to nest your subpartitions.

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

- For v2.0, you cannot change the primary key after you create the table. Provision for all future subpartitions by including those columns in the primary key. In the example of the online learning portal, if you think you might want to subpartition based on `expected_graduation_date` in the future, define the primary key as `(country, expected_graduation_date, id)`. v2.1 will allow you to change the primary key.
- The order in which the columns are defined in the primary key is important. The partitions and subpartitions need to follow that order. In the example of the online learning portal, if you define the primary key as `(country, expected_graduation_date, id)`, the primary partition is by `country`, and then subpartition is by `expected_graduation_date`. You cannot skip `country` and partition by `expected_graduation_date`.

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

#### Define partitions on interleaved tables

[인터리브된 테이블](interleave-in-parent.html)의 경우, 파티션은 인터리브 계층 구조의 루트 테이블에서만 정의될 수 있으며 자식은 부모와 같은 인터리브로 나타납니다.

### 복제 영역

On their own, partitions are inert and simply apply a label to the rows of the table that satisfy the criteria of the defined partitions.자체적으로, 파티션은 비활성이며 정의 된 파티션의 기준을 충족시키는 테이블의 행에 레이블을 적용하기 만하면됩니다. Applying functionality to a partition requires creating and applying [replication zone](configure-replication-zones.html) to the corresponding partitions.

CockroachDB uses the most granular zone config available. Zone configs that target a partition are considered more granular than those that target a table or index, which in turn are considered more granular than those that target a database.

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

Subpartition names must be unique within a table. In our example, even though `graduated` and `current` are sub-partitions of distinct partitions, they still need to be uniquely named. Hence the names `graduated_au`, `graduated_us`, and `current_au` and `current_us`.

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

Consider the partitioned table of students of RoachLearn. Suppose the table has been partitioned on range to store the current students on fast and expensive storage devices (example: SSD) and store the data of the graduated students on slower, cheaper storage devices(example: HDD). Now suppose we want to change the date after which the students will be considered current to `2018-08-15`. We can achieve this by using the [`PARTITION BY`](partition-by.html) subcommand of the [`ALTER TABLE`](alter-table.html) command.

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

읽기/쓰기를 빠르게 하는 것과 생존 실패 사이에는 상충 관계가 있습니다. Consider a partition with three replicas of `roachlearn.students` for Australian students.

- If only one replica is pinned to an Australian datacenter, then reads may be fast (via leases follow the workload) but writes will be slow.
- If two replicas are pinned to an Australian datacenter, then reads and writes will be fast (as long as the cross-ocean link has enough bandwidth that the third replica doesn’t fall behind). If those two replicas are in the same datacenter, then the loss of one datacenter can lead to data unavailability, so some deployments may want two separate Australian datacenters.
- If all three replicas are in Australian datacenters, then three Australian datacenters are needed to be resilient to a datacenter loss.

## CockroachDB의 파티셔닝이 다른 데이터베이스와 어떻게 다른가

다른 데이터베이스는 세 가지 추가 사용 경우에 대해 파티셔닝을 사용합니다: 보조 인덱스, 샤딩 및 대량 로드/삭제. CockroachDB는 파티셔닝을 사용하지 않고 다음과 같은 방법으로 이러한 사용-경우를 해결합니다.

- **보조 인덱스로 변경:** CockroachDB는 온라인 계획 변경을 통해 이러한 변경 사항을 해결합니다. Online schema changes are a superior feature to partitioning because they require zero-downtime and eliminate the potential for consistency problems. 온라인 계획 변경은 중단 시간을 필요로하고 일관성 문제의 가능성을 없애기 때문에 파티셔닝보다 우수한 기능입니다.
- **샤딩:** CockroachDB는 분산 데이터베이스 아키텍처의 일부로 데이터를 자동으로 파쇄합니다.
- **대량 로드 및 삭제:** CockroachDB에는 현재 이 사용 경우를 지원하는 기능이 없습니다.

## 알려진 제한 사항

{% include {{ page.version.version }}/known-limitations/partitioning-with-placeholders.md %}

## 더 보기

- [`CREATE TABLE`](create-table.html)
- [`ALTER TABLE`](alter-table.html)
- [계산된 열](computed-columns.html)
