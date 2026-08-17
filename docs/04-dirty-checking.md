---
layout: default
title: "Dirty Checking Internals"
description: "Hibernate + JPA Internals - Dirty Checking Internals"
permalink: /docs/04-dirty-checking/
---

<nav class="page-nav">
[← Persistence Context and First-Level Cache](/docs/03-persistence-context/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Flushing: Synchronize, Do Not Finalize →](/docs/05-flushing/)
</nav>

## 4. Dirty checking internals

## Snapshot-based algorithm

1.  Hydration: Hibernate reads a row and creates/populates the entity.

2.  Registration: the entity is placed in the persistence context under its EntityKey.

3.  Baseline: Hibernate retains loaded state (typically an Object\[\] of property values) and version/status metadata.

4.  Mutation: application changes fields directly; no repository call is required while the instance is managed.

5.  Flush traversal: Hibernate checks managed entities. For each dirty candidate, it compares current state with the baseline using Hibernate types/mutability rules.

6.  Scheduling: dirty properties produce an EntityUpdateAction; collection changes produce collection actions. Lifecycle callbacks/interceptors may participate.

7.  Execution: actions are ordered and translated to JDBC statements; the snapshot/version state is updated after successful execution.

## Bytecode enhancement and mutable values

With enhancement, field interception can mark which attributes changed (“self-dirty tracking”), reduce full comparisons, support lazy basic attributes, and improve association management. It does not remove flush semantics. Hibernate may still need snapshots for optimistic locking, mutable types, or correctness.

Mutable values are subtle. Replacing an immutable BigDecimal is easy to detect. Mutating a Date, byte array, embeddable, or custom mutable type in place requires Hibernate’s type system to deep-copy and compare correctly. A bad custom UserType/MutabilityPlan can hide changes or cause constant updates.

## SQL generation and @DynamicUpdate

Default UPDATE SQL is commonly pre-generated and sets all mapped updatable columns, even if only one changed. @DynamicUpdate generates SQL containing dirty columns at runtime. It can reduce writes and trigger/index churn for wide rows, but increases SQL-shape variety, statement-cache pressure, and per-flush work. @Version remains part of the predicate and is incremented.

```sql
-- typical optimistic update
UPDATE payment
SET amount = ?, status = ?, version = ?
WHERE id = ? AND version = ?;

-- affected rows == 0 => optimistic lock failure
```

## Read-only is not one universal switch

Spring @Transactional(readOnly=true) is primarily a hint/optimization. With Hibernate integration, Spring commonly sets a read-only-oriented flush mode and may mark the Session/default entities read-only, reducing snapshot/flush work. It is not a security boundary: the database may still allow writes, native SQL can still write, and behavior depends on transaction manager/provider/version. Enforce true read-only at database/connection/permissions where required.

| **COST MODEL** Dirty checking cost grows with the number of managed entities and properties considered, not merely the entities you intended to update. Keep persistence contexts intentionally sized. |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

<nav class="page-nav page-nav-bottom">
[← Persistence Context and First-Level Cache](/docs/03-persistence-context/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Flushing: Synchronize, Do Not Finalize →](/docs/05-flushing/)
</nav>
