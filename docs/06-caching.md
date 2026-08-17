---
layout: default
title: "Second-Level Cache (L2) and Query Cache"
description: "Hibernate + JPA Internals - Second-Level Cache (L2) and Query Cache"
permalink: /docs/06-caching/
---

<nav class="page-nav">
[← Flushing: Synchronize, Do Not Finalize](/docs/05-flushing/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Query Behavior and Loading →](/docs/07-query-behavior/)
</nav>

## 6. Second-level cache (L2) and query cache

## Architecture

L2 is optional and scoped to the EntityManagerFactory/SessionFactory. It stores disassembled entity state by entity type and id, not shared live Java objects. On an L2 hit, each persistence context assembles its own managed instance, preserving L1 isolation and identity.

| **Cache**     | **Key → value**                                                        | **Scope**                                     | **Invalidation / risk**                                                                            |
|---------------|------------------------------------------------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------|
| L1            | (entity type,id) → managed object + tracking metadata                  | One persistence context; mandatory            | Cleared/detached/closed locally; transaction-aware state.                                          |
| Entity L2     | (entity type,id) → cached attribute state/version                      | SessionFactory / configured cluster; optional | Writes update/invalidate region entries according to strategy. External writers can make it stale. |
| Collection L2 | (role,owner id) → element ids/keys                                     | SessionFactory / cluster                      | Caches membership, not necessarily element state; both collection and entity regions matter.       |
| Query cache   | query text/plan parameters/pagination/tenant/etc. → result ids/scalars | SessionFactory / cluster                      | Invalidated by timestamps/query spaces; result ids still resolve through L1/L2/DB.                 |

## Regions, providers, and strategies

- Regions partition cache entries by entity, collection, query, and update-timestamp concerns. Region configuration controls TTL, capacity, replication/distribution, and eviction.

- Providers integrate through Hibernate’s cache SPI/JCache ecosystem; examples historically include Ehcache, Infinispan, Hazelcast, and vendor/cloud products. Compatibility with your Hibernate major version matters.

| **Strategy**         | **Use**                                                                       | **Trade-off**                                                                      |
|----------------------|-------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| READ_ONLY            | Immutable reference data.                                                     | Fastest; updating cached entity is invalid.                                        |
| NONSTRICT_READ_WRITE | Rarely updated data tolerating a staleness window.                            | Invalidation around writes is not strict transactional coherence.                  |
| READ_WRITE           | Read-mostly mutable data requiring stronger soft-lock/timestamp coordination. | More cache traffic and complexity; not serializable isolation.                     |
| TRANSACTIONAL        | Provider supports transactional cache semantics.                              | Requires compatible transaction/cache integration; highest operational complexity. |

## Clustered deployments and payment guidance

Local per-node L2 caches require invalidation/replication across nodes or tolerate inconsistent reads until expiry. Distributed caches add serialization, network, split-brain, topology, and deployment-version concerns. Database writes performed outside Hibernate must also invalidate or expire cached state; otherwise the cache can serve stale data.

| **PAYMENT RECORDS** High-churn Payment and PaymentAttempt rows, strict freshness, status transitions, version checks, and write-heavy access often make them poor L2 candidates. Cache stable FundingSource metadata only if token/security policy, invalidation, tenant boundaries, and revocation freshness are designed explicitly. Never treat L2 as the authority for balances, idempotency, or authorization. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

L2 helps when reads greatly outnumber writes, entity state is reused across many sessions, objects are small/stable, cache hit ratio is high, and invalidation is reliable. It is useless or harmful for one-off scans, high-cardinality cold data, write-heavy records, strict real-time truth, huge graphs, or queries that still hit the DB and rarely repeat.

<nav class="page-nav page-nav-bottom">
[← Flushing: Synchronize, Do Not Finalize](/docs/05-flushing/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Query Behavior and Loading →](/docs/07-query-behavior/)
</nav>
