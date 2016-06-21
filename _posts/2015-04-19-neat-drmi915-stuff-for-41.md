---
layout: post
title: Neat drm/i915 stuff for 4.1
date: '2015-04-19T20:57:00.001-07:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2015-04-19T20:57:44.735-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-1727790391245288063
blogger_orig_url: http://blog.ffwll.ch/2015/04/neat-drmi915-stuff-for-41.html
---

With [Linux kernel v3.20^W v4.0](http://blog.ffwll.ch/2015/02/neat-drmi915-stuff-for-320.html) already out there door my overview of what's in 4.1 for drm/i915 is way overdue.

<!--more-->

First looking at the modeset side of the driver the big overall thing is all the work to convert i915 to atomic. In this release there's code from Ander Conselvan de Oliveira to have a<b> struct drm_atomic_state allocated for all the legacy modeset code paths</b> in the driver. With that we can switch the internals to start using atomic state objects and gradually convert over everything to atomic on the modeset side. Matt Roper on the other hand was busy to prepare the plane code to land the atomic watermark update code. Damien has <b>reworked the initial plane configuration code</b> used for fastboot, which also needs to be adapted to the atomic world.





For more specific feature work there's the <b>DRRS</b> (dynamic refresh rate switching) from Sonika, Vandana and more people, which is now enabled where supported. The idea is to reduce the refresh rate of the panel to save power when nothing changes on the screen. And Paulo Zanoni has provided patches to <b>improve the FBC code</b>, hopefully we can enable that by default soon too. Under the hood Ville has <b>refactored the DP link rate computation and the sprite color key handling</b>, both to prepare for future work and platform enabling. <b>Intermediate link rate support for eDP</b> 1.4 from Sonika built on top of this. Imre Deak has also reworked the Baytrail/Braswell <b>DPLL code to prepare for Broxton</b>.



Speaking of platforms, Skyleigh has gained<b> runtime PM</b> support from Damien, and <b>RPS</b> (render turbo and sleep states) from Akash. Another SKL exclusive is support for <b>scanout of Y-tiled buffers</b> and scanning out buffers rotated by 90°/270° (instead of just normal and rotated by 180°) from Tvrtko and Damien. Well the rotation support didn't quite land yet, but Tvrtko's support for the special pagetable binding needed for that feature in the form of <b>rotated GGTT views</b>. Finally Nick Hoath and Damien also submitted a lot of workaround patches for SKL.



Moving on to <b>Braswell/Cherryview </b>there have been tons of fixes to the DPLL and watermark code from Vijay and Ville, and <b>BSW left the preliminary hw support</b>. And also for the SoC platforms Chris Wilson has supplied a pile of patches to <b>tune the rps code</b> and bring it more in-line with the big core platforms.



On the GT side the big ongoing work is <b>dyanmic pagetable allocations</b> Michel Thierry based upon patches from Ben Widawsky. With per-process address spaces and even more so with the big address spaces gen8+ supports it would be wasteful if not impossible to allocate pagetables for the entire address space upfront. But changing the code to handle another possible memory allocation failure point needed a lot of work. Most of that has landed now, but the benefits of enabling bigger address spaces haven't made it into 4.1.



Another big work is <b>XenGT client-side support</b> fromYu Zhang and team. This is paravirtualization to allow virtual machines to tap into the render engines without requiring exclusive access, but also with a lot less overhead than full virtual hardware like vmware or virgil would provide. The host-side code is also submitted already, but needs a bit more work still to integrate cleanly into the driver.



And of course there's been lots of other smaller work all over, as usual. Internal documentation for the shrinker, more dead UMS code removed, the vblank interrupt code cleaned up and more.