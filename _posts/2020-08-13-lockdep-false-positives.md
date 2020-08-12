---
layout: post
title: "Lockdep False Positives, some stories about"
tags:
- In-Depth Tech
---
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Lockdep is giving false positives are the new the compiler is broken.</p>&mdash; David Airlie (@DaveAirlie) <a href="https://twitter.com/DaveAirlie/status/1291932064606859269?ref_src=twsrc%5Etfw">August 8, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

Recently we've looked a bit at lockdep annotations in the GPU subsystems, and I
figured it's a good opportunity to explain how this all works, and what the
tradeoffs are. Creating working locking hierarchies for the kernel isn't easy,
making sure the kernel's locking validator *lockdep* is happy and reviewers
don't have their brains explode even more so.

First things first, and the fundamental issue:

>   Lockdep is about trading false positives against better testing.

The only way to avoid false positives for deadlocks is to only report a deadlock
when the kernel actually deadlocked. Which is useless, since the entire point of
lockdep is to catch potential deadlock issues before they actually happen. Hence
false postives are not avoidable, at least not in theory, to be able to report
potential issues before they hang the machine. Read on for what to do in
practice.

<!--more-->

We need to understand how exactly lockdep trades false positives to better
discovery locking inconsistencies. Lockdep makes a few assumptions about how
real code does locking in practice:

## Invariance of locking rules over time

First assumption baked into lockdep is that the locking rules for a given lock
do not change over the lifetime of the lock's existence. This already throws out
a large chunk of perfectly correct locking designs, since state transitions can
control how an object is accessed, and therefore how the lock is used.  Examples
include different rules for creation and destruction, or whether an object is on
a specific list (e.g. only a gpu buffer object that's in the lru can be
evicted). It's not possible to proof automatically that certain code flat
out wont ever run together with some other code on the same structure, at least
not in generality. Hence this is pretty much a required assumption to make
lockdep useful - if every new <code>lock()</code> call could follow new rules
there's nothing to check. Besides realizing that an actual deadlock indeed
occured and all is lost already.

And of course getting such state transitions correct, with the guarantee that
all the old code will no longer run, is tricky to get right, and very hard on
reviewers. It's a good thing lockdep has problems with such code too.

## Common locking rules for the same objects

Second assumption is that all locks initialized by the same code are following
the same locking rules. This is achieved by making all lock initializers C
macros, which create the corresponding lockdep class as a static variable within
the calling function. Again this is pretty much required, since to spot
inconsistencies you need as many observations of all the different code path
possibilities. Best to share them all between the same object. Also a distinct
lockdep class for each individual object would explode the runtime overhead in
both memory and cpu cycles.

And again this is good from a code design point too, since having the same data
structure and code follow different locking rules for different objects is at
best very confusing for reviewers.

## Fighting lockdep, badly

Now things go wrong, you have a lockdep splat at your hands, concluded it's a
false positive and go ahead trying to teach lockdep about what's going on. The
first class of annotains are special <code>lock_nested(lock, subclass)</code>
functions. Without lockdep nothing in the generated code changes, but it tells
lockdep that for this lock acquisition, we're using a different class to track
the observed locking.

This breaks both the time invariance - nothing is stopping you from using
different classes for the same lock at different times - and commonality of
locking for the same objects. Worse, you can write code which obviously
deadlocks, but lockdep will think everything is perfectly fine:

    mutex_init(&A);

    mutex_lock(&A);
    mutex_lock_nested(&A, SINGLE_DEPTH_NESTING);

This is no good and puts a huge burden on reviewers to carefully check all these
places themselves, manually. Exactly the kind of tedious and error prone work
lockdep was meant to take over.

Slightly better are the annotations which adjust the lockdep class once, when
the object is initialized, using <code>lockdep_set_class()</code> and related
functions. This at least does not break time invariance, hence will at least
guarantee that lockdep spots the deadlock latest when it happens. It still
reduces how much lockdep can connect what's going on, but occasionally "rewrite
the entire subsystem" to resolve a locking inconsistency is just not a
reasonable option.

It still means that reviewers always need to remember what the locking rules for
all types of different objects behind the same structure are, instead of just
one. And then check against every path whether that code needs to work with all
of them, or just some, or only one. Again tedious work that really lockdep is
supposed to help with. If it's hard to come by a system where you can
easily run the code for the different types of object without rebooting,
then lockdep cannot help at all.

All these annotations have in common that they don't change the code logic, only
how lockdep interprets what's going on.

An even more insideous trick on reviewers and lockdep is to push locking into an
asynchronous worker of some sorts. This hides issues because lockdep does not
follow dependencies between threads through waiter/wakee relationships like
<code>wait_for_completion()</code> and <code>complete()</code>, or through wait
queues. There are lockdep annotations for specific dependencies, like in
the kernel's workqueue code when flushing workers or specific work items with
<code>flush_work()</code>. Automatic annotations have been attemped with the
[lockdep cross-release extension](https://lwn.net/Articles/709849/), which for
various reasons had to be backed out again. Therefore hand-rolled asynchronous
code is a great place to create complexity and hide locking issues from both
lockdep and reviewers.

## Playing to lockdep's strength

Except when there's very strong justification for all the complexity, the real
fix is to change the locking and make it simpler. Simple enough for lockdep to
understand what's going on, which also makes reviewer's lifes a lot better.
Often this means substantial code rework, but at least in some cases there
are useful tricks.

A special kind of annotations are the <code>lock_nest_lock(lock,
superlock)</code> family of functions - these tell lockdep that when multiple
locks of the same class are acquired, it's all serialized by the single
superlock. Lockdep then validates that the right superlock is indeed held. A
great example is <code>mm_take_all_locks()</code>, which as the name implies,
takes all locks related to the given <code>mm_struct</code>. In a sense this is
not a pure annotation, unlike the ones above, since it requires that the
superlock is actually locked. That's generally the easier to understand scheme
than clever sorting of lock acquisition of some sort for reviewers too, not just
for lockdep.

A different situation often arises when creating or destroying an object. But at
that stage often no other thread has a reference to the object and therefore can
take the lock, and the best way to resolve locking inconsistency over the
lifetime of an object due to creation and destruction code is to not take any
locks at all in these paths. There is nothing to protect against after all!

In all these cases the best option for long term maintainability is to simplify
the locking design, not reduce lockdep's power by reducing the amount of false
positives it reports. And that should be the general principle.

> tldr; do not fix lockdep false positives, fix your locking
