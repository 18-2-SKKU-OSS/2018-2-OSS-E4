---
title: SQL Audit Logging
summary: Use the EXPERIMENTAL_AUDIT setting to turn SQL audit logging on or off for a table.
toc: true
---

SQL 감사 기록은 시스템에 대해 실행 중인 쿼리에 대한 자세한 정보를 제공합니다. 이 기능은 개인 식별 가능 정보(PII)를 포함하는 테이블에 대해 실행되는 모든 쿼리를 기록하려는 경우에 특히 유용하게 사용될 수 있습니다.

이 페이지에서는 다음과 같은 예제를 보여줍니다:

- 감사 로깅을 활성/비활성 하는 방법
- 감사 로그 파일이 있는 위치
- 감사 로그 파일 형식

감사 로그 파일 형식에 대한 자세한 설명은 [`ALTER TABLE ... EXPERIMENTAL_AUDIT`](experimental-audit.html)를 참조하십시오.

{% include {{ page.version.version }}/misc/experimental-warning.md %}


## 1단계. 샘플 테이블 생성

아래와 같이 테이블을 생성합니다:

- 이름, 주소 등의 PII가 포함된 `customers` 테이블
- `customers`에 대해 PII를 포함하지 않는 외부 키를 갖는 `orders` 테이블

나중에, `customers` 테이블에 대한 감사 로그를 어떻게 활성화하는지 볼 것 입니다.

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING NOT NULL,
    address STRING NOT NULL,
    national_id INT NOT NULL,
    telephone INT NOT NULL,
    email STRING UNIQUE NOT NULL
);
~~~

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id INT NOT NULL,
    delivery_status STRING check (delivery_status='processing' or delivery_status='in-transit' or delivery_status='delivered') NOT NULL,
    customer_id UUID NOT NULL REFERENCES customers (id)
);
~~~

## 2단계. `customers` 테이블에 대한 감사 활성화

[`ALTER TABLE`](alter-table.html)의 하위 명령인 [`EXPERIMENTAL_AUDIT`](experimental-audit.html)를 사용하여 감사를 활성화합니다.

{% include copy-clipboard.html %}
~~~ sql
> ALTER TABLE customers EXPERIMENTAL_AUDIT SET READ WRITE;
~~~

{{site.data.alerts.callout_info}}
두 개 이상의 테이블에 대한 감사를 설정하려면 각 테이블에 대해 별도의 `ALTER` 문을 실행하십시오.
{{site.data.alerts.end}}

## 3단계. `customers` 테이블에 데이터 덧붙이기

이제 감사 기능이 활성화되어 있으므로 고객 데이터를 몇 가지 추가해 봅시다.

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO customers (name, address, national_id, telephone, email) VALUES (
    'Pritchard M. Cleveland',
    '23 Crooked Lane, Garden City, NY USA 11536',
    778124477,
    12125552000,
    'pritchmeister@aol.com'
);
~~~

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO customers (name, address, national_id, telephone, email) VALUES (
    'Vainglorious K. Snerptwiddle III',
    '44 Straight Narrows, Garden City, NY USA 11536',
    899127890,
    16465552000,
    'snerp@snerpy.net'
);
~~~

고객이 성공적으로 추가되었는지 확인하십시오:

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM customers;
~~~

~~~
+--------------------------------------+----------------------------------+------------------------------------------------+-------------+-------------+-----------------------+
|                  id                  |               name               |                    address                     | national_id |  telephone  |         email         |
+--------------------------------------+----------------------------------+------------------------------------------------+-------------+-------------+-----------------------+
| 4bd266fc-0b62-4cc4-8c51-6997675884cd | Vainglorious K. Snerptwiddle III | 44 Straight Narrows, Garden City, NY USA 11536 |   899127890 | 16465552000 | snerp@snerpy.net      |
| 988f54f0-b4a5-439b-a1f7-284358633250 | Pritchard M. Cleveland           | 23 Crooked Lane, Garden City, NY USA 11536     |   778124477 | 12125552000 | pritchmeister@aol.com |
+--------------------------------------+----------------------------------+------------------------------------------------+-------------+-------------+-----------------------+
(2 rows)
~~~

## 4단계. 감사 로그 확인

기본적으로 활성 감사 로그 파일의 이름은 `cockroach-sql-audit.log`이며, CockroachDB의 표준 로그 디렉터리에 저장됩니다. 감사 로그 파일을 특정 디렉터리에 저장하려면  `--sql-audit-dir` 플래그를 [`cockroach start`](start-a-node.html)로 전달하십시오. 감사 로그 파일은 다른 로그 파일과 마찬가지로 `--log-file-max-size` 설정에 따라 교대됩니다.

이 예제의 감사 로그를 보면, 우리가 지금까지 실행한 모든 명령을 보여주는 다음과 같은 내용이 보입니다.

~~~
I180321 20:54:21.381565 351 sql/exec_log.go:163  [n1,client=127.0.0.1:60754,user=root] 2 exec "psql" {"customers"[76]:READWRITE} "ALTER TABLE customers EXPERIMENTAL_AUDIT SET READ WRITE" {} 4.811 0 OK
I180321 20:54:26.315985 351 sql/exec_log.go:163  [n1,client=127.0.0.1:60754,user=root] 3 exec "psql" {"customers"[76]:READWRITE} "INSERT INTO customers(\"name\", address, national_id, telephone, email) VALUES ('Pritchard M. Cleveland', '23 Crooked Lane, Garden City, NY USA 11536', 778124477, 12125552000, 'pritchmeister@aol.com')" {} 6.319 1 OK
I180321 20:54:30.080592 351 sql/exec_log.go:163  [n1,client=127.0.0.1:60754,user=root] 4 exec "psql" {"customers"[76]:READWRITE} "INSERT INTO customers(\"name\", address, national_id, telephone, email) VALUES ('Vainglorious K. Snerptwiddle III', '44 Straight Narrows, Garden City, NY USA 11536', 899127890, 16465552000, 'snerp@snerpy.net')" {} 2.809 1 OK
I180321 20:54:39.377395 351 sql/exec_log.go:163  [n1,client=127.0.0.1:60754,user=root] 5 exec "psql" {"customers"[76]:READ} "SELECT * FROM customers" {} 1.236 2 OK
~~~

