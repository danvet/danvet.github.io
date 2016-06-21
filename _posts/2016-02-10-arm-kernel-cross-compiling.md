---
layout: post
title: ARM kernel cross compiling
date: '2016-02-10T08:01:00.000-08:00'
author: danvet
tags:
- Maintainer-Stuff
modified_time: '2016-02-10T08:01:34.700-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-1109414631376380336
blogger_orig_url: http://blog.ffwll.ch/2016/02/arm-kernel-cross-compiling.html
---

I do a lot of cross driver subsytem refactorings, and DRM has lots of drivers that only run on ARM. Which means I routinely break a leg or arm since at least in the past cross-compiling was somehow always super painful. But I've just learned (thanks to Daniel Stone) that cross-compiling this stuff has become real easy, so here's my handy script for this. This assumes Debian, but the difference is just in installing a different cross-compiler toolchain.<br /><br />First get the tooling:<br /><br /><code>$ sudo apt-get install gcc-arm-linux-gnueabihf</code><br /><code>&nbsp;</code><br />Then create another git checkout. I prefer the recently merged worktree support, since with that all your branches and remotes transparently work in the new checkout, too.<br /><br /><code>~/kernel/src/ $ git worktree add ../armhf HEAD</code><br /><code>&nbsp;</code><br />With that we're all set up. For building any <code>$branch</code> wrap the following lines into a scrip:<br /><br /><code>$ cd ~/kernel/armhf<br />$ git checkout --detach $branch<br />$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- multi_v7_defconfig zImage modules</code><br /><br />I'm using --detach to avoid complaints from the git worktree code that I've checked out a branch already in the main repo. Note: Never accidentally run plain make in the ARM build directory - mixing up architectures seriously confuses the kernel build system.