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
the art of ensuring that there's sufficient consistency in reading and
manipulating data structures, and not just sprinkling <code>mutex_lock()</code>
and <code>mutex_unlock()</code> calls around until the result looks reasonable
and lockdep has gone quiet.
<!--more-->

## Level 0: No Locking

The dumbest possible locking is no need for locking at all. Which does not mean
extremely clever lockless tricks for a "look, no calls to
<code>mutex_lock()</code>" feint, but an overall design which guarantees that
any writers cannot exist concurrently with any other access at all. This
removes the need for consistency guarantees while accessing an object at the
architectural level.

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
object. Additional functions can set up different or optional aspects of an
object, e.g.  <code>drm_connector_attach_encoder()</code> sets up the invariant
links to the preceding element in a kernel modesetting display chain.

2. The fully formed object is published to the world, in the kernel this often
happens by registering it under some kind of identifier. This could be a global
identifier like <code>register_chrdev()</code> for character devices, something attached to a device like
registering a new display output on a driver with
<code>drm_connector_register()</code> or some <code>struct xarray</code> in the
file private structure. Note that this step here requires memory barriers of
some sort. If you hand roll the data structure like a list or lookup
tree with your own fancy locking scheme instead of using existing standard
interfaces you are on a fast path to level 3 locking hell. Don't do that.

3. From this point on there are no consistency issues anymore and all threads
can access the object without any locking.

### Locking Pattern: Single Owner

Another way to ensure there's no concurrent access is by only allowing one
thread to own an object at a given point of time, and have well defined handover
points if that is necessary.

Most often this pattern is used for asynchronously processing a userspace
request:

1. The syscall or IOCTL constructs an object with sufficient information to
process the userspace's request.

2. That object is handed over to a worker thread with e.g.
<code>queue_work()</code>.

3. The worker thread is now the sole owner of that piece of memory and can do
whatever it feels like with it.

Again the second step requires memory barriers, which means if you hand roll
your own lockless queue you're firmly in level 3 territory and won't get rid of
the burned in red hot afterglow in your retina for quite some time. Use standard
interfaces like <code>struct completion</code> or even better libraries like the
workqueue subsystem here.

Note that the handover can also be chained or split up, e.g. for a nonblocking
atomic kernel modeset requests there's three asynchronous processing pieces
involved:

* The main worker, which pushes the display state update to the hardware and
  which is enqueued with <code>queue_work()</code>.

* The userspace completion event handling built around <code>struct
  drm_pending_event</code> and generally handed off to the interrupt handler of
  the driver from the main worker and processed in the interrupt handler.

* The cleanup of the no longer used old scanout buffers from the preceding
  update. The synchronization between the preceding update and the cleanup is
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
  references by e.g. continuing to hold a lock that protects a different pointer
  are often good enough too for a temporary pointer.

* The cleanup code runs when the last reference is released with
  <code>kref_put()</code>. Note that this again requires memory barriers to work
  correctly, which means if you're not using <code>struct kref</code> then it's
  safe to assume you've screwed up.

Note that this scheme falls apart when released objects are put into some kind
of cache and can be resurrected. In that case your cleanup code needs to somehow
deal with these zombies and ensure there's no confusion, and vice versa any code
that resurrects a zombie needs to deal the wooden spikes the cleanup code might
throw at an inopportune time. The worst example of this kind is
<code>SLAB_TYPESAFE_BY_RCU</code>, where readers that are only protected with
<code>rcu_read_lock()</code> may need to deal with objects potentially going
through simultaneous zombie resurrections, potentially multiple times, while
the readers are trying to figure out what is going on. This generally leads to 
lots of sorrow, wailing and ill-tempered maintainers, as the GPU subsystem
has and continues to experience with <code>struct dma_fence</code>.

Hence use standard reference counting, and don't be tempted by the siren of
trying to implement clever caching of any kind.

## Level 1: Big Dumb Lock

It would be great if nothing ever changes, but sometimes that cannot be avoided.
At that point you add a single lock for each logical object. An object could be
just a single structure, but it could also be multiple structures that are
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

  One issue is that locks aren't free, the overhead of fine-grained locking can
  seriously hurt, especially when common operations have to take most of the
  locks anyway and so there's no chance of any concurrency benefit. Furthermore
  fine-grained locking leads to the temption of solving locking overhead with
  ever more clever lockless tricks, instead of radically simplifying the
  design.

  The other issue is that more locks improve the odds for locking inversions,
  and those can be tough nuts to crack. Again trying to solve this with more
  lockless tricks to avoid inversions is tempting, and again in most cases the
  wrong approach.

