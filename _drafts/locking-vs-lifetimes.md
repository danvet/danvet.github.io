---
layout: post
title: "Locking vs. Lifetimes"
tags:
- In-Depth Tech
---

notifier fun
- both ways -> locking inversion
- register races -> BKL
- case study of the madness: dma_resv

destroying stuff (kref_put and friends)
- weak vs. strong references
- kref_put_mutex vs. kref_get_unless_zero
- temporary references for weak references
- rcu with delayed freeing as the most obvious "locking as a lifetime manager"
-> don't enforce lifetimes with big locks

ordering vs locking
- registering/unregistering/cleanup
- worker vs. locking
- lockdep can't see through wait_completion/complete
-> don't enforce ordering with big locks

memory alloc vs. everything above
- special case of deadlock
- lockdep annotation vs reality
- ordering e.g. dma_fence_wait

separate concerns
- example: drm_connector - separate lock for idr/list, runtime state, and then
  refcount on top to tie it all up

Summary
- Avoid creating locks that implicitly serialize object
  construction/destruction.
- Handle all lifetime needs explicitly, through refcounting and ordering.
- RCU is fun
