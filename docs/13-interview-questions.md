---
layout: default
title: "Twenty Difficult Interview Questions"
description: "Hibernate + JPA Internals - Twenty Difficult Interview Questions"
permalink: /docs/13-interview-questions/
---

<nav class="page-nav">
[← Comparison Tables](/docs/12-comparison-tables/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Ten "Predict the SQL" Exercises →](/docs/14-exercises/)
</nav>

## 13. Twenty difficult interview questions — with answers

### 1. Why does a JPQL query still hit the database when the entity is already in L1?

L1 indexes entities by identity; it is not a general predicate index. The database must determine which ids match. Hibernate then resolves matching ids through L1 so an existing managed instance is reused.

### 2. Can find() return stale data under READ COMMITTED?

Yes. If the entity is already managed, find returns it without re-reading. READ COMMITTED governs database reads; L1 identity semantics can mask a newer committed row until refresh/clear/new context.

### 3. Is flush equivalent to commit?

No. Flush executes synchronization SQL in the open transaction. Commit makes it durable. A later rollback reverses flushed SQL.

### 4. Why might persist execute INSERT immediately?

IDENTITY ids require the database INSERT to produce the key; some transaction/flush circumstances also require early SQL. With pooled SEQUENCE, Hibernate can assign ids without inserting and batch later.

### 5. Why is merge commonly misunderstood?

It copies state into a managed instance and returns that instance. The argument remains detached/transient. Continuing to mutate the argument is not tracked.

### 6. Why can save() be redundant?

A managed entity is automatically dirty-checked. JpaRepository.save is useful for new/detached instances but is not the trigger for updating an already managed entity.

### 7. What exactly does @Version prevent?

It detects conflicting writes to the same versioned row by adding version to DML predicates. It does not automatically protect cross-row invariants or prevent write skew.

### 8. Does L2 cache violate transaction isolation?

It can introduce stale reads if misconfigured, but it does not share live objects. Concurrency strategies and invalidation coordinate entries; the database remains authoritative. Strict requirements may justify disabling L2.

### 9. Why can @Transactional(readOnly=true) improve performance?

Hibernate integration can reduce flush/dirty-check snapshot work and choose a read-oriented flush mode. It is a hint, not guaranteed write prevention.

### 10. What happens if you catch a constraint exception from flush?

The transaction is normally marked rollback-only or the Session is inconsistent. Do not continue using it; roll back. The database may have rejected a batch after partial statement execution within the transaction.

### 11. Why can an eager mapping still produce N+1?

EAGER promises availability, not join syntax. Hibernate may load roots first and issue secondary selects per association, especially with JPQL that does not fetch join.

### 12. Why is collection fetch-join pagination dangerous?

The SQL result has one row per root-child pair, while pagination counts SQL rows. Results may be incomplete or paginated in memory. Page ids first, then fetch the graph.

### 13. What does bulk JPQL bypass?

Per-entity state, dirty checking, callbacks, cascades, and L1 synchronization. It directly updates rows and can leave managed entities stale.

### 14. L1 cache versus database repeatable read?

L1 guarantees one Java instance per identity and reuses its state. It does not make predicate queries repeatable or prevent phantoms; database isolation controls those.

### 15. Can clear() lose data?

Yes. It detaches all entities. Unflushed in-memory changes are no longer tracked. Flush before clear if those changes must reach the transaction.

### 16. What is cached for a collection region?

Usually collection membership—element ids/keys—keyed by owner and role. Element attributes come from their entity regions or database.

### 17. How can REQUIRES_NEW exhaust the connection pool?

The outer transaction may retain its connection while the inner independent transaction needs another. Concurrent threads can each hold one and wait for another, so the pool must account for nested demand or the design must change.

### 18. Why can @DynamicUpdate hurt?

It creates many SQL shapes based on dirty-column combinations, reducing statement/plan cache reuse and adding runtime generation work. Benefits are workload-dependent.

### 19. Why does flush order differ from code order?

Hibernate queues actions and orders them to satisfy FK constraints and improve batching. The action queue, not source-line order, determines much DML execution order.

### 20. How would you debug “Hibernate issued an update I did not expect”?

Capture transaction/flush boundaries and SQL with binds safely; inspect managed graph mutations, setters/listeners, mutable types, cascades, collection changes, merge copying, and dirty snapshots/statistics. Reproduce with a query-count integration test.

<nav class="page-nav page-nav-bottom">
[← Comparison Tables](/docs/12-comparison-tables/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Ten "Predict the SQL" Exercises →](/docs/14-exercises/)
</nav>
