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
manipulating data structures, and not just sprinklock <code>mutex_lock()</code>
and <code>mutex_unlock()</code> calls around until the result looks reasonable
and lockdep has gone quiet.
<!--more-->

## Level 0: No Locking

The dumbest possible locking is no need for locking at all. Which does not mean
extremely clever lockless tricks for a "look, not calls to
<code>mutex_lock()</code>" feint, but an overall design which guarantees that
any writers cannot exist concurrently with any other access at all. This
removes the need for consistency guarantees while accessing an object at the
architecturaly level.

There's a few standard patterns to achieve locking nirvana.

### Locking Pattern: Immutable State

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

### Locking Pattern: Single Owner

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

### Locking Pattern: Reference Counting

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
At that point you add a single lock for each logical object. An object could be
just a single structure, but it could also be multiple structure that are
dynamically allocated and freed under the protection of that single big dumb
lock, e.g. when managing GPU virtual address space with different mappings.

The tricky part is figuring out what is an object to ensure that your lock is
neither too big nor too small:

* If you make your lock too big you run the risk of creating a dreaded subsystem
  lock, or violating the ["Protect Data, not
  Code"](/2022/07/locking-engineering.html#protect-data-not-code) principle in
  some other way. Split your locking further so that a single lock really only
  protects a single object, and not a random collection of unrelated ones. So
  one lock per device instance, not one lock for all the device instances in a
  driver or worse in an entire subsystem.

  The trouble is that once a lock is too big and has firmly moved into "protects
  some vague collection of code" territory, it's very hard to get out of that
  hole.

* Different problems strike when the locking scheme is too fine-grained, e.g. in
  the GPU virtual memory management example when every address mapping in the
  big vma tree has its own private lock. Or when a structure has a lot of
  different locks for different member fields.

  On one hand locks aren't free, the overhead of fine grained locking can
  seriously hurt, especially when common operations have to take most of the
  locks anyway and so there's no chance of any concurrence benefit. Furthermore
  fine-grained locking leads to the temption of solving locking overhead with
  ever more clever lockless tricks, instead of radically simplifying the
  approach.

  The other main issue is that more locks improve the odds for lockiing
  inversions, and those can be tough nuts to crack. Again trying to solve this
  with more lockless tricks to avoid inversions is tempting, and again in most
  cases the wrong approach.

Ideal big dumb locking therefore needs to be right-sized everytime the
requirements on the datastructures changes. If not done there's a high chance
the locking complexity tailspinning into ever bigger locking, or ever smaller
more complex locking, depending upon on which side of the optimal track you've
fallen of the wagon train.

<h2 style="background:yellow;"> Level 2: Fine-grained Locking</h2>

It would be great if this is all the locking we ever need, but sometimes there's
functional reasons that force us to go beyond the single lock for each logical
object approach. This section will go through a few of the common examples, and
the usual pitfalls to avoid.

But before we dwelve into details remember to document in kerneldoc with the
inline per-member kerneldoc comment style once you go beyond a simple single
lock per object approach. It's the best place for future bug fixers and
reviewers - meaning you - to find the rules for how at least things were meant
to work.

### Locking Pattern: Object Tracking Lists

One of the main duties of the kernel is to track everything, least to make sure
there's no leaks and everything gets cleaned up again. But there's other reasons
to maintain lists (or other container structures) of objects.

Now sometimes there's a clear parent object, with its own lock, which could also
protect the list with all the objects, but this does not always work:

* It might force the lock of the parent object to essentially become a subsystem
  lock and so protect much more than it should when following the ["Protect Data, not
  Code"](/2022/07/locking-engineering.html#protect-data-not-code) principle. In
  that case it's better to have a separate (spin-)lock just for the list to be
  able to clearly untangle what the parent and subordinate object's lock each
  protect.

* Different code paths might need to walk and possibly manipulate the list both
  from the container object and contained object, which would lead to locking
  inversion if the list isn't protected by it's own stand-alone (nested) lock.
  This tends to especially happen when an object can be attached to multiple
  other objects, like a GPU buffer object can be mapped into multiple GPU
  virtual address spaces of different processes.

* The calling contexts for adding or removing objects from the list walking
  need itself. The main example here are LRU lists where the shrinker needs to
  be able to walk the list from reclaim context, whereas the superior object
  locks often have a need to allocate memory while holding each lock. Those
  object locks the shrinker can then only trylock, which is generally good
  enough, but only being able to trylock the LRU is not.

Simplicity should still win, therefore only add a (nested) lock for lists or
other container objects if there's really no suitable object lock that could do
the same job.

### Locking Pattern: Interrupt Handler State

Another example that requires nested locking is when part of the object is
manipulated from a different execution context. The prime example here are
interrupt handlers. Interrupt handlers can only use interrupt safe spinlocks,
but often the main object lock must be a mutex to allow sleeping or allocating
memory or nesting with other mutexes.

Hence the need for a nested spinlock to just protect the object state shared
between the interrupt handler and code running from process context. Process
context should generally only acquire the spinlock nested with the main object
lock, to avoid surprises and limit any concurrency issues to just the singleton
interrupt handler.

### Locking Pattern: Async Processing

Very similar is coordination with async workers. The best approach is the [single
owner pattern](#locking-pattern-single-owner), but often state needs to be shared between the worker and other
threads operating on the same object.

The naive approach of just using a single object lock tends to deadlock:

```
start_processing(obj)
{
	mutex_lock(&obj->lock);
	/* set up the data for the async work */;
	schedule_work(&obj->work);
	mutex_unlock(&obj->lock);
}

stop_processing(obj)
{
	mutex_lock(&obj->lock);
	/* clear the data for the async work */;
	cancel_work_sync(&obj->work);
	mutex_unlock(&obj->lock);
}

work_fn(work)
{
	obj = container_of(work, work);

	mutex_lock(&obj->lock);
	/* do some processing */
	mutex_unlock(&obj->lock);
}
```

There's a bunch of variations of this theme, with problems in different
scenarios:

* Replacing the <code>cancel_work_sync()</code> with <code>cancel_work()</code>
  avoids the deadlock, but often means the <code>work_fn()</code> is prone to
  use-after-free issues.

* Calling <code>cancel_work_sync()</code>before taking the mutex can work in
  some cases, but falls apart when the work is self-rearming. Or maybe the
  work or overall object isn't guaranteed to exist without holding it's lock,
  e.g. if this is part of an async processing queue for a parent structure.

* Cancelling the work after the call to <code>mutex_unlock()</code> might race
  with concurrent restarting of the work and upset the bookkeeping.

Like with interrupt handlers the clean solution tends to be an additional nested
lock which protects just the mutable state shared with the work function and
nests within the main object lock. That way work can be cancelled while the main
object lock, which avoids a ton of races, but without holding the sublock that
<code>work_fn()</code> needs, which avoids the deadlock.

Note that in some cases the superior lock doesn't need to exist, e.g.
<code>struct drm_connector_state</code> is protected by the [single
owner pattern](#locking-pattern-single-owner), but drivers might have some need
for some further decoupled asynchronous processing, e.g. for handling the
content protect or link training machinery.

### Locking Pattern: Weak References

[Reference counting](#locking-pattern-refernce-counting) is a great pattern, but
sometimes you need be able to store pointers without them holding a full
reference. This could be for lookup caches, or because your userspace API
mandates that some references do not keep the object alive - we've unfortunately
committed that mistake in the GPU world. Or because holding full references
everywhere would lead to unreclaimable references loops and there's no better
way to break them than to make some of the references weak.

Since weak references are such a standard pattern <code>struct kref</code> has
ready-made support for them:

## Locking Antipattern: Confusing Object Lifetime and Consistency

<h2 style="background:orange;"> Level 2.5: Splitting Locks for Performance
Reasons</h2>

- read-write locks might apply

<h2 style="background:red"> Level 3: Lockless Tricks</h2>

Do not go here wanderer!


