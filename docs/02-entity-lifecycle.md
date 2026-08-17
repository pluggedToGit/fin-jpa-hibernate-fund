---
layout: default
title: "Entity Lifecycle"
description: "Hibernate + JPA Internals - Entity Lifecycle"
permalink: /docs/02-entity-lifecycle/
---

<nav class="page-nav">
[← JPA vs Hibernate](/docs/01-jpa-vs-hibernate/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Persistence Context and First-Level Cache →](/docs/03-persistence-context/)
</nav>

## 2. Entity lifecycle: object and row are different things

| **State**            | **Java object / persistence context**                                                             | **Database row**                                                                                        |
|----------------------|---------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Transient            | Ordinary object; no persistence identity tracked by this context; may have an assigned id.        | Usually no row. Hibernate has no obligation to insert it.                                               |
| Managed / persistent | Present in identity map; changes tracked; exactly one managed instance per entity type + id.      | Row may already exist, or INSERT may only be scheduled. SQL timing depends on id generation and flush.  |
| Detached             | Has identity but is no longer tracked because of detach/clear/close/serialization/end of context. | Row remains. Mutating object alone does nothing.                                                        |
| Removed              | Still associated until flush but marked for deletion.                                             | DELETE is scheduled; row usually remains until flush and is recoverable by rollback after SQL executes. |

## Operations and transitions

| **Operation** | **Input state → result**                                                 | **Java-object effect**                                                                                                | **Row / SQL effect**                                                                                  |
|---------------|--------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| persist(x)    | transient → managed                                                      | Same instance becomes managed. Cascades PERSIST. Error if detached identity conflicts.                                | INSERT scheduled; may execute early for IDENTITY or to obtain key, otherwise usually at flush.        |
| merge(x)      | transient/detached → x stays transient/detached; managed copy y returned | Hibernate finds/creates managed y and copies merge-cascaded state. Never assume x became managed.                     | SELECT may be needed; INSERT/UPDATE for y occurs on flush if needed.                                  |
| remove(x)     | managed → removed                                                        | x marked removed; cascade REMOVE may traverse graph.                                                                  | DELETE scheduled, ordered at flush; rollback restores DB, not necessarily safe reuse of object graph. |
| detach(x)     | managed → detached                                                       | Stops tracking x; pending dirty changes for x are lost unless already flushed. Cascade DETACH applies selectively.    | No row change by detach itself.                                                                       |
| refresh(x)    | managed → managed                                                        | Overwrites in-memory state from DB; pending local changes to refreshed attributes are lost. Cascade REFRESH possible. | SELECT executes; locking options may add lock.                                                        |
| clear()       | all managed/removed → detached                                           | Empties context and action associations; unflushed changes are discarded from tracking.                               | No SQL solely because of clear.                                                                       |
| close()       | context unusable; managed instances become detached                      | Releases persistence-context resources. Spring usually owns this.                                                     | Does not mean commit; transaction owner decides outcome.                                              |
| flush()       | states unchanged conceptually                                            | Synchronizes tracked changes/action queue. Objects remain managed.                                                    | Executes SQL but does not commit.                                                                     |
| find(T,id)    | none → managed or returns existing managed                               | Identity-map lookup first; may use L2, then DB.                                                                       | SELECT only on cache miss unless lock/refresh requires DB.                                            |

### The merge trap

```java
Payment detached = ...;
Payment managed = em.merge(detached);

assert detached != managed;          // normally true
detached.setStatus(APPROVED);         // not tracked
managed.setStatus(APPROVED);          // tracked
```

Spring Data JpaRepository.save chooses persist for a new entity and merge for a non-new entity (subject to its new-detection rules). Therefore save(existingDetached) returns the instance you should continue using. For an already managed entity, save is usually redundant; merge may still copy state and add overhead.

<nav class="page-nav page-nav-bottom">
[← JPA vs Hibernate](/docs/01-jpa-vs-hibernate/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Persistence Context and First-Level Cache →](/docs/03-persistence-context/)
</nav>
