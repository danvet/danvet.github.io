---
layout: post
title: Locking Engineering Hierarchy
author: danvet
tags:
- In-Depth Tech
---

blabla
<!--more-->

## Level 0: No Locking

### Pattern: Immutable State

### Pattern: Single Owner

### Pattern: Reference Counting

## Level 1: Big Dumb Lock

<h2 style="background:yellow;"> Level 2: Fine-grained Locking</h2>

### Pattern: Object Tracking Lists

- lru lists

### Pattern: Interrupt Handler State

- irq handler

### Pattern: Weak References

- weak references

### Pattern: Async Processing

- async processing

Document in kerneldoc with the inline per-member kerneldoc comment style.

## Antipattern: Confusing Object Lifetime and Consistency

<h2 style="background:orange;"> Level 2.5: Splitting Locks for Performance
Reasons</h2>

- read-write locks might apply

<h2 style="background:red"> Level 3: Lockless Tricks</h2>

Do not go here wanderer!


