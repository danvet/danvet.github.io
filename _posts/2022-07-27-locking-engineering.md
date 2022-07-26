---
layout: post
title: Locking Engineering Principles
author: danvet
tags:
- In-Depth Tech
---
For various reasons I spent the last two years way too much looking at code with
terrible locking design and trying to rectify it, instead of a lot more actual
building cool things. Symptomatic that the last post here on my neglected blog
is also a [rant on lockdep abuse](/2020/08/lockdep-false-positives.html).

I tried to distill all the lessons learned into some training slides, and this
two part is the writeup of the same. There are some GPU specific rules, but I
think the key points should apply to at least apply to kernel drivers in
general.

The first part here lays out some principles, the second part builds a locking
engineering design pattern hierarchy from the most easiest to understand and
maintain to the most nightmare inducing approaches.

Also with locking engineering I mean the general problem of protecting data
structures against concurrent access by multiple threads and trying to ensure
that each sufficiently consistent view of the data it reads and that the updates
it commits won't result in confusion. Of course it highly depends upon the
precise requirements what exactly sufficiently consistent means, but figuring
out these kind of questions is out of scope for this little series here.

<!--more-->
## Priorities in Locking Engineering

Designing a correct locking scheme is hard, validating that your code actually
implements your design is harder, and then debugging when - not if! - you
screwed up is even worse. Therefore the absolute most important rule in locking
engineering, at least if you want to have any chance at winning this game, is to
make the design as simple and dumb as possible.

### 1. Make it Dumb

Since this is _the_ key principle the entire second part of this series will go
through a lot of different locking design patterns, from the simplest and
dumbest and easiest to understand, to the most hair-raising horrors of
complexity and trickiness.

Meanwhile let's continue to look at everything else that matters.

### 2. Make it Correct

Since simple doesn't necessarily mean correct, especially when transferring a
concept from design to code, we need guidelines. On the design front the most
important one is to [design for lockdep, and not fight
it](/2020/08/lockdep-false-positives.html), for which I already wrote a full length
rant. Here I will only go through the main lessons: Validating locking by hand
against all the other locking designs and nesting rules the kernel has overall
is nigh impossible, extremely slow, something only few people can do with any
chance of success and hence in almost all cases a complete waste of time. We
need tools to automate this, and in the Linux kernel this is lockdep.

Therefore if lockdep doesn't understand your locking design your design is at
fault, not lockdep. Adjust accordingly.

A corrollary is that you actually need to teach lockdep your locking rules,
because otherwise different drivers or subsystems will end up with defacto
incompatible nesting and dependencies. Which, as long as you never exercise them
on the same kernel boot-up, much less same machine, wont make lockdep grumpy.
But it will make maintainers very much question why they are doing what they're
doing.

