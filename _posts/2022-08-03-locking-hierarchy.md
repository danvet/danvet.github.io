---
layout: post
title: Locking Engineering Hierarchy
author: danvet
tags:
- In-Depth Tech
---
The first part of this series covered [principles of locking
engineering](/2022/07/locking-engineering.html). This part goes through a pile
of locking pattern and designs, from most favourable and easiest to adjust and
hence resulting in a long term maintainable code base, to the least favourable
since hardest to ensure it works correctly and stays that way while the code
evolves. For convenience even color coded, with the dangerous levels getting
progressively more crispy red indicating how close to the burning fire you are!
Think of it as Dante's Inferno, but for locking.

As a reminder from the intro of the first part, with locking engineering I mean
the art of ensure that there's sufficient consistency in reading and
manipulating data structures, and not just sprinklock <code>mutex_lock</code>
and <code>mutex_unlock</code> calls around until the result looks reasonable and
lockdep has gone quiet.
<!--more-->

## Level 0: No Locking

The dumbest possible locking is no need for locking at all. Which does not mean
extremely clever lockless tricks for a "look, not calls to
<code>mutex_lock()</code>" feint, but an overall design which guarantees that
any writers cannot exist concurrently with any other access at all. This
removes the need for consistency guarantees while accessing an object at the
architecturaly level.

There's a few standard patterns to achieve locking nirvana.

### Pattern: Immutable State

_The_ lesson in graphics API design over the last decide is that immutable state
objects rule, because they both lead to simpler driver stacks and also better
performance. Vulkan instead of the OpenGL with it's ridiculous amount of
mutable and implicit state is the big example, but atomic instead of legacy
kernel mode setting or Wayland instead of the X11 are also built on the
assumption that immutable state objects are a Great Thing (tm).

The usual pattern is:

1. A single thread fully constructs an object, including any sub structures and
anything else you might need. Often subsytems provide initialization helpers for
objects that driver can subclass through embedding, e.g.
<code>drm_connector_init()</code> for initializing a kernel modesetting output
object. Often the subsystem provides additional functions to set up different
or optional aspects of an object, e.g.
<code>drm_connector_attach_encoder()</code> sets up the invariant links to the
preceeding element in a kernel modesetting display chain.

2. The fully formed object is published to the world, in the kernel this often
happens by registering it under some kind of identifier. This could be a global
identifier like <code>register_chrdev()</code> for character devices, something attached to a device like
registering on a driver like <code>drm_connector_register()</code> or some
<code>struct xarray</code> in the file private structure. Note that this step
here requires memory barriers of some sort, which means if you hand roll it
instead of using existing standard interfaces you are on a fast path to level
3 locking hell. Don't do that.

3. From this point there is not consistency issues anymore and all threads can
access the object without any locking.

### Pattern: Single Owner

Another way to ensure there's no concurrent access is by only allowing one
thread to own an object at a given point of time, and have well defined handover
points if that is necessary.

Most often this pattern is used for asynchronously processing a userspace
request:

1. The syscall or IOCTL constructs an object with suifficient information to
process the userspace's request.

2. That object is handed over to a worker thread with e.g.
<code>queue_work()</code>.

3. The worker thread is now the sole owner of that piece of memory and can do
whatever it feels like with it.

Again the second step requires memory barriers, which means if you hand roll
your own lockless queue you're firmly in level 3 territory and wont get rid of
the burned in red hot afterglow in your retina for quite some time. Use standard
interfaces like <code>struct completion</code> or even better libraries like the
workqueue subsystem here.

Note that the handover can also be chained or split up, e.g. for a nonblocking
atomic kernel modeset request there's three asynchronous processing pieces
involved:

* The main worker, which pushes the display state update to the hardware.

* The userspace completion event handling built around <code>struct
  drm_pending_event</code> and generally handed off to the interrupt handler of
  the driver from the main worker and processed in the interrupt handler.

* The cleanup of the no longer used old scannout buffers from the preceeding
  update. The synchronization between the preceeding update and the cleanup is
  done through <code>struct completion</code> to ensure that there's only ever a
  single worker which owns a state structure and is allowed to change it.

### Pattern: Reference Counting

Users generally don't appreciate if the kernel leaks memory too much, and
cleaning up objects by freeing their memory and releasing any other resources
tends to be an operation of the very much mutable kind. Reference counting to
the rescue!

* Every pointer to the reference counted object must guarantee that a reference
  exists for as long as the pointer is in use. Usually that's done by calling
  <code>kref_get()</code> when making a copy of the pointer, but implied
  refernces by e.g. continuing to hold a lock that protects a different pointer
  are fine too.

* The cleanup code runs when the last reference is released with
  <code>kref_put()</code>. Note that this again requires memory barriers to work
  correctly, which means if you're not using <code>struct kref</code> then it's
  safe to assume you've screwed up.

Note that this scheme falls apart when released objects are put into some kind
of cache and can be resurrected. In that case your cleanup code needs to somehow
deal with these zombies and ensure there's no confusion, and vice versa any code
that resurrects a zombie needs to deal wooden spikes the cleanup code might
throw at an inopportune time. The worst example of this kind is
<code>SLAB_TYPESAFE_BY_RCU</code>, where readers only protected with
<code>rcu_read_lock()</code> need to deal with objects potentially going through
multiple zombie resurrections, while they're trying to figure out what is going
on. Unfortunately <code>struct dma_fence</code>, mostly used in the GPU
subsystem, is operating under these rules and has lead to lots of sorrow,
wailing and ill-tempered maintainers.

Hence use standard reference count, and don't be tempted by the siren of trying
to implement clever caching of any kind.

## Level 1: Big Dumb Lock

It would be great if nothing ever changes, but sometimes that cannot be avoided.


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


