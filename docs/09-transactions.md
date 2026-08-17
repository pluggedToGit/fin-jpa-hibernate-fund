---
layout: default
title: "Transactions and Concurrency"
description: "Hibernate + JPA Internals - Transactions and Concurrency"
permalink: /docs/09-transactions/
---

<nav class="page-nav">
[← Relationships and Aggregate Correctness](/docs/08-relationships/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Performance Engineering →](/docs/10-performance/)
</nav>

## 9. Transactions and concurrency

## Spring @Transactional and Hibernate

8.  Caller crosses the Spring proxy. The interceptor resolves a PlatformTransactionManager.

9.  JpaTransactionManager obtains/creates an EntityManager, binds it to the thread, obtains a JDBC connection through the provider, and begins the transaction.

10. Repositories and @PersistenceContext proxies resolve the bound EntityManager, so they share one persistence context.

11. Business code loads and mutates managed entities. SQL may execute immediately for reads/identity keys, at query auto-flush, explicit flush, or completion.

12. On normal return, synchronization flushes; the manager commits; resources are unbound/closed. On configured rollback exceptions, it rolls back.

13. Default Spring rollback rules: unchecked RuntimeException/Error roll back; checked exceptions do not unless configured. An inner REQUIRED participant marking rollback-only can cause UnexpectedRollbackException at the outer commit.

## Optimistic locking

@Version adds the loaded version to UPDATE/DELETE predicates. Two transactions may both read version 7. First commits UPDATE ... version=8 WHERE id=? AND version=7. Second affects zero rows at flush and receives OptimisticLockException / StaleObjectStateException, commonly translated by Spring. Retry the entire business transaction only if the operation is safe and bounded; reload and revalidate external assumptions.

## Pessimistic locking

PESSIMISTIC_WRITE generally issues SELECT ... FOR UPDATE (dialect-specific), holding a database lock until transaction completion. It serializes contenders but can deadlock, time out, reduce throughput, and lock more rows than expected depending on query/index plan. Hibernate’s L1 state does not itself lock the database. A lock request on an already managed entity may still require SQL to acquire/upgrade the database lock.

## Lost update, write skew, and isolation

| **Anomaly**         | **Example**                                                                                                                           | **Defense**                                                                                                 |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Lost update         | T1 and T2 read Payment v7; both write status; last writer wins without version predicate.                                             | @Version, atomic conditional UPDATE, or pessimistic lock.                                                   |
| Write skew          | Two transactions update different rows after checking a cross-row invariant, e.g., total daily funding limit remains below threshold. | SERIALIZABLE, explicit predicate/advisory locking, materialized invariant row, or atomic constraint/design. |
| Non-repeatable read | Same row read twice yields different committed values.                                                                                | REPEATABLE READ or explicit lock; L1 may mask repeated find but not arbitrary query semantics.              |
| Phantom             | Re-running predicate query returns new rows.                                                                                          | Database-specific REPEATABLE READ/SERIALIZABLE behavior; predicate/range locks or redesign.                 |

READ COMMITTED prevents dirty reads but does not inherently prevent lost updates or write skew. REPEATABLE READ names differ by database implementation. SERIALIZABLE is strongest but can abort transactions and requires retry. Hibernate cannot upgrade weak database isolation merely by tracking objects.

## Bulk DML bypasses managed state

JPQL UPDATE/DELETE and native DML operate directly in the database. They do not run per-entity dirty checking, cascades, lifecycle callbacks, or automatically update already managed objects. Hibernate normally auto-flushes relevant pending entity work before bulk DML, executes it, and may invalidate affected L2/query regions, but L1 objects remain stale. Use @Modifying(clearAutomatically=true), explicit clear(), or run bulk work in a deliberately isolated context. Beware: clearing discards other unflushed changes unless flushed first.

<nav class="page-nav page-nav-bottom">
[← Relationships and Aggregate Correctness](/docs/08-relationships/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Performance Engineering →](/docs/10-performance/)
</nav>
