---
layout: post
title: Atomic Modesetting Design Overview
date: '2015-08-16T06:52:00.003-07:00'
author: danvet
tags:
- In-Depth Tech
modified_time: '2015-08-16T06:52:37.045-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-2572132837736503140
blogger_orig_url: http://blog.ffwll.ch/2015/08/atomic-modesetting-design-overview.html
---

After a few years of development the atomic display update IOCTL for drm drivers is finally ready for prime time with the <a href="http://mid.mail-archive.com/alpine.DEB.2.00.1506260158440.13786@skynet.skynet.ie">4.2 pull request from Dave Airlie</a>. It's been a long road, with a lot of drivers <a href="http://blog.ffwll.ch/2014/11/atomic-modeset-support-for-kms-drivers.html">already converted over to atomic</a> and even more in progress, the <a href="http://blog.ffwll.ch/2015/01/update-for-atomic-display-updates.html">atomic helper libraries and support code in the drm subsystem</a> sufficiently polished. But what's really missing is a design overview of what the overall atomic infrastructure looks like and why some decisions and details are implemented like they are.



That's now done and published on LWN: <a href="https://lwn.net/Articles/653071/">Part 1 talks about the problem space, issues with the Android atomic display framework and the basic atomic IOCTL interface.</a> <a href="https://lwn.net/Articles/653466/">Part 2 goes into more detail about a few specific things like locking, helper library design and the exact semantics of atomic modessetting updates.</a> Happy Reading! 