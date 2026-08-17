---
layout: default
title: "Performance Engineering"
description: "Hibernate + JPA Internals - Performance Engineering"
permalink: /docs/10-performance/
---

<nav class="page-nav">
[← Transactions and Concurrency](/docs/09-transactions/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Payment-Service Execution Traces →](/docs/11-traces/)
</nav>

## 10. Performance engineering

| **Lever**                  | **Mechanism**                                                                        | **Failure mode / measurement**                                                                                 |
|----------------------------|--------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| JDBC batching              | hibernate.jdbc.batch_size groups same-shape DML; driver sends batches.               | IDENTITY inserts and mixed SQL shapes reduce batching; verify driver metrics/logs, not just config.            |
| Insert/update ordering     | order_inserts/order_updates groups statements by entity/key.                         | Sorting overhead; can change lock order beneficially but never rely on incidental ordering for business logic. |
| flush + clear batching     | Every N records flush SQL, then detach all to bound memory.                          | clear detaches references; exceptions invalidate transaction; one giant transaction still holds locks/log.     |
| SEQUENCE allocation        | Preallocates id blocks; supports batched inserts.                                    | allocationSize must align with sequence strategy; gaps are normal.                                             |
| IDENTITY                   | DB generates id on INSERT.                                                           | Often forces immediate per-row INSERT and disables effective insert batching.                                  |
| Connection pool            | Persistence context usually acquires/holds connection around transactional DB work.  | Long transactions, remote calls, nested REQUIRES_NEW, or streaming exhaust pool.                               |
| JDBC fetch size            | Hints rows fetched per network round trip for large result sets.                     | Not association batch size; driver/database semantics vary.                                                    |
| Hibernate batch fetch size | Batches lazy proxy/collection loads by identifiers.                                  | Does not control JDBC network fetch buffering.                                                                 |
| Plan/index                 | SQL shape, predicates, join order, cardinality, indexes dominate DB time.            | ORM tuning cannot rescue scans, bad indexes, parameter skew, or lock contention.                               |
| SQL observability          | Statement count, bind values safely, timings, rows, plans, Hibernate statistics/APM. | Raw show_sql in production is noisy and may leak sensitive data.                                               |

## Detecting unnecessary SQL

- Assert query counts in integration tests for critical endpoints; inspect N+1 under realistic graph access.

- Enable Hibernate statistics selectively; correlate slow traces with SQL, connection wait, row count, and flush events.

- Look for repeated selects, full-column updates, unexpected eager loads, selects caused by merge, collection delete/reinsert, and premature flushes.

- Use EXPLAIN/EXPLAIN ANALYZE safely on representative parameter values and data volumes. Index foreign keys and high-selectivity predicates based on workload, not annotations alone.

<nav class="page-nav page-nav-bottom">
[← Transactions and Concurrency](/docs/09-transactions/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Payment-Service Execution Traces →](/docs/11-traces/)
</nav>
