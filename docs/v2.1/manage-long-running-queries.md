---
title: Manage Long-Running Queries
summary: Learn how to identify and cancel long-running queries.
toc: true
---

이 페이지는 처리하는 데 예상보다 오래 걸리는 SQL 쿼리를 식별하고, 필요한 경우 취소하는 방법을 보여줍니다.

{{site.data.alerts.callout_success}} 스키마 변경은 다른 SQL 쿼리와 다르게 처리됩니다. <a href="show-jobs.html"><code>SHOW JOBS</code></a>을 사용하여 스키마 변경 진행 상황을 모니터링 할 수 있으며, 버전 2.1에서는 <a href="cancel-job.html"><code>CANCEL JOB</code></a>를 클릭하면 예상보다 오래 걸리는 스키마 변경 사항을 취소할 수 있습니다. {{site.data.alerts.end}}



## 장기-실행 쿼리 식별

[`SHOW QUERIES`](show-queries.html) 명령문을 사용하여 각 쿼리의 `start` 타임스탬프를 포함하여 현재 활성화된 SQL 쿼리에 대한 세부 정보를 나열하십시오:

{% include copy-clipboard.html %}
~~~ sql
> SHOW QUERIES;
~~~

~~~
+----------------------------------+---------+----------+----------------------------------+-------------------------------------------+---------------------+------------------+-------------+-----------+
|             query_id             | node_id | username |              start               |                   query                   |   client_address    | application_name | distributed |   phase   |
+----------------------------------+---------+----------+----------------------------------+-------------------------------------------+---------------------+------------------+-------------+-----------+
| 14db657443230c3e0000000000000001 |       1 | root     | 2017-08-16 18:00:50.675151+00:00 | UPSERT INTO test.kv(k, v) VALUES ($1, $2) | 192.168.12.56:54119 | test_app         | false       | executing |
| 14db657443b68c7d0000000000000001 |       1 | root     | 2017-08-16 18:00:50.684818+00:00 | UPSERT INTO test.kv(k, v) VALUES ($1, $2) | 192.168.12.56:54123 | test_app         | false       | executing |
| 14db65744382c2340000000000000001 |       1 | root     | 2017-08-16 18:00:50.681431+00:00 | UPSERT INTO test.kv(k, v) VALUES ($1, $2) | 192.168.12.56:54103 | test_app         | false       | executing |
| 14db657443c9dc660000000000000001 |       1 | root     | 2017-08-16 18:00:50.686083+00:00 | SHOW CLUSTER QUERIES                      | 192.168.12.56:54108 | cockroach        | NULL        | preparing |
| 14db657443e30a850000000000000003 |       3 | root     | 2017-08-16 18:00:50.68774+00:00  | UPSERT INTO test.kv(k, v) VALUES ($1, $2) | 192.168.12.58:54118 | test_app         | false       | executing |
| 14db6574439f477d0000000000000003 |       3 | root     | 2017-08-16 18:00:50.6833+00:00   | UPSERT INTO test.kv(k, v) VALUES ($1, $2) | 192.168.12.58:54122 | test_app         | false       | executing |
| 14db6574435817d20000000000000002 |       2 | root     | 2017-08-16 18:00:50.678629+00:00 | UPSERT INTO test.kv(k, v) VALUES ($1, $2) | 192.168.12.57:54121 | test_app         | false       | executing |
| 14db6574433c621f0000000000000002 |       2 | root     | 2017-08-16 18:00:50.676813+00:00 | UPSERT INTO test.kv(k, v) VALUES ($1, $2) | 192.168.12.57:54124 | test_app         | false       | executing |
| 14db6574436f71d50000000000000002 |       2 | root     | 2017-08-16 18:00:50.680165+00:00 | UPSERT INTO test.kv(k, v) VALUES ($1, $2) | 192.168.12.57:54117 | test_app         | false       | executing |
+----------------------------------+---------+----------+----------------------------------+-------------------------------------------+---------------------+------------------+-------------+-----------+
(9 rows)
~~~

특정 시간 동안 실행된 쿼리를 필터링할 수도 있습니다. 예를 들어, 3시간 이상 실행 된 쿼리를 찾으려면, 다음을 실행합니다:

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM [SHOW CLUSTER QUERIES]
      WHERE start < (now() - INTERVAL '3 hours');
~~~

## 장기-실행 쿼리 취소

[`SHOW QUERIES`](show-queries.html)를 통해 장시간-실행되는 쿼리를 식별했으면, `query_id`를 기록하고 [`CANCEL QUERY`](cancel-query.html) 명령문과 함께 사용하십시오 :

{% include copy-clipboard.html %}
~~~ sql
> CANCEL QUERY '14dacc1f9a781e3d0000000000000001';
~~~

쿼리가 성공적으로 취소되면, CockroachDB는 쿼리를 실행한 클라이언트에게 `쿼리 실행 취소됨` 오류를 보냅니다.

- 취소된 쿼리가 단일 독립 실행형 명령문인 경우, 클라이언트에서 추가 작업이 필요하지 않습니다.
- 취소된 쿼리가 더 큰 다중-명령문 [트랜잭션](transactions.html)의 일부인 경우, 클라이언트는 [`ROLLBACK`] 명령문을 실행해야 합니다.

## 쿼리 성능 향상

장기-실행 쿼리를 취소한 후, [`EXPLAIN`](explain.html) 명령문을 사용하여 검사합니다. 전체-테이블 스캔을 수행하므로 쿼리가 느려질 수 있습니다. 이러한 경우, [인덱스 추가](create-index.html)를 통해 쿼리 성능을 향상시킬 수 있습니다.

*(쿼리 성능 최적화에 대한 자세한 지침이 곧 제공됩니다.)*

## 더 보기

- [`SHOW QUERIES`](show-queries.html)
- [`CANCEL QUERY`](cancel-query.html)
- [`EXPLAIN`](explain.html)
- [쿼리 동작 문제 해결](query-behavior-troubleshooting.html) 
