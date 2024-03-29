---
layout: post
title: Atomic Modesetting Design Overview
date: '2015-08-16T06:52:00.003-07:00'
author: sima
tags:
- In-Depth Tech
modified_time: '2015-08-16T06:52:37.045-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-2572132837736503140
blogger_orig_url: http://blog.ffwll.ch/2015/08/atomic-modesetting-design-overview.html
---

After a few years of development the atomic display update IOCTL for drm drivers
is finally ready for prime time with the [4.2 pull request from Dave
Airlie](http://mid.mail-archive.com/alpine.DEB.2.00.1506260158440.13786@skynet.skynet.ie).
It's been a long road, with a lot of drivers [already converted over to
atomic](/2014/11/atomic-modeset-support-for-kms-drivers.html) and even more in
progress, the [atomic helper libraries and support code in the drm
subsystem](/2015/01/update-for-atomic-display-updates.html) sufficiently
polished. But what's really missing is a design overview of what the overall
atomic infrastructure looks like and why some decisions and details are
implemented like they are.

That's now done and published on LWN: [Part 1 talks about the problem space,
issues with the Android atomic display framework and the basic atomic IOCTL
interface.](https://lwn.net/Articles/653071/) [Part 2 goes into more detail
about a few specific things like locking, helper library design and the exact
semantics of atomic modessetting updates.](https://lwn.net/Articles/653466/)
Happy Reading! 
