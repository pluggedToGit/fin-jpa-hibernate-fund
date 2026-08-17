---
layout: default
title: "Query Behavior and Loading"
description: "Hibernate + JPA Internals - Query Behavior and Loading"
permalink: /docs/07-query-behavior/
---

<nav class="page-nav">
[← Second-Level Cache (L2) and Query Cache](/docs/06-caching/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Relationships and Aggregate Correctness →](/docs/08-relationships/)
</nav>

## 7. Query behavior and loading

| **API**        | **Strength**                                                 | **Caution**                                                                                                              |
|----------------|--------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| JPQL/HQL       | Entity-oriented, portable JPQL; Hibernate HQL adds features. | Returns managed entities unless projection/read-only; still executes query even when matching entities are in L1.        |
| Criteria API   | Programmatic, composable, type-oriented with metamodel.      | Verbose; same persistence and loading semantics as JPQL.                                                                 |
| Native SQL     | DB-specific control and features.                            | Mapping/synchronization/cache invalidation are harder; AUTO flush behavior differs by API and synchronized query spaces. |
| DTO projection | Loads only selected data; no entity dirty checking.          | Not managed; cannot navigate lazy relationships as entities.                                                             |

## Lazy, eager, proxies, and initialization

LAZY means the association may be represented by a proxy or persistent collection wrapper and initialized when accessed. EAGER is a requirement that it be loaded by the time the entity is available, not necessarily one join; providers may issue secondary selects. To-one associations default EAGER in JPA, but production models often override to LAZY where provider enhancement/proxy constraints allow.

LazyInitializationException occurs when an uninitialized proxy/collection needs a Session but the entity is detached or the Session is closed. Fix the use-case boundary: fetch the required graph inside the transaction or project to a DTO. Open Session in View postpones the symptom and can hide accidental SQL, inconsistent reads, and connection/latency problems.

## N+1 and fetch tools

| **Technique**                         | **What it does**                                                               | **Limits**                                                                              |
|---------------------------------------|--------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| JOIN FETCH                            | Loads root + association in one SQL join.                                      | To-many duplicates rows; DISTINCT/entity de-dup; multiple bags and pagination problems. |
| EntityGraph                           | Declaratively selects attributes/associations for a use case.                  | Provider generates joins/secondary loads; inspect actual SQL.                           |
| @BatchSize / default_batch_fetch_size | When one proxy/collection initializes, fetches a batch of similar pending ids. | Reduces N+1 to roughly N/batch; not one query and requires a useful access pattern.     |
| SUBSELECT fetching                    | Loads collections for owners returned by prior query using a subselect.        | Hibernate-specific and context-sensitive.                                               |
| DTO projection                        | Fetches exact columns/aggregation.                                             | No managed graph; best for read APIs.                                                   |

## Pagination with collection fetch joins

A collection join multiplies database rows, while pagination is supposed to count root entities. Applying LIMIT/OFFSET at the SQL row level can truncate a root’s collection or return fewer roots; Hibernate may warn or paginate in memory, which is dangerous. Use two-phase pagination: first page Payment ids with stable ordering, then fetch the graph for those ids; or use keyset pagination and a second fetch.

<nav class="page-nav page-nav-bottom">
[← Second-Level Cache (L2) and Query Cache](/docs/06-caching/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Relationships and Aggregate Correctness →](/docs/08-relationships/)
</nav>
