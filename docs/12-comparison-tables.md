---
layout: default
title: "Comparison Tables"
description: "Hibernate + JPA Internals - Comparison Tables"
permalink: /docs/12-comparison-tables/
---

<nav class="page-nav">
[← Payment-Service Execution Traces](/docs/11-traces/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Twenty Difficult Interview Questions →](/docs/13-interview-questions/)
</nav>

## 12. Comparison tables

## L1 versus L2 versus query cache

| **Property**       | **L1 / persistence context**                                | **L2 entity/collection cache**                     | **Query cache**                                       |
|--------------------|-------------------------------------------------------------|----------------------------------------------------|-------------------------------------------------------|
| Required           | Yes                                                         | No                                                 | No                                                    |
| Scope              | EntityManager / Session                                     | EntityManagerFactory / SessionFactory / cluster    | SessionFactory / cluster                              |
| Stores             | Live managed instances + snapshots/actions                  | Disassembled entity state or collection membership | Result ids/scalars keyed by query inputs              |
| Checked by find    | First                                                       | After L1 miss                                      | No                                                    |
| JPQL avoids DB?    | Usually no; query still executes                            | Not by itself                                      | Possible exact cache hit                              |
| Object identity    | Guarantees == within context                                | No shared objects; L1 assembles identity           | Returned entity ids resolve through L1/L2             |
| Transactional role | Unit of work                                                | Optimization with concurrency strategy             | Optimization with invalidation timestamps             |
| Staleness          | Can be stale after bulk/external update until refresh/clear | Can be stale with bad invalidation/external writes | Can be stale/low-hit; invalidated by affected spaces  |
| Best use           | All entity work                                             | Stable read-mostly reused data                     | Repeated identical expensive queries with stable data |
| Clear/evict        | clear/detach/close                                          | Region/entity eviction APIs                        | Query-region eviction                                 |

## Flush versus commit

| **Question**                   | **Flush**                                                          | **Commit**                                        |
|--------------------------------|--------------------------------------------------------------------|---------------------------------------------------|
| What boundary?                 | ORM-to-database synchronization inside transaction                 | Database transaction completion                   |
| Can DML execute?               | Yes                                                                | Yes; usually flushes first                        |
| Visible to other transactions? | Depends on isolation; normally uncommitted changes are not visible | Yes after success                                 |
| Can rollback undo it?          | Yes                                                                | Not after successful commit                       |
| Can constraints fail?          | Yes; immediate constraints and DB errors                           | Yes; deferred constraints and commit failures too |
| Does L1 close?                 | No                                                                 | Often after transaction-scoped cleanup            |
| Who triggers it?               | Explicit call, AUTO query, provider synchronization                | Transaction manager / resource transaction        |

<nav class="page-nav page-nav-bottom">
[← Payment-Service Execution Traces](/docs/11-traces/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Twenty Difficult Interview Questions →](/docs/13-interview-questions/)
</nav>
