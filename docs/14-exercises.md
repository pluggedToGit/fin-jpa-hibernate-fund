---
layout: default
title: "Ten "Predict the SQL" Exercises"
description: "Hibernate + JPA Internals - Ten "Predict the SQL" Exercises"
permalink: /docs/14-exercises/
---

<nav class="page-nav">
[← Twenty Difficult Interview Questions](/docs/13-interview-questions/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Common Production Pitfalls →](/docs/15-production-pitfalls/)
</nav>

## 14. Ten “predict the SQL and outcome” exercises

## Exercise 1

Within one transaction: find(Payment,42); find(Payment,42). L2 off.

| **ANSWER** One SELECT. Same Java instance returned. No DML if unchanged; commit. |
|----------------------------------------------------------------------------------|

## Exercise 2

find P42; set status APPROVED; execute JPQL count(Payment p where p.status=APPROVED), AUTO.

| **ANSWER** SELECT to load; UPDATE auto-flushed before COUNT because query space overlaps; COUNT sees change inside same transaction; then commit. |
|---------------------------------------------------------------------------------------------------------------------------------------------------|

## Exercise 3

persist new Payment with SEQUENCE allocation; print id; rollback before explicit flush.

| **ANSWER** Sequence/id allocation SQL may occur; INSERT may not. Transaction rolls back; sequence numbers normally are not rolled back, so gap possible. |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|

## Exercise 4

persist new Payment with IDENTITY; print id; then rollback.

| **ANSWER** INSERT usually executes immediately to obtain id. Rollback removes row, but generated identity value may be consumed. |
|----------------------------------------------------------------------------------------------------------------------------------|

## Exercise 5

load P42; detach(p); mutate p; commit.

| **ANSWER** SELECT only. Mutation is untracked; no UPDATE. DB unchanged. |
|-------------------------------------------------------------------------|

## Exercise 6

load P42; bulk JPQL sets all pending payments expired; immediately return p.status.

| **ANSWER** Bulk UPDATE executes (after relevant auto-flush). p can still report PENDING from L1. Commit stores EXPIRED unless a later stale managed flush overwrites it. |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## Exercise 7

T1/T2 load version 3. T1 flushes update but has not committed; T2 updates same row with PESSIMISTIC_WRITE request.

| **ANSWER** T1 holds row lock after UPDATE. T2’s lock/update waits or times out. After T1 commits, optimistic/version conditions determine whether T2 can update. |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## Exercise 8

Repository method without outer transaction loads Payment with lazy attempts; controller accesses attempts after return.

| **ANSWER** Repository read transaction/context ends; root SELECT occurred. Access triggers LazyInitializationException unless initialized, OSIV keeps context open, or mapping was fetched. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## Exercise 9

Loop persist 1000 SEQUENCE entities; batch_size=50; never flush until commit.

| **ANSWER** Ids allocated in blocks; L1/action queue holds 1000. At commit, inserts can execute in JDBC batches (subject to ordering/SQL shape). Memory remains O(1000). |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## Exercise 10

L2 has P42 v5. External SQL updates row to v6 without cache eviction; new context find(P42).

| **ANSWER** Potential L2 hit returns stale v5 without SELECT. A subsequent versioned UPDATE using v5 affects zero rows and fails optimistically; a read-only response may silently serve stale state. |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

<nav class="page-nav page-nav-bottom">
[← Twenty Difficult Interview Questions](/docs/13-interview-questions/) &nbsp;|&nbsp; [🏠 Home](/) &nbsp;|&nbsp; [Common Production Pitfalls →](/docs/15-production-pitfalls/)
</nav>
