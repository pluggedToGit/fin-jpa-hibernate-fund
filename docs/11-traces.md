---
layout: default
title: "Payment-Service Execution Traces"
description: "Hibernate + JPA Internals - Payment-Service Execution Traces"
permalink: /docs/11-traces/
---

<nav class="page-nav">
[← Performance Engineering](/docs/10-performance/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Comparison Tables →](/docs/12-comparison-tables/)
</nav>

## 11. Payment-service execution traces

Assumptions unless stated: transaction-scoped persistence context; FlushMode.AUTO; Payment is versioned; L2 disabled except the explicit L2 scenario; PostgreSQL-like database; SEQUENCE ids. “L1” entries are abbreviated P42 → Payment#42.

## 11.1 Loading the same entity twice in one transaction

| **Line** | **Application action**   | **Entity state / L1**                         | **L2**        | **SQL / flush / outcome**                        |
|----------|--------------------------|-----------------------------------------------|---------------|--------------------------------------------------|
| 1        | @Transactional begins    | L1 = {}                                       | none          | BEGIN; no entity SQL.                            |
| 2        | p1 = em.find(Payment,42) | p1 managed; L1={P42→p1}                       | miss/disabled | SELECT payment ... WHERE id=42.                  |
| 3        | p2 = em.find(Payment,42) | same managed instance; p1 == p2; L1 unchanged | not consulted | No SELECT.                                       |
| 4        | method returns           | dirty check: unchanged                        | none          | No DML; COMMIT; context closes → p1/p2 detached. |

| **RESULT** L1 is an identity map, not merely a result cache. Repeated find returns the identical Java reference. |
|------------------------------------------------------------------------------------------------------------------|

## 11.2 Modifying a managed entity without save()

| **Line** | **Application action** | **Entity state / L1**                        | **L2**                                | **SQL / flush / outcome**                                   |
|----------|------------------------|----------------------------------------------|---------------------------------------|-------------------------------------------------------------|
| 1        | begin + find P42       | P42 managed; snapshot status=PENDING         | miss                                  | SELECT.                                                     |
| 2        | p.setStatus(APPROVED)  | P42 managed; current differs from snapshot   | none                                  | No UPDATE yet.                                              |
| 3        | return from method     | flush dirty-checks P42; update action queued | L2 invalidated/updated if enabled     | UPDATE payment SET ... version=8 WHERE id=42 AND version=7. |
| 4        | commit                 | P42 managed until cleanup                    | strategy completes cache coordination | COMMIT; object becomes detached after context close.        |

| **RESULT** save() is unnecessary for a managed entity. Dirty checking plus transaction completion causes the update. |
|----------------------------------------------------------------------------------------------------------------------|

## 11.3 Calling save() and then flush()

| **Line** | **Application action**    | **Entity state / L1**                                                          | **L2**                   | **SQL / flush / outcome**                                              |
|----------|---------------------------|--------------------------------------------------------------------------------|--------------------------|------------------------------------------------------------------------|
| 1        | p = repo.findById(42)     | managed P42                                                                    | possible lookup          | SELECT or cache hit.                                                   |
| 2        | p.status=APPROVED         | managed dirty                                                                  | none                     | No SQL.                                                                |
| 3        | repo.save(p)              | still same managed P42; Spring Data may call merge depending API/new detection | none                     | Usually no UPDATE yet; merge on already managed instance is redundant. |
| 4        | repo.flush() / em.flush() | dirty snapshot reconciled                                                      | cache write coordination | UPDATE executes; constraint/version errors surface now.                |
| 5        | later method returns      | remains managed                                                                | none                     | COMMIT; flush did not commit.                                          |

| **RESULT** saveAndFlush is not “save permanently.” It synchronizes now, and the transaction can still roll back. |
|------------------------------------------------------------------------------------------------------------------|

## 11.4 Flush followed by rollback

| **Line** | **Application action** | **Entity state / L1**                                   | **L2**                              | **SQL / flush / outcome**                               |
|----------|------------------------|---------------------------------------------------------|-------------------------------------|---------------------------------------------------------|
| 1        | load and mutate P42    | managed dirty                                           | none                                | SELECT, then no DML yet.                                |
| 2        | em.flush()             | managed; snapshot/action state synchronized             | cache strategy protects/invalidates | UPDATE executes; row is uncommitted; locks may be held. |
| 3        | throw RuntimeException | transaction marked rollback-only                        | cache coordination aborts           | ROLLBACK.                                               |
| 4        | cleanup                | object usually detached and still says APPROVED in Java | no committed new entry              | DB row remains prior status/version.                    |

| **RESULT** Rollback restores database state, not the fields of the Java object. Do not return or reuse it as authoritative state. |
|-----------------------------------------------------------------------------------------------------------------------------------|

## 11.5 Loading in two separate transactions

| **Line** | **Application action** | **Entity state / L1** | **L2**                   | **SQL / flush / outcome**               |
|----------|------------------------|-----------------------|--------------------------|-----------------------------------------|
| 1        | Tx A find(42)          | L1A={P42→a}           | possible miss/hit        | SELECT/cache assembly.                  |
| 2        | Tx A commit/close      | a detached; L1A gone  | entry may remain         | COMMIT.                                 |
| 3        | Tx B find(42)          | L1B={P42→b}; b != a   | possible hit             | L2 hit avoids SELECT; otherwise SELECT. |
| 4        | Tx B ends              | b detached            | entry remains per policy | COMMIT.                                 |

| **RESULT** Identity equality is guaranteed only inside one persistence context. Across transactions, instances normally differ even when ids match. |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|

## 11.6 Concurrent update with @Version

