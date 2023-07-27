---
layout: post
title: Midlayers, Once More With Feelings!
date: '2016-12-14T10:00'
author: sima
tags: 
- Maintainer-Stuff
---
The collective internet troll fest had it's fun recently discussing AMD's DAL.
[Hacker news discussed the
rejection](https://news.ycombinator.com/item?id=13136426) and [some of the
reactions](https://news.ycombinator.com/item?id=13142285), [reddit had some
fun](https://www.reddit.com/r/linux/comments/5hhnur/amd_responds_to_linux_kernel_maintainers/)
and of course everyone on [phoronix forums was going totally
nuts](https://www.phoronix.com/forums/forum/phoronix/general-discussion/916781-it-looks-like-amdgpu-dc-dal-will-not-be-accepted-in-the-linux-kernel).
Luckily reason seems to finally prevail with [LWN covering things
too](https://lwn.net/Articles/708891/). I don't want to spill more bits over the
topic itself (read the LWN coverage and mailing list threads for that), but I
think it's worth looking at the fundamental underlying problem a bit more.

<!--more-->

Discussing midlayers seems to be one of the recuring topics in the linux kernel.
There's the [original midlayer-mistake article from Neil
Brown](https://lwn.net/Articles/336262/) that seems to have started it all. But
LWN gained more articles in the years since, covering [the iscsi driver as a
study in avoiding OS abstraction layers](https://lwn.net/Articles/454716/), or a
similar [case in wireless with the Broadcom
driver](https://lwn.net/Articles/456762/). The dismissal of midlayers and
hailing of helper libraries has become so prevalent that calling your shiny new
subsystem [libfoo (viz. libnvdimm)](https://lwn.net/Articles/654071/) seems to
be a powerful trick to speed up its acceptance.

It seems common knowledge and accepted fact, but still there's a constant stream
of drivers that come packaged with huge abstraction layers - mostly to abstract OS
internals, but very often also just a few midlayers between different components
within the driver. A major reason for this is certainly that submissions by former
proprietary teams, or just generally code developed internally behind closed
doors is suffering from the [platform problem - again LWN has you
covered](https://lwn.net/Articles/443531/). If your driver is not open and part
of upstream (or even for an open source OS), then the provided services and
interfaces are fixed. And there's no way to improve things, worse, when change
does happen the driver team generally doesn't have any influence at all over
what and how things changes. Hence it makes sense to insert a big abstraction
layer to isolate the driver from the outside madness.

But that does not explain why big drivers (and more so, subsystems) come with
some nice abstraction layers wedged in-between different parts. I believe, with
not any proof really, that this is because company planners are extremely risk
averse: Sure, working together and sharing code across the entire project has
long-term benefits. But the more people are involved the bigger the risk for a
bikeshed fest or some other delay, and you never want to be the one team that
delayed a release by a few months. Adding isolation in the form of lots of fixed
abstraction layers helps with that. But long term, and for really big projects,
sharing code and working together has clear benefits, even if there's routinely
a hiccup - the neck-breaking speed of Linux kernel development overall is
testament enough for that I think.

All that is just technicalities really, because in the end upstream and open
source is about collaboratively developing software. That requires shared
control, and interfaces between different components need to be a lot more
permeable. In my experience that core idea of handing control over
development to outsiders is the really scary part of joining upstream, since
followed through to its conclusions it means you need to completely rethink how
products are planned and developed: The entire organisation, from individual
developers, to teams and including the management chain have to change. And that
freaks people out.

In summary I think code submissions with lots of midlayers are bound to stay
with us, and that's good: Because new midlayers means new teams and companies
start to take upstream seriously. And new midlayers getting cleaned up means new
teams and new companies are going to the painful changes necessary to adjust to an
upstream first model. The code itself is just the canary for the real shifts
happening.

In other news: World domination is still progressing according to plan.
