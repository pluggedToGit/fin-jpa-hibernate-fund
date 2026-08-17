---
layout: default
title: "Flushing: Synchronize, Do Not Finalize"
description: "Hibernate + JPA Internals - Flushing: Synchronize, Do Not Finalize"
permalink: /docs/05-flushing/
---

<nav class="page-nav">
[← Dirty Checking Internals](/docs/04-dirty-checking/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Second-Level Cache (L2) and Query Cache →](/docs/06-caching/)
</nav>

## 5. Flushing: synchronize, do not finalize

## What flush does

Flush converts the in-memory unit of work into SQL required to make database state consistent with the persistence context: detect dirtiness, cascade relevant operations, validate transient references, order entity/collection actions, execute JDBC statements/batches, check row counts and optimistic versions, and surface many constraint errors. The transaction is still open.

| **Flush mode**   | **Meaning**                                                                                                    | **Important nuance**                                                                                      |
|------------------|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| JPA AUTO         | Provider must ensure queries that could be affected observe pending changes; provider may flush before commit. | Hibernate uses query-space overlap to decide whether an HQL/JPQL query needs an auto-flush.               |
| JPA COMMIT       | Flush is required at commit; query visibility of unflushed changes is unspecified.                             | Hibernate may delay query flush, but identity/L1 effects can still influence returned objects.            |
| Hibernate ALWAYS | Flush before every query.                                                                                      | Provider-specific and often more work than AUTO.                                                          |
| Hibernate MANUAL | Only explicit flush (or provider-specific explicit handling).                                                  | Dangerous in write flows; commit behavior can differ from ordinary JPA expectations when forced natively. |
| Hibernate AUTO   | Native counterpart of typical JPA AUTO behavior.                                                               | Native-query synchronization behavior depends on API and declared synchronized query spaces.              |

## Flush versus commit timeline

```mermaid
sequenceDiagram
    participant App
    participant PC as Persistence Context
    participant DB
    App->>PC: payment.setStatus(APPROVED)
    App->>PC: flush()
    PC->>DB: UPDATE ... WHERE id=? AND version=?
    Note over DB: SQL visible per isolation; locks may be held
    alt success path
        App->>DB: COMMIT
    else later exception
        App->>DB: ROLLBACK
    end
```

## Why SQL runs early and why flush fails

- A JPQL/HQL query whose query spaces overlap pending Payment changes may trigger AUTO flush so its database result is not stale relative to pending writes.

- Explicit flush is used to detect errors early, obtain database-generated effects, or bound a batch.

- IDENTITY key generation often requires an immediate INSERT to obtain the id; this is SQL execution, though the row is still uncommitted.

- Flush can fail on NOT NULL, UNIQUE, FK, CHECK, length/type errors, deadlocks/timeouts, optimistic row-count mismatch, trigger failures, or connection errors.

A caught flush exception generally leaves the Hibernate Session and transaction unusable. Mark/allow rollback; do not continue as if only one statement failed. At commit, Spring’s transaction manager invokes provider synchronization/flush before the actual JDBC commit. A flush failure causes rollback. SQL already executed during flush is rolled back if the transaction rolls back.

| **Dimension**               | **flush()**                                                     | **commit()**                                                                        |
|-----------------------------|-----------------------------------------------------------------|-------------------------------------------------------------------------------------|
| Purpose                     | Synchronize persistence context to database transaction.        | Finalize the transaction atomically.                                                |
| SQL                         | May issue INSERT/UPDATE/DELETE and acquire locks.               | May first trigger flush, then database COMMIT.                                      |
| Durability                  | No. Changes remain uncommitted.                                 | Yes after successful commit, subject to DB guarantees.                              |
| Rollback possible afterward | Yes.                                                            | No ordinary rollback after successful commit.                                       |
| Entity state                | Entities generally remain managed.                              | Transaction-scoped context ends; objects usually become detached after cleanup.     |
| Errors                      | Constraints, optimistic locks, batching, DB errors can surface. | Flush errors plus commit-time/deferred constraints/connection failures can surface. |

<nav class="page-nav page-nav-bottom">
[← Dirty Checking Internals](/docs/04-dirty-checking/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Second-Level Cache (L2) and Query Cache →](/docs/06-caching/)
</nav>
