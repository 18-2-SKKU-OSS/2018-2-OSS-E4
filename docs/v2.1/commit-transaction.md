---
title : COMMIT
summary: Commit a transaction with the COMMIT statement in CockroachDB.
toc: true
---

`COMMIT` [statement](sql-statements.html)은 현재 [transaction](transactions.html)을 커밋하거나, [client-side transaction retries](transactions.html#client-side-transaction-retries)을 사용할 때, 새 처리를 시작할 수 있도록 연결을 지웁니다.

[client-side transaction retries](transactions.html#client-side-transaction-retries)을 사용할 때, [`SAVEPOINT cockroach_restart`](savepoint.html) 이후에 이슈된 statement들은 `COMMIT`대신에 [`RELEASE SAVEPOINT cockroach_restart`](release-savepoint.html)이 이슈되었을 때 커밋됩니다. 그러나, 다음 처리를 위해 연결을 취소하려면 `COMMIT`을 이슈해야 합니다. 

반환할 수 없는 처리의 경우, [generated any errors](transactions.html#error-handling) 처리에 있는 statement의 경우, `COMMIT`은 `ROLLBACK`과 같고, 'ROLLBACK'은 트랜젝션을 중단하고 해당 보고서에 의해 수행된 모든 업데이트를 삭제합니다.

## 시놉시스

<section> {% include {{ page.version.version }}/sql/diagrams/commit_transaction.html %} </section>

## 필수권한

거래를 수행하는데 [privileges](privileges.html)는 필요하지 않습니다. 그러나, 트랜젝션 내의 각 statement에는 권한이 필요합니다.

## Aliases

CockroachDB에서, `END`는 `COMMIT`의 별칭이다.

## Example

### Commit a transaction

트랜젝션을 커밋하는 방법은 응용프로그램에서 [transaction retries](transactions.html#transaction-retries)을 처리하는 방식에 따라 달라집니다.

#### Client-side retryable transactions

[client-side transaction retries](transactions.html#client-side-transaction-retries)을 사용 할 때, statements들은 [`RELEASE SAVEPOINT cockroach_restart`](release-savepoint.html)에 의해 커밋됩니다. `COMMIT` 자체는 다음 트렌젝션을 위한 연결을 처리하는데에만 사용됩니다.

{% include copy-clipboard.html %}
~~~ sql
> BEGIN;
~~~

{% include copy-clipboard.html %}
~~~ sql
> SAVEPOINT cockroach_restart;
~~~

{% include copy-clipboard.html %}
~~~ sql
> UPDATE products SET inventory = 0 WHERE sku = '8675309';
~~~

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO orders (customer, sku, status) VALUES (1001, '8675309', 'new');
~~~

{% include copy-clipboard.html %}
~~~ sql
> RELEASE SAVEPOINT cockroach_restart;
~~~

{% include copy-clipboard.html %}
~~~ sql
> COMMIT;
~~~

{{site.data.alerts.callout_danger}}이 예에서는 트랜젝션 재시도를 처리하기 위해 <a href="transactions.html#client-side-intervention">클라이언트 측 개입을 사용하고 있다고 가정합니다</a>.{{site.data.alerts.end}}

#### Automatically retried transactions

CockroachDB가 [automatically retry](transactions.html#automatic-retries) (i.e., all statements sent in a single batch)할 트랜젝션을 사용하는 경우, `COMMIT`으로 트랜젝션을 커밋합니다.

{% include copy-clipboard.html %}
~~~ sql
> BEGIN; UPDATE products SET inventory = 100 WHERE = '8675309'; UPDATE products SET inventory = 100 WHERE = '8675310'; COMMIT;
~~~

## See also

- [Transactions](transactions.html)
- [`BEGIN`](begin-transaction.html)
- [`RELEASE SAVEPOINT`](release-savepoint.html)
- [`ROLLBACK`](rollback-transaction.html)
- [`SAVEPOINT`](savepoint.html)
