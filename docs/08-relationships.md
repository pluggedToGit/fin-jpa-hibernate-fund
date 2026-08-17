---
layout: default
title: "Relationships and Aggregate Correctness"
description: "Hibernate + JPA Internals - Relationships and Aggregate Correctness"
permalink: /docs/08-relationships/
---

<nav class="page-nav">
[← Query Behavior and Loading](/docs/07-query-behavior/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Transactions and Concurrency →](/docs/09-transactions/)
</nav>

## 8. Relationships and aggregate correctness

## Owning side and mappedBy

The owning side is the mapping that writes the FK/join table. mappedBy names the Java property on the owning side; it does not mean “parent.” For Payment(1)–PaymentAttempt(many), the @ManyToOne PaymentAttempt.payment usually owns payment_id. Payment.attempts is inverse with mappedBy="payment". Changing only the inverse collection does not reliably update the FK; helper methods must update both sides in memory.

```java
void addAttempt(PaymentAttempt attempt) {
    attempts.add(attempt);
    attempt.setPayment(this);
}

void removeAttempt(PaymentAttempt attempt) {
    attempts.remove(attempt);
    attempt.setPayment(null);
}
```

| **Mapping**         | **Typical physical model**                   | **Main warning**                                                                          |
|---------------------|----------------------------------------------|-------------------------------------------------------------------------------------------|
| @ManyToOne          | FK on many-side table.                       | Default EAGER in JPA; set LAZY intentionally. Owning side writes FK.                      |
| @OneToMany mappedBy | Inverse collection; FK lives on child.       | Initialize collection; update both sides; avoid unbounded eager load.                     |
| @OneToOne           | Unique FK or shared primary key via @MapsId. | Choose owner deliberately; optional/proxy behavior can force loads.                       |
| @ManyToMany         | Join table containing both FKs.              | Often hides a real domain entity with attributes; use explicit link entity when possible. |

## Cascade versus orphan removal

Cascade propagates an EntityManager operation across an object graph; it is not database cascade and not a fetch setting. PERSIST, MERGE, REMOVE, REFRESH, DETACH, ALL should follow aggregate ownership. Cascade REMOVE across shared entities (especially many-to-many) is dangerous. orphanRemoval=true means a privately owned child removed from the relationship is scheduled for deletion, not merely FK nulling. It is appropriate for PaymentAttempt only if attempts cannot exist independently and history-retention rules permit deletion—which payment audit records often do not.

## equals/hashCode

- Database-generated ids are null before persist and later assigned; hashing on id can change while an entity is inside a HashSet.

- Reference equality is stable but detached copies representing the same row compare unequal.

- Business keys can work if immutable, unique, and available at construction; mutable fields must not participate.

- Proxies complicate getClass(); use a proxy-safe strategy consistent with your provider and inheritance. Never include lazy associations/collections in equals, hashCode, or toString.

<nav class="page-nav page-nav-bottom">
[← Query Behavior and Loading](/docs/07-query-behavior/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Transactions and Concurrency →](/docs/09-transactions/)
</nav>