{{site.data.alerts.callout_info}}
감사 로그 파일 형식에 대한 자세한 설명은 [`ALTER TABLE ... EXPERIMENTAL_AUDIT`](experimental-audit.html)를 참조하십시오.
{{site.data.alerts.end}}

## 5단계. `orders` 테이블에 데이터 덧붙이기

`customers` 테이블과 다르게 `orders` 테이블은 PII를 포함하지 않고 제품 ID와 배송 상태 등만 저장합니다. (사용하지 않는 `ENUM`에 대한 해결 방법으로 [`CHECK` 제약 조건](check.html)을 사용합니다. 자세한 내용은 [SQL 기능 지원](sql-feature-support.html)을 참조하십시오.)

[`CREATE SEQUENCE`](create-sequence.html)를 사용하여 `orders` 테이블에 몇 개의 더미 데이터를 덧붙입니다.

{% include copy-clipboard.html %}
~~~ sql
> CREATE SEQUENCE product_ids_asc START 1 INCREMENT 1;
~~~

데이터를 생성하기 위해 아래 사항을 몇 번 평가하십시오.
[`SELECT`](select-clause.html)가 여러 결과를 반환할 경우 오류가 발생하지만 이 경우에는 그렇지 않다는 점에 주목하십시오.

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO orders (product_id, delivery_status, customer_id) VALUES (
    nextval('product_ids_asc'),
    'processing',
    (SELECT id FROM customers WHERE name ~ 'Cleve')
);
~~~

Let's verify that our orders were added successfully:

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM orders ORDER BY product_id;
~~~

~~~
+--------------------------------------+------------+-----------------+--------------------------------------+
|                  id                  | product_id | delivery_status |             customer_id              |
+--------------------------------------+------------+-----------------+--------------------------------------+
| 6e85c390-3bbf-48da-9c2f-a73a0ab9c2ce |          1 | processing      | df053c68-fcb0-4a80-ad25-fef9d3b408ca |
| e93cdaee-d5eb-428c-bc1b-a7367f334f99 |          2 | processing      | df053c68-fcb0-4a80-ad25-fef9d3b408ca |
| f05a1b0f-5847-424d-b8c8-07faa6b6e46b |          3 | processing      | df053c68-fcb0-4a80-ad25-fef9d3b408ca |
| 86f619d6-9f18-4c84-8ead-68cd07a1ee37 |          4 | processing      | df053c68-fcb0-4a80-ad25-fef9d3b408ca |
| 882c0fc8-64e7-4fab-959d-a4ff74f170c0 |          5 | processing      | df053c68-fcb0-4a80-ad25-fef9d3b408ca |
+--------------------------------------+------------+-----------------+--------------------------------------+
(5 rows)
~~~

##  6단계. 감사 로그 다시 확인하기

`orders` 테이블을 위한 더미 데이터를 생성하기 위해 `customers` 테이블에 대해 `SELECT`를 사용했기 때문에 감사 로그에도 다음과 같이 쿼리가 나타납니다.

~~~
I180321 21:01:59.677273 351 sql/exec_log.go:163  [n1,client=127.0.0.1:60754,user=root] 7 exec "psql" {"customers"[76]:READ, "customers"[76]:READ} "INSERT INTO orders(product_id, delivery_status, customer_id) VALUES (nextval('product_ids_asc'), 'processing', (SELECT id FROM customers WHERE \"name\" ~ 'Cleve'))" {} 5.183 1 OK
I180321 21:04:07.497555 351 sql/exec_log.go:163  [n1,client=127.0.0.1:60754,user=root] 8 exec "psql" {"customers"[76]:READ, "customers"[76]:READ} "INSERT INTO orders(product_id, delivery_status, customer_id) VALUES (nextval('product_ids_asc'), 'processing', (SELECT id FROM customers WHERE \"name\" ~ 'Cleve'))" {} 5.219 1 OK
I180321 21:04:08.730379 351 sql/exec_log.go:163  [n1,client=127.0.0.1:60754,user=root] 9 exec "psql" {"customers"[76]:READ, "customers"[76]:READ} "INSERT INTO orders(product_id, delivery_status, customer_id) VALUES (nextval('product_ids_asc'), 'processing', (SELECT id FROM customers WHERE \"name\" ~ 'Cleve'))" {} 5.392 1 OK
~~~

{{site.data.alerts.callout_info}}
감사 로그 파일 형식에 대한 자세한 설명은 [`ALTER TABLE ... EXPERIMENTAL_AUDIT`](experimental-audit.html)를 참조하십시오.
{{site.data.alerts.end}}

## 더 알아보기

- [`ALTER TABLE ... EXPERIMENTAL_AUDIT`](experimental-audit.html)
- [`cockroach start` 로깅 플래그](start-a-node.html#logging)
- [SQL FAQ - 고유한 행 ID 생성](sql-faqs.html#how-do-i-auto-generate-unique-row-ids-in-cockroachdb)
- [`CREATE SEQUENCE`](create-sequence.html)
- [SQL 기능 지원](sql-feature-support.html)
