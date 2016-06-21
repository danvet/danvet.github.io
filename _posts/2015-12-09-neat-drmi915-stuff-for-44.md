---
layout: post
title: Neat drm/i915 stuff for 4.4
date: '2015-12-09T00:55:00.003-08:00'
author: danvet
tags: 
modified_time: '2015-12-09T02:52:48.769-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-64982716988550007
blogger_orig_url: http://blog.ffwll.ch/2015/12/neat-drmi915-stuff-for-44.html
---

Due to vacations, conferences and other things I'm way later than usual and [4.3
has been released](/2015/09/neat-drmi915-stuff-for-43.html)
a while ago. More than overdue to take a look at what's in store in the next
kernel release.

<!--more-->

First looking at overall infrastructure work on the display side there's a lot of <b>atomic conversion</b> progress again. One feature that's now on solid fundations is <b>fastboot, built on top of atomic infrastructure</b> with patches from Maarten. Unfortunately we had to disable it again due to some backligh issues early in 4.4-rc. The other big piece is reworking the watermark update code (Ville&amp;Matt), which unfortunately ran into regression roadblocks already in the development cycle and had to be reverted partially. Another piece of infrastructure building on top of atomic is <b>validation&amp;adjusting the display clock</b> - some ULT chips can't drive all DP screens and the driver now detects that, and it should also downclock when less bandwidth is needed. This was implemented by Mika Kahola and Ville.



Again this round has seen a lot of improvements and <b>bug fixes to PSR code</b> (from Rodrigo) and for <b>FBC</b> (from Paulo). Unfortunately we're not yet done with those, but it looks really good that at least PSR can finally be enabled for 4.5. Still on the display side of the driver there was a pile of smaller improvements all over: Prep work for Broxton DSI support (Shashank Sharma). HDMI detection finally checks the hotplug sense, after some workaround from Sonika. And tons of cleanups all over. Fixing up DMC support (for new low-power display states) was also a topic, but we've only managed to fix it up for real in 4.5.



On the GEM side the big thing for sure is support for the extended <b>48-bit GPU address space</b> on Broadwell and later chips, from Michel Thierry. And then there's the code for <b>GuC-based command submission</b> (Alex Dai and Dave Gordon), which is merged but not yet enabled by default. The idea behind that is to feed all command submission through an on-chip microcontroller, which can then react much faster to changing workloads and tune power states accordingly. It should also help long-term with better scheduling by supporting preemption. But none of that is implemented yet, so this is just fundations.



For existing features there are <b>bugfixes for userptr and shrinker improvements</b> from Chris Wilson. And Tvrtko has extended the vma view code in prepartion of rotation support for NV12.



Of course there's also been the usual enabling work for new platforms, this time around mostly consisting of workaround patches for Skylake and Broxton. But Zhiyuan Lv submitted support for the <b>virtualized XenGT gpu support on Broadwell</b>.



Finally for driver internals there's the massive work from Ville to <b>make the register access functions type safe</b>. This is escpecially a problem for writing registers, where both the register and the value that needs to be written are of type <code>uin32_t</code>. That resulted in subtile bugs fairly often. Ville encapsulated the register offset into a struct and converted all the thousands of register #defines and users over to that, and now compilation will fail if we ever get this wrong again.