| **Line** | **Application action**   | **Entity state / L1**            | **L2**                                      | **SQL / flush / outcome**                                                      |
|----------|--------------------------|----------------------------------|---------------------------------------------|--------------------------------------------------------------------------------|
| 1        | T1 and T2 find P42 v7    | separate L1s: a(v7), b(v7)       | L2 may supply v7 to each                    | Each obtains same committed version.                                           |
| 2        | T1 approves; T2 declines | both managed dirty independently | none                                        | No update until each flush.                                                    |
| 3        | T1 flush + commit        | a becomes v8                     | L2 updated/invalidated                      | UPDATE ... WHERE id=42 AND version=7 → 1 row; COMMIT.                          |
| 4        | T2 flush                 | b expected v7                    | cache not allowed to legitimize stale write | UPDATE ... WHERE id=42 AND version=7 → 0 rows; optimistic exception; ROLLBACK. |

| **RESULT** @Version converts silent last-writer-wins into a detectable conflict at flush. Retry must re-run the whole decision on fresh data. |
|-----------------------------------------------------------------------------------------------------------------------------------------------|

## 11.7 JPQL after changing a managed entity

| **Line** | **Application action**                                        | **Entity state / L1**                          | **L2**                                              | **SQL / flush / outcome**                                        |
|----------|---------------------------------------------------------------|------------------------------------------------|-----------------------------------------------------|------------------------------------------------------------------|
| 1        | find P42 then set status=APPROVED                             | P42 managed dirty                              | none                                                | SELECT only.                                                     |
| 2        | execute JPQL: select p from Payment p where p.status=APPROVED | AUTO evaluates overlapping Payment query space | none                                                | Auto-flush UPDATE first.                                         |
| 3        | execute query                                                 | L1 still maps P42 to same object               | query cache typically bypass/invalidated for writes | SELECT ... WHERE status=APPROVED; hydration reuses P42 instance. |
| 4        | commit                                                        | managed until close                            | cache coordination completes                        | COMMIT.                                                          |

| **RESULT** AUTO flush makes the database query reflect pending Payment changes. A query on an unrelated entity may not trigger a flush in Hibernate AUTO. |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|

## 11.8 Bulk update while an older entity remains in L1

| **Line** | **Application action**                               | **Entity state / L1**                                                                   | **L2**                                | **SQL / flush / outcome**                                                                                          |
|----------|------------------------------------------------------|-----------------------------------------------------------------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| 1        | p=find P42, status=PENDING                           | L1={P42 pending}                                                                        | possible read                         | SELECT.                                                                                                            |
| 2        | JPQL update Payment p set p.status=EXPIRED where ... | p remains managed with old field                                                        | affected L2/query regions invalidated | Bulk UPDATE executes directly; count returned.                                                                     |
| 3        | read p.getStatus()                                   | still PENDING in Java                                                                   | L2 irrelevant because L1 wins         | No SELECT.                                                                                                         |
| 4        | later dirty change/flush                             | stale p can overwrite fields; version behavior depends on bulk statement/version update | none                                  | Potential lost overwrite or optimistic behavior; bulk DML does not automatically increment version unless written. |
| 5        | em.clear(); find again                               | old p detached; new managed instance                                                    | cache must be invalidated             | SELECT returns EXPIRED.                                                                                            |

| **RESULT** Flush pending work first if needed, run bulk DML, then clear or refresh. Prefer bulk statements that update version where concurrency requires it. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|

## 11.9 Reading an entity from L2 cache

| **Line** | **Application action** | **Entity state / L1**                       | **L2**                                | **SQL / flush / outcome**                           |
|----------|------------------------|---------------------------------------------|---------------------------------------|-----------------------------------------------------|
| 1        | new Tx; find P42       | L1 empty                                    | lookup entity-region P42 → hit        | No payment SELECT.                                  |
| 2        | assemble entity        | L1={P42→p}; p is a new per-session instance | cached disassembled state copied      | No SQL; lazy associations remain separate concerns. |
| 3        | find P42 again         | same p returned from L1                     | not consulted again                   | No SQL.                                             |
| 4        | commit                 | p detached                                  | L2 entry survives according to policy | COMMIT.                                             |

| **RESULT** L2 never shares a live managed object. It supplies state used to create a distinct managed instance inside the current L1. |
|---------------------------------------------------------------------------------------------------------------------------------------|

## 11.10 Processing 100,000 entities in a batch

| **Line** | **Application action**          | **Entity state / L1**                     | **L2**                        | **SQL / flush / outcome**                                                                             |
|----------|---------------------------------|-------------------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------|
| 1        | stream/page ids; batch size=500 | L1 starts empty                           | disable/bypass cache for scan | SELECT pages/stream with fetch-size.                                                                  |
| 2        | for each payment mutate         | L1 and snapshots grow toward 500          | avoid polluting L2            | No UPDATE until flush threshold.                                                                      |
| 3        | every 500: em.flush()           | dirty-check ≤500; execute batches         | invalidate relevant entries   | Batched UPDATEs; still uncommitted if one transaction.                                                |
| 4        | every 500: em.clear()           | L1 becomes {}; prior references detached  | none                          | No SQL from clear.                                                                                    |
| 5        | repeat to 100k                  | bounded context memory                    | none                          | Many batches.                                                                                         |
| 6        | commit/chunk commit             | single commit or safer chunk transactions | none                          | One giant transaction has huge undo/WAL/locks and all-or-nothing failure; chunking changes atomicity. |

| **RESULT** flush+clear bounds Hibernate memory, not transaction-log/lock duration. Choose transaction chunking based on restartability, idempotency, and atomicity requirements. |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

<nav class="page-nav page-nav-bottom">
[← Performance Engineering](/docs/10-performance/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Comparison Tables →](/docs/12-comparison-tables/)
</nav>
