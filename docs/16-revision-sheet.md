---
layout: default
title: "Concise Revision Sheet"
description: "Hibernate + JPA Internals - Concise Revision Sheet"
permalink: /docs/16-revision-sheet/
---

<nav class="page-nav">
[← Common Production Pitfalls](/docs/15-production-pitfalls/) &nbsp;|&nbsp; [🏠 Home](/)
</nav>

## 16. Concise revision sheet

| **If asked…**      | **Answer in one precise sentence**                                                                                                  |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| JPA vs Hibernate   | JPA is the persistence contract; Hibernate is a provider implementing it plus native features.                                      |
| EntityManager      | A non-thread-safe façade over one persistence context/unit of work.                                                                 |
| L1                 | Mandatory per-context identity map plus snapshots/actions; same identity returns same managed object.                               |
| L2                 | Optional factory/cluster-scoped cache of disassembled state, never shared managed objects.                                          |
| Query cache        | Caches result ids/scalars for exact query inputs; entity state still resolves through L1/L2/DB.                                     |
| Dirty checking     | At flush, Hibernate detects managed-state changes via snapshots and/or enhanced dirty flags and schedules DML.                      |
| flush              | Synchronizes ORM state to SQL inside the current transaction; it is rollbackable and not durable.                                   |
| commit             | Finalizes the database transaction; usually flushes first.                                                                          |
| persist            | Makes the same new instance managed and schedules INSERT.                                                                           |
| merge              | Copies state into and returns a managed instance; the argument stays detached/transient.                                            |
| detach/clear       | Stop tracking one/all entities; unflushed changes can be lost.                                                                      |
| AUTO query flush   | Hibernate flushes when pending changes overlap the query’s affected tables/query spaces.                                            |
| @Version           | Adds a version predicate so concurrent same-row writes fail instead of silently overwriting.                                        |
| Pessimistic lock   | Database lock held to transaction end; can block/deadlock and depends on SQL/index plan.                                            |
| Bulk JPQL          | Direct DML bypasses managed state/callbacks/cascades and leaves L1 stale; flush then clear/refresh.                                 |
| N+1                | One root query followed by per-row association queries; solve with planned fetch graph, batching, or DTOs.                          |
| Batch inserts      | Prefer SEQUENCE allocation, JDBC batch size, ordered inserts, and periodic flush+clear.                                             |
| Payment L2         | Usually avoid for volatile authoritative payment state; freshness and invalidation cost outweigh hit benefit.                       |
| Spring transaction | Proxy opens/binds EntityManager and DB transaction, method mutates entities, completion flushes then commits/rolls back.            |
| Debugging          | Observe transaction boundary, L1 state, flush trigger, generated SQL/binds, row counts, locks, and final commit outcome separately. |

## The 30-second mental execution model

1\. Spring proxy starts transaction and binds EntityManager.  
2. find/query resolves rows into the L1 identity map.  
3. Managed objects are mutated; snapshots/dirty flags remember divergence.  
4. AUTO query, explicit flush, or transaction completion triggers flush.  
5. Hibernate cascades, dirty-checks, orders actions, batches SQL, and verifies versions.  
6. Database constraints/locks apply when SQL executes.  
7. Commit makes it durable; rollback reverses flushed SQL.  
8. Context closes; objects detach. L2 may retain only cached state, never live objects.

| **STAFF-LEVEL CLOSE** Do not optimize Hibernate in isolation. Choose aggregate boundaries, transaction length, consistency guarantees, fetch plans, database constraints/indexes, cache policy, observability, and failure/retry semantics as one system design. |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

<script type="module">
  import mermaid from "https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs";
  document.querySelectorAll("pre > code.language-mermaid").forEach((code) => {
    const diagram = document.createElement("div");
    diagram.className = "mermaid";
    diagram.textContent = code.textContent;
    code.parentElement.replaceWith(diagram);
  });
  mermaid.initialize({ startOnLoad: false, theme: "neutral" });
  await mermaid.run({ querySelector: ".mermaid" });
</script>

<nav class="page-nav page-nav-bottom">
[← Common Production Pitfalls](/docs/15-production-pitfalls/) &nbsp;|&nbsp; [🏠 Home](/)
</nav>
