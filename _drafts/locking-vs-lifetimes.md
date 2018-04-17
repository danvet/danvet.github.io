---
layout: post
title: "Locking vs. Lifetimes"
tags:
- In-Depth Tech
---

notifier fun
- both ways -> locking inversion
- register races -> BKL

destroying stuff (kref_put and friends)
- weak vs. strong references
- kref_put_mutex vs. kref_get_unless_zero
- temporary references for weak references

ordering vs locking
- registering/unregistering/cleanup
- worker vs. locking

Summary
- Avoid creating locks that implicitly serialize object
  construction/destruction.
- Handle all lifetime needs explicitly, through refcounting and ordering.
- RCU is fun