Hence at driver/subsystem/whatever load time, when CONFIG_LOCKDEP is enabled,
take all key locks in the correct order. One example for this relevant
to GPU drivers is [in the dma-buf
subsystem](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/dma-buf/dma-resv.c?h=v5.18#n685).

In the same spirit, at every entry point to your library or subsytem, or
anything else big, validate that the callers hold up the locking contract with
<code>might_lock(), might_sleep(), might_alloc()</code> and all the variants and
more specific implementations of this. Note that there's a huge overlap between
locking contracts and calling context in general (like interrupt safety, or
whether memory allocation is allowed to call into direct reclaim), and since all
these functions compile away to nothing when debugging is disabled there's
really no cost in sprinkling them around very liberally.

On the implementation and coding side there's a few rules of thumb to follow:

* Never invent your own locking primitives, you'll get them wrong, or at least
  build something that's slow. The kernel's locks are built and tuned by people
  who've done nothing else their entire career, you wont beat them except in bug
  count, and that by a lot.

* The same holds for synchronization primitives - don't build your own with a
  <code>struct wait_queue_head</code>, or worse, hand-roll your own wait queue.
  Instead use the most specific existing function that provides the
  synchronization you need, e.g. <code>flush_work()</code> or
  <code>flush_workqueue()</code> and the enormous pile of variants available for
  synchronizing against scheduled work items.

  A key reason here is that very often these more specific functions already
  come with elaborate lockdep annotations, whereas anything hand-roll tends to
  require much more manual design validation.

* Finally at the intersection of "make it dumb" and "make it correct", pick the
  simplest lock that works, like a normal mutex instead of an read-write
  semaphore. This is because in general, stricter rules catch bugs and design
  issues quicker, hence picking a very fancy "anything goes" locking primitives
  is a bad choice.

  As another example pick spinlocks over mutexes because spinlocks are a lot
  more strict in what code they allow in their critical section. Hence much less
  risk you put something silly in there by accident and close a dependency loop
  that could lead to a deadlock.

### 3. Make it Fast

Speed doesn't matter if you don't understand the design anymore in the future,
you need simplicity first.

Speed doesn't matter if all you're doing is crashing faster. You need
correctness before speed.

Finally speed doesn't matter where users don't notice it. If you
micro-optimize a path that doesn't even show up in real world workloads users
care about, all you've done is wasted time and committed to future maintenance
pain for no gain at all.

Similarly optimizing code paths which should never be run when you instead
improve your design are not worth it. This holds especially for GPU drivers,
where the real application interfaces are OpenGL, Vulkan or similar, and there's
an entire driver in the userspace side - the right fix for performance issues
is very often to radically update the contract and sharing of responsibilities
between the userspace and kernel driver parts.

The big example here is GPU address patch list processing at command submission
time, which was necessary for old hardware that completely lacked any useful
concept of a per process virtual address space. But that has changed, which
means virtual addresses can stay constant, while the kernel can still freely
manage the physical memory by manipulating pagetables, like on the CPU.
Unfortunately one driver in the DRM subsystem instead spent an easy engineer
decade of effort to tune relocations, write lots of testcases for the resulting
corner cases in the multi-level fastpath fallbacks, and even more time handling
the impressive amounts of fallout in the form of bugs and future headaches due
to the resulting unmaintainable code complexity ...

In other subsystems where the kernel ABI is the actual application contract
these kind of design simplifications might instead need to be handled between
the subsystem's code and driver implementations. This is what we've done when
moving from the old kernel modesetting infrastructure to atomic modesetting.
But sometimes no clever tricks at all help and you only get true speed with a
radically revamped uAPI - io_uring is a great example here.


## Protect Data, not Code

A common pitfall is to design locking by looking at the code, perhaps just
sprinkling locking calls over it until it feels like it's good enough. The right
approach is to design locking for the data structures, which means specifying
for each structure or member field how it is protected against concurrent
changes, and how the necessary amount of consistency is maintained across the
entire data structure with rules that stay invariant, irrespective of how code
operates on the data. Then roll it out consistently to all the functions,
because the code-first approach tends to have a lot of issues:

* A code centric approach to locking often leads to locking rules changing over
  the lifetime of an object, e.g. with different rules for a structure or member
  field depending upon whether an object is in active use, maybe just cached or
  undergoing reclaim. This is hard to teach to lockdep, especially when also the
  nesting rules change for different states, because lockdep assumes that the
  locking rules are completely invariant over the lifetime of the entire kernel,
  not just over the lifetime of an individual object or structure even.

  Starting from the data structures on the other hand encourages that locking
  rules stay the same for a structure or member field.

* Data structure driven locking design means there's a perfect place to document
  the rules - in the kerneldoc of each structure or member field. Locking design
  that changes depending upon the code that can touch the data would need either need
  complicated documentation entirely separate from the code - so high risk of
  becoming stale. Or it's sprinkled over the various functions, which means
  reviewers need to reacquire the entire relevant chunks of the code base again
  to make sure they don't miss an odd corner cases.

  This would mean that to recheck the locking design for a code first approach
  every function and flow has to be checked against all others, and changes need
  to be checked against all the existing code. If this is not done you might
  miss a corner cases where the locking falls apart with a race condition or
  could deadlock. With a data first approach to locking though changes can be
  reviewed incrementally against the invariant rules, which means review of
  especially big or complex subsystems actually scales.

* When facing a locking bug it's tempting to try and fix it just in the affected
  code. By repeating that often enough a locking scheme that protects data
  acquires code specific special cases. Therefore locking issues always
  need to be first mapped back to new or changed requirements on the data
  structures and how they are protected.

_The_ big antipattern of how you end up with code centric locking is to protect
an entire subsystem (or worse, a group of related subsystems) with a single
huge lock. The canonical example was the big kernel lock *BKL*, that's gone, but
in many cases it's just replaced by smaller, but still huge locks like
<code>console_lock()</code>.

This results in a lot of long term problems when trying to adjust the locking
design later on:

* Since the big lock protects everything, it's often very hard to tell what it
  does not protect. Locking at the fringes tends to be inconsistent, and due to
  that its coverage tends to creep ever further when people try to fix bugs
  where a given structure is not consistently protected by the same lock.

* Also often subsystems have different entry points, e.g. consoles can be
  reached through the console subsystem directly, through vt, tty subsystems and
  also through an enourmous pile of driver specific interfaces with the fbcon
  IOCTLs as an example. Attempting to split the big lock into smaller
  per-structure locks pretty much guarantees that different entry points have to
  take the per-object locks in opposite order, which often can only be resolved
  through a large-scale rewrite of all impacted subsystems.

  Worse, as long as the big subsystem lock continues to be in us no one is
  spotting these design issues in the code flow. Hence they will slowly get
  worse instead of the code moving towards a better structure.

For these reasons big subsystem locks tend to live way past their justified
usefulness until code maintenance becomes nigh impossible: Because no individual
bugfix is worth the task to really rectify the design, but each bugfix tends to
make the situation worse.

## From Principles to Practice

Stay tuned for next week's installment, which will cover what these principles
mean when applying to practice: Going through a large pile of locking design
patterns from the most desireable to the most hair raising complex.
