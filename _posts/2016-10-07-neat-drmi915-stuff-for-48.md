---
layout: post
title: Neat drm/i915 Stuff for 4.8
date: '2016-10-07T10:00-0200'
author: danvet
tags: 
- Kernel Relnotes
---
I procristanated rather badly on this one, so instead of [the previous kernel
release happening](/2016/05/neat-drmi915-stuff-for-47.html) the v4.8 release is already
out of the door. Read on for my slightly more terse catch-up report.

<!--more-->
Since I'm this late I figured instead of the usual comprehensive list I'll do
something new and just list some of the work that landed in 4.8, but with a bit
more focus on the impact and why things have been done.

## Midlayers, Be Gone!

The first thing I want
to highlight is the driver de-midlayering. In the linux kernel community the
[mid-layer mistake or helper library design pattern, see the linked article from
LWN](https://lwn.net/Articles/336262/) is a set of rules to design subsystems
and common support code for drivers. The underlying rule is that the driver
itself must be in control of everything, like allocating memory, handling all
requests. Common code is only shared in helper library functions, which the
driver can call if they are suitable. The reason for that is that there is
always some hardware which needs special treatment, and when you have a special
case and there's a midlayer, it will get in the way.

Due to the shared history with BSD kernels DRM originally had a full-blown
midlayer, but over time this has been fixed. For example kernel modesetting was
designed from the start with the helper library pattern. The last hold is the
device structure itself, and for the Intel driver this is now fixed. This has
two main benefits:

 - First we can get rid of a lot of pointer dereferencing in the compiled
   binaries. With the midlayer DRM allocated a <code>struct drm_device</code>,
   and the Intel driver allocated it's own, separate structure. Both are
   connected with pointers, and every time control transferred from driver
   private functions to shared code those pointers had to be walked.

   With a helper approach the driver allocates the DRM device structure embedded
   into it's own device strucure. That way the pointer derefencing just becomes
   a fixed adjustement offset of the original pointer. And fixed offsets can be
   baked into each access of individual member fields for free, resulting in a
   big reduction of compiled code.

 - The other benefit is that the Intel driver is now in full control of the
   driver load and unload sequence. The DRM midlayer functions for loading had a
   few driver callbacks, but for historical reasons at the wrong spots. And
   fixing that is impossible without rewriting the load code for all the
   drivers. Without the midlayer we can have as many steps in the load
   sequence as we want, and where we want it. The most important fix here is
   that the driver will now be initialiazed completely before any part of it is
   registered and visible to userspace (through the /dev node, sysfs or anywhere
   else).

## Thundering Herds

GPUs process rendering asynchronously, and sometimes the CPU needs to wait for
them. For this purpose there's a wait queue in the driver. Userspace processes
block on that until the interrupt handler wakes them up. The trouble now is that
thus far there was just one wait queue per engine, which means every time the GPU
completed something all waiters had to be woken up. Then they checked whether
the work they needed to wait for completed, and if not, again block on the
wait queue until the next batch job completed. That's all rather inefficient. On
top there's only one per-engine knob to enable interrupts. Which means even if
there was only one waiting process, it was woken for every completed job. And
GPUs can have a lot of jobs in-flight.

In summary, waiting for the GPU worked more like a frantic herd trampling all
over things instead of something orderly. To fix this the request and completion
tracking was entirely revamped, to make sure that the driver has a much better
understanding of what's going on. On top there's now also an efficient search
structure of all current waiting processes. With that the interrupt handler can
quickly check whether the just completed GPU job is of interest, and if so,
which exact process should be woken up.

But this wasn't just done to make the driver more efficient. Better tracking of
pending and completed GPU requests is an important fundation to implement proper
GPU scheduling on top of. And it's also needed to interface the completion
tracking with other drivers, to finally fixing tearing for multi-GPU machines.
Having a thundering herd in your own backyard is unsightly, letting it loose on
your neighbours is downright bad! A lot of this follow-up work already landed
for the 4.9 kernel, hence I will talk more about this in a future installement
of this seris.
