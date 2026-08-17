---
layout: default
title: "Persistence Context and First-Level Cache"
description: "Hibernate + JPA Internals - Persistence Context and First-Level Cache"
permalink: /docs/03-persistence-context/
---

<nav class="page-nav">
[← Entity Lifecycle](/docs/02-entity-lifecycle/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Dirty Checking Internals →](/docs/04-dirty-checking/)
</nav>

## 3. Persistence context and mandatory first-level cache

## Identity map

Hibernate stores managed entities under an EntityKey roughly equivalent to (entity persister/type, identifier). It also tracks loaded-state snapshots, entity status, version, collection wrappers, proxies, and queued actions. Calling this merely a “cache” understates its role: it is the unit of work and identity map.

| **JPA GUARANTEE** Within one persistence context, repeated access to the same persistent identity returns the same managed Java object (==), barring explicit detachment/clear or special projection/native cases. |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

| **Operation**                  | **L1 behavior**                                                                               | **Database behavior**                                                                                               |
|--------------------------------|-----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| find(Payment.class, 42)        | Return existing managed instance if present.                                                  | No SELECT on L1 hit.                                                                                                |
| JPQL select p where p.id=42    | Query normally executes to determine result rows/ids; hydration resolves identity through L1. | SELECT usually executes, but result element is existing managed object; query row values do not blindly replace it. |
| getReference(Payment.class,42) | May create/return proxy associated with EntityKey.                                            | No SELECT until non-identifier state is accessed; missing row may fail then.                                        |
| refresh(payment)               | Bypasses ordinary “keep current state” behavior and overwrites it.                            | SELECT executes.                                                                                                    |
| clear() then find(42)          | L1 is empty; new managed instance created.                                                    | L2 or SELECT supplies state.                                                                                        |

## Why mandatory and why not shared

- Identity: two Java objects for one row in one unit of work would make dirty checking, relationships, cascades, and write ordering ambiguous.

- Repeatable application-level reference: even under READ COMMITTED, the same managed instance remains until refreshed/cleared; this is not the same as database repeatable-read semantics for arbitrary queries.

- Isolation: sharing mutable managed objects across requests would violate thread safety and transaction isolation. Therefore L1 is per EntityManager/Session, not SessionFactory-wide.

- Lifecycle correctness: snapshots and action queue belong to one unit of work and must be completed or abandoned together.

## Memory growth in batch jobs

Every loaded/inserted entity, snapshot, collection wrapper, and pending action consumes memory. Processing 100,000 rows in one context can become O(n) memory and make dirty checking O(n) at flush. Stream/page through rows, periodically flush and clear, avoid retaining references elsewhere, and control JDBC fetch size. StatelessSession is an advanced alternative when identity map, cascades, and dirty checking are unnecessary.

<nav class="page-nav page-nav-bottom">
[← Entity Lifecycle](/docs/02-entity-lifecycle/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Dirty Checking Internals →](/docs/04-dirty-checking/)
</nav>
