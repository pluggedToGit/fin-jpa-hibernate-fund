---
layout: default
title: "Common Production Pitfalls"
description: "Hibernate + JPA Internals - Common Production Pitfalls"
permalink: /docs/15-production-pitfalls/
---

<nav class="page-nav">
[← Ten "Predict the SQL" Exercises](/docs/14-exercises/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Concise Revision Sheet →](/docs/16-revision-sheet/)
</nav>

## 15. Common production pitfalls

- Long @Transactional methods perform HTTP/Kafka calls while holding connections and row locks; local DB rollback cannot undo remote side effects. Use outbox/inbox, idempotency, sagas, and narrow transaction boundaries.

- Self-invocation bypasses the Spring transaction proxy, so the expected propagation/readOnly/isolation does not apply.

- Catching an inner REQUIRED exception and continuing even though the shared transaction is rollback-only, leading to UnexpectedRollbackException later.

- Using REQUIRES_NEW per item while outer transactions hold connections, causing pool starvation.

- Returning entities directly from APIs, triggering lazy loads during serialization and leaking persistence concerns.

- Open Session in View conceals N+1 and performs database work outside the intended service transaction.

- Default EAGER to-one mappings generate unexpected secondary selects.

- Calling save repeatedly on already managed entities and assuming save controls SQL timing.

- Ignoring the object returned from merge and mutating the detached input.

- Calling clear without flush and silently discarding pending changes.

- Running bulk DML and continuing to use stale L1 entities.

- Using L2/query cache for volatile payment status or authorization state and underestimating invalidation.

- Caching sensitive FundingSource token material without encryption, tenancy, expiry, and revocation design.

- Using equals/hashCode with generated mutable ids or lazy collections, breaking Sets and triggering SQL.

- CascadeType.ALL/REMOVE on shared relationships, deleting more data than the aggregate owns.

- orphanRemoval on immutable audit/history records, accidentally deleting evidence.

- Collection fetch join plus pagination, producing in-memory pagination or incomplete/huge results.

- Multiple large bag fetch joins, causing exceptions or Cartesian explosion.

- IDENTITY generation quietly disabling insert batching.

- Batch jobs that flush but never clear, so L1 memory still grows.

- Batch jobs that flush+clear but use one enormous database transaction, retaining locks/WAL/undo and making restart costly.

- Logging SQL without bind awareness—or logging sensitive binds in production.

- Treating readOnly=true as database-enforced read-only or as authorization.

- Retrying an optimistic/deadlock failure inside the same failed persistence context instead of starting a fresh transaction.

- Missing indexes on foreign keys/query predicates; ORM configuration is tuned while the database plan remains the bottleneck.

<nav class="page-nav page-nav-bottom">
[← Ten "Predict the SQL" Exercises](/docs/14-exercises/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Concise Revision Sheet →](/docs/16-revision-sheet/)
</nav>
