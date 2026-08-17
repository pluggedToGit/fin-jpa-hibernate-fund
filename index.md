---
layout: default
title: "Hibernate + JPA Internals"
description: "A payment-service field guide for Senior and Staff Java Backend Engineer interviews"
permalink: /
---

# Hibernate + JPA Internals

A payment-service field guide for Senior / Staff Java Backend interviews.

> **Core mental model:** A transaction is the database atomicity boundary. A persistence context is Hibernate's in-memory unit of identity and change tracking. Flush synchronizes that state to SQL; commit makes the database transaction durable.

*Baseline: Jakarta Persistence 3.x, Hibernate ORM 6.x, Spring Data JPA*

---

## Sections

| # | Topic |
|---|-------|
| 1 | [JPA vs Hibernate](/docs/01-jpa-vs-hibernate/) |
| 2 | [Entity Lifecycle: Object and Row are Different Things](/docs/02-entity-lifecycle/) |
| 3 | [Persistence Context and First-Level Cache](/docs/03-persistence-context/) |
| 4 | [Dirty Checking Internals](/docs/04-dirty-checking/) |
| 5 | [Flushing: Synchronize, Do Not Finalize](/docs/05-flushing/) |
| 6 | [Second-Level Cache (L2) and Query Cache](/docs/06-caching/) |
| 7 | [Query Behavior and Loading](/docs/07-query-behavior/) |
| 8 | [Relationships and Aggregate Correctness](/docs/08-relationships/) |
| 9 | [Transactions and Concurrency](/docs/09-transactions/) |
| 10 | [Performance Engineering](/docs/10-performance/) |
| 11 | [Payment-Service Execution Traces](/docs/11-traces/) |
| 12 | [Comparison Tables](/docs/12-comparison-tables/) |
| 13 | [Twenty Difficult Interview Questions](/docs/13-interview-questions/) |
| 14 | [Ten "Predict the SQL" Exercises](/docs/14-exercises/) |
| 15 | [Common Production Pitfalls](/docs/15-production-pitfalls/) |
| 16 | [Concise Revision Sheet](/docs/16-revision-sheet/) |

---

## How to use this guide

Read chapters 1–5 in order; together they form the execution model. Then use the payment traces as a debugger's view of the same ideas. The interview questions, prediction exercises, pitfalls, and revision sheet are designed for active recall.

| Layer | Question it answers |
|-------|---------------------|
| Spring transaction | When is the EntityManager bound, when is the JDBC transaction committed or rolled back? |
| Persistence context / L1 | Which Java instance represents a database identity, and what changes are pending? |
| Hibernate action queue | Which INSERT/UPDATE/DELETE operations are scheduled and in what order? |
| JDBC / database | Which SQL has executed, what locks/constraints apply, and what is durable? |
| L2 / query cache | Can state or identifiers be reused across persistence contexts? |