Ideally, your big dumb lock would always be right-sized everytime the
requirements on the datastructures changes. But working magic 8 balls tend to be
on short supply, and you tend to only find out that your guess was wrong when
the pain of the lock being too big or too small is already substantial. The
inherent struggles of resizing a lock as the code evolves then keeps pushing you
further away from the optimum instead of closer. Good luck!

<h2 style="background:yellow;"> Level 2: Fine-grained Locking</h2>

It would be great if this is all the locking we ever need, but sometimes there's
functional reasons that force us to go beyond the single lock for each logical
object approach. This section will go through a few of the common examples, and
the usual pitfalls to avoid.

But before we delve into details remember to document in kerneldoc with the
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

* The constraints of calling contexts for adding or removing objects from the
  list could be different and incompatible from the requirements when walking
  the list itself. The main example here are LRU lists where the shrinker needs
  to be able to walk the list from reclaim context, whereas the superior object
  locks often have a need to allocate memory while holding each lock. Those
  object locks the shrinker can then only trylock, which is generally good
  enough, but only being able to trylock the LRU list lock itself is not.

Simplicity should still win, therefore only add a (nested) lock for lists or
other container objects if there's really no suitable object lock that could do
the job instead.

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

Very similar to the interrupt handler problems is coordination with async
workers. The best approach is the [single owner
pattern](#locking-pattern-single-owner), but often state needs to be shared
between the worker and other threads operating on the same object.

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

Do not worry if you don't spot the deadlock, because it is a cross-release
dependency between the entire <code>work_fn()</code> and
<code>cancel_work_sync()</code> and these are a lot trickier to spot. Since
cross-release dependencies are a entire huge topic on themselves I won't go into
more details, a good starting point is [this LWN
article](https://lwn.net/Articles/709849/).

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
object lock is held, which avoids a ton of races. But without holding the
sublock that <code>work_fn()</code> needs, which avoids the deadlock.

Note that in some cases the superior lock doesn't need to exist, e.g.
<code>struct drm_connector_state</code> is protected by the [single
owner pattern](#locking-pattern-single-owner), but drivers might have some need
for some further decoupled asynchronous processing, e.g. for handling the
content protect or link training machinery. In that case only the sublock for
the mutable driver private state shared with the worker exists.

### Locking Pattern: Weak References

[Reference counting](#locking-pattern-reference-counting) is a great pattern, but
sometimes you need be able to store pointers without them holding a full
reference. This could be for lookup caches, or because your userspace API
mandates that some references do not keep the object alive - we've unfortunately
committed that mistake in the GPU world. Or because holding full references
everywhere would lead to unreclaimable references loops and there's no better
way to break them than to make some of the references weak. In languages with a
garbage collector weak references are implemented by the runtime, and so no real
worry. But in the kernel the concept has to be implemented by hand.

Since weak references are such a standard pattern <code>struct kref</code> has
ready-made support for them. The simple approach is using
<code>kref_put_mutex()</code> with the same lock that also protects the
structure containing the weak reference. This guarantees that either the weak
reference pointer is gone too, or there is at least somewhere still a strong
reference around and it is therefore safe to call <code>kref_get()</code>. But
there are some issues with this approach:

* It doesn't compose to multiple weak references, at least if they are protected
  by different locks - all the locks need to be taken before the final
  <code>kref_put()</code> is called, which means minimally some pain with lock
  nesting and you get to hand-roll it all to boot.

* The mutex required to be held during the final put is the one which protects
  the structure with the weak reference, and often has little to do with the
  object that's being destroyed. So a pretty nasty violation of the [big dumb
  lock pattern](#level-1-big-dumb-lock). Furthermore the lock is held
  over the entire cleanup function, which defeats the point of the [reference
  counting pattern](#locking-pattern-reference-counting), which is meant to
  enable "no locking" cleanup code. It becomes very tempting to stuff random
  other pieces of code under the protection of this look, making it a sprawling
  mess and violating the [principle to protect data, not
  code](/2022/07/locking-engineering.html#protect-data-not-code): The
  lock held during the entire cleanup operation is protecting against that
  cleanup code doing things, and not anymore a specific data structure.

The much better approach is using <code>kref_get_unless_zero()</code>, together
with a spinlock for your data structure containing the weak reference. This
looks especially nifty in combination with <code>struct xarray</code>.

```
obj_find_in_cache(id)
{
	xa_lock();
	obj = xa_find(id);
	if (!kref_get_unless_zero(&obj->kref))
		obj = NULL;
	xa_unlock();

	return obj;
}
```

With this all the issues are resolved:

* Arbitrary amounts of weak references in any kind of structures protected by
  their own spinlock can be added, without causing dependencies between them.

* In the object's cleanup function the same spinlock only needs to be held right
  around when the weak references are removed from the lookup structure. The
  lock critical section is no longer needlessly enlarged, we're back to
  protecting data instead of code.

With both together the locking does no longer leak beyond the lookup structure
and it's associated code any more, unlike with <code>kref_put_mutex()</code> and
similar approaches.  Thankfully <code>kref_get_unless_zero()</code> has become
the much more popular approach since it was added 10 years ago!

## Locking Antipattern: Confusing Object Lifetime and Data Consistency

We've now seen a few examples where the ["no locking" patterns from level
0](#level-0-no-locking) collide in annoying ways when more locking is added to
the point where we seem to violate the [principle to protect data, not
code](/2022/07/locking-engineering.html#protect-data-not-code). It's worth to
look at this a bit closer, since we can generalize what's going on here to a
fairly high-level antipattern.

The key insight is that the "no locking" patterns all rely on memory barrier
primitives in disguise, not classic locks, to synchronize access between
multiple threads. In the case of the [single owner
pattern](#locking-pattern-single-owner) there might also be blocking semantics
involved, when the next owner needs to wait for the previous owner to finish
processing first. These are functions like <code>flush_work()</code> or the
various wait functions like <code>wait_event()</code> or
<code>wait_completion()</code>.

Calling these barrier functions while holding locks commonly leads to issues:

* Blocking functions like <code>flush_work()</code> pull in every lock or other
  dependency the work we wait on, or more generally, any of the previous owners
  of an object needed as a so called cross-release dependency. Unfortuantely
  lockdep does not understand these natively, and the usual tricks to add manual
  annotations have severe limitations. There's work ongoing to add
  [cross-release dependency tracking to
  lockdep](https://lwn.net/Articles/709849/), but nothing looks anywhere near
  ready to merge. Since these dependency chains can be really long and get ever
  longer when more code is added to a worker - dependencies are pulled in even
  if only a single lock is held at any given time - this can quickly become a
  nightmare to untangle.

* Often the requirement to hold a lock over these barrier type functions comes
  from the fact that the object would disappear. Or otherwise undergo some
  serious confusion about it's lifetime state - not just whether it's still
  alive or getting destroyed, but also who exactly owns it or whether it's maybe
  a resurrected zombie representing a different instance now. This encourages
  that the lock morphs from a "protects some specific data" to a "protects
  specific code from running" design, leading to all the code maintenance issues
  discussed in the [protect data, not code
  principle](/2022/07/locking-engineering.html#protect-data-not-code).

For these reasons try as hard as possible to not hold any locks, or as few as
feasible, when calling any of these memory barriers in disguise functions used
to manage object lifetime or ownership in general. The antipattern here is
abusing locks to fix lifetime issues. We have seen two specific instances thus
far:

* <code>kref_put_mutex</code> instead of <code>kref_get_unless_zero()</code> in
  the [weak reference pattern](#locking-pattern-weak-reference). This is a
  special case of the [reference counting
  pattern](#locking-pattern-reference-counting), but with some finer-grained
  locking added to support weak references.

* Calling <code>flush_work()</code> while holding locks in the [async
  worker](#locking-pattern-async-processing). This is a special case of the
  [single owner pattern](#locking-pattern-single-owner), again with a bit more
  locking added to support some mutable state.

We will see some more, but the antipattern holds in general as a source of
troubles.

<h2 style="background:orange;"> Level 2.5: Splitting Locks for Performance
Reasons</h2>

We've looked at a pile of functional reasons for complicating the locking
design, but sometimes you need to add more fine-grained locking for performance
reasons. This is already getting dangerous, because it's very tempting to tune
some microbenchmark just because we can, or maybe delude ourselves that it will
be needed in the future. Therefore only complicate your locking if:

* You have actual real world benchmarks with workloads relevant to users that
  show measurable gains outside of statistical noise.

* You've fully exhausted architectural changes to outright avoid the overhead,
  like io_uring pre-registering file descriptors locally to avoid manipulating
  the file descriptor table.

* You've fully exhausted algorithim improvements like batching up operations to
  amortize locking overhead better.

Only then make your future maintenance pain guaranteed worse by applying more
tricky locking than the bare minimum necessary for correctness. Still, go with the simplest approach, often converting a lock to its read-write
variant is good enough.

Sometimes this isn't enough, and you actually have to split up a lock into more
fine-grained locks to achieve more parallelism and less contention among
threads. Note that doing so blindly will backfire because locks are not free.
When common operations still have to take most of the locks anyway, even if it's
only for short time and in strict succession, the performance hit on single
threaded workloads will not justify any benefit in more threaded use-cases.

Another issue with more fine-grained locking is that often you cannot define a
strict nesting hierarchy, or worse might need to take multiple locks of the same
object or lock class. I've written previously about this specific issue, and
more importantly, [how to teach lockdep about lock nesting, the bad and the
good ways](/2020/08/lockdep-false-positives.html#fighting-lockdep-badly).

One really entertaining story from the GPU subsystem, for bystanders at least,
is that we really screwed this up for good by defacto allowing userspace to
control the lock order of all the objects involved in an IOCTL. Furthermore
disjoint operations should actually proceed without contention. If
you ever manage to repeat this feat you can take a look at the [wait-wound
mutexes](https://www.kernel.org/doc/html/latest/locking/ww-mutex-design.html).
Or if you just want some pretty graphs, [LWN has an old article about wait-wound
mutexes too](https://lwn.net/Articles/548909/).

<h2 style="background:red"> Level 3: Lockless Tricks</h2>

Do not go here wanderer!

Seriously, I have seen a lot of very fancy driver subsystem locking designs, I
have not yet found a lot that were actually justified. Because only real world,
non-contrived performance issues can ever justify reaching for this level, and
in almost all cases algorithmic or architectural fixes yield much better
improvements than any kind of (locking) micro-optimization could ever hope for.

Hence this is just a long list of antipatterns, so that people who have not yet
a grumpy expression permanently chiseled into their facial structure know when
they're in trouble.

Note that this section isn't limited to lockless tricks in the academic sense of
guaranteed constant overhead forward progress, meaning no spinning or retrying
anywhere at all. It's for everything which doesn't use standard locks like
<code>struct mutex</code>, <code>spinlock_t</code>, <code>struct
rw_semaphore</code>, or any of the others provided in the Linux kernel.

### Locking Antipattern: Using RCU

Yeah RCU is really awesome and impressive, but it comes at serious costs:

* By design, at least with standard usage, RCU elevates [mixing up lifetime and
  consistency
  concerns](#locking-antipattern-confusing-object-lifetime-and-data-consistency)
  to a virtue. <code>rcu_read_lock()</code> gives you both a read-side critical
  section *and* it extends the lifetime of any RCU protected object. There's
  absolutely no way you can avoid that antipattern, it's built in.

  Worse, RCU read-side critical section nest rather freely, which means unlike
  with real locks abusing them to keep objects alive won't run into nasty locking
  inversion issues when you pull that stunt with nesting different objects or
  classes of objects. Using locks to paper over lifetime issues is bad, but with
  RCU it's weapons-grade levels of dangerous.

* Equally nasty, RCU practically forces you to deal with zombie objects, which
  breaks the [reference counting pattern](#locking-pattern-reference-counting)
  in annoying ways.

* On top of all this breaking out of RCU is costly and kinda defeats the point,
  and hence there's a huge temptation to delay this as long as possible. Meaning
  check as many things and derefence as many pointers under RCU protection as
  you can, before you take a real lock or upgrade to a proper reference with
  <code>kref_get_unless_zero()</code>.

  Unless extreme restraint is applied this results in RCU leading you towards
  locking antipatterns. Worse RCU tends to spread them to ever more objects and
  ever more fields within them.
 
All together all freely using RCU achieves is proving that there really is no
bottom on the code maintainability scale. It is not a great day when your driver
dies in <code>synchronize_rcu()</code> and lockdep has no idea what's going on,
and I've seen such days.

Personally I think in driver subsytem the most that's still a legit and
justified use of RCU is for object lookup with <code>struct xarray</code> and
<code>kref_get_unless_zero()</code>, and cleanup handled entirely by
<code>kfree_rcu()</code>. Anything more and you're very likely chasing a rabbit
down it's hole and have not realized it yet.

### Locking Antipattern: Atomics

Firstly, Linux atomics have two annoying properties just to start:

* Unlike e.g. C++ atomics in userspace they are unordered or weakly ordered
  by default in a lot of cases. A lot of people are surprised by that, and then
  have an even harder time understanding the memory barriers they need to
  sprinkle over the code to make it work correctly.

* Worse, many atomic functions neither operate on the atomic types
  <code>atomic_t</code> and <code>atomic64_t</code> nor have <code>atomic</code>
  anywhere in their names, and so pose serious pitfalls to reviewers:
  - <code>READ_ONCE()</code> and <code>WRITE_ONCE</code> for volatile stores and
    loads.
  - <code>cmpxchg()</code> and the various variants of atomic exchange with or
    without a compare operation.
  - Atomic bitops like <code>set_bit()</code> are all atomic. Worse, their
    non-atomic variants have the <code>__set_bit()</code> double underscores to
    scare you away from using them, despite that these are the ones you really
    want by default.

Those are a lot of unnecessary trap doors, but the real bad part is what people
tend to build with atomic instructions:

* I've seen at least three different, incomplete and ill-defined
  reimplementations of read write semaphores without lockdep support. Reinventing
  completions is also pretty popular. Worse, the folks involved didn't realize
  what they built. That's an impressive violation of the ["Make it Correct"
  principle](/2022/07/locking-engineering.html#2-make-it-correct).

* It seems very tempting to build terrible variations of the ["no locking"
  patterns](#level-0-no-locking). It's very easy to screw them up by extending
  them in a bad way, e.g. reference counting with weak reference or RCU
  optimizations done wrong very quickly leads to a complete mess. There are 
  reasons why you should never deviate from these.

* What looks innocent are statistical counters with atomics, but almost always
  there's already a lock you could take instead of unordered counter updates.
  Often resulting in better code organization to boot since the statistics for a
  list and it's manipulation are then closer together. There are some exceptions
  with real performance justification, a recent one I've seen is memory
  shrinkers where you really want your <code>shrinker->count_objects()</code> to
  not have to acquire any locks.  Otherwise in a memory intense workload all
  threads are stuck on the one thread doing actual reclaim holding the same lock
  in your <code>shrinker->scan_objects()</code> function.

In short, unless you're actually building a new locking or synchronization
primitive in the core kernel, you most likely do not want to get seen even
looking at atomic operations as an option.

### Locking Antipattern: <code>preempt/local_irq/bh_disable()</code> and Friends ...

This one is simple: Lockdep doesn't understand them. The real-time folks hate
them. Whatever it is you're doing, use proper primitives instead, and at least
read up on the [LWN coverage on why these are
problematic what to do instead](https://lwn.net/Articles/828477/). If you need
some kind of synchronization primitive - maybe to avoid the [lifetime vs.
consistency antipattern
pitfalls](#locking-antipattern-confusing-object-lifetime-and-data-consistency) -
then use the proper functions for that like <code>synchronize_irq()</code>.

### Locking Antipattern: Memory Barriers

Or more often, lack of them, incorrect or imbalanced use of barriers, badly or
wrongly or just not at all documented memory barriers, or ...

Fact is that exceedingly most kernel hackers, and more so driver people, have no
useful understanding of the Linux kernel's memory model, and should never be
caught entertaining use of explicit memory barriers in production code.
Personally I'm pretty good at spotting holes, but I've had to learn the hard way
that I'm not even close to being able to positively prove correctness. And for
better or worse, nothing short of that tends to cut it.

For a still fairly cursory discussion read the [LWN series on lockless
algorithms](https://lwn.net/Articles/844224/). If the code comments and commit
message are anything less rigorous than that it's fairly safe to assume there's
an issue.

Now don't get me wrong, I love to read an article or watch a talk by Paul
McKenney on RCU like anyone else to get my brain fried properly. But aside from
extreme exceptions this kind of maintenance cost has simply no justification in
a driver subsystem. At least unless it's packaged in a driver hacker proof
library or core kernel service of some sorts with all the memory barriers well
hidden away where ordinary fools like me can't touch them.

## Closing Thoughts

I hope you enjoyed this little tour of propressively more worrying levels of
locking engineering, with really just one key take away:

Simple, dumb locking is good locking, since with that you have a fighting chance
to make it correct locking.

Thanks to Daniel Stone and Jason Ekstrand for reading and commenting on drafts of this text.
