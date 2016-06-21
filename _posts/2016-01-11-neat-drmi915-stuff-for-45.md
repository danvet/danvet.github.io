---
layout: post
title: Neat drm/i915 stuff for 4.5
date: '2016-01-11T08:12:00.000-08:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2016-01-11T08:12:27.039-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-1984383501718899742
blogger_orig_url: http://blog.ffwll.ch/2016/01/neat-drmi915-stuff-for-45.html
---

[Kernel version 4.4](http://blog.ffwll.ch/2015/12/neat-drmi915-stuff-for-44.html) is released, it's time for our regular look at what's in store for the Intel graphics driver in the next release.

<!--more-->

Overall this cycle has seen lots and lots of bugfixes, and the reason for that is that we're rebuilding our CI infrastructure after it went up in a poof of smoke last summer. Really big thanks to the entire team for the effort invested! And that's why this overview is a bit different and we'll start with bugfix efforts before delving into the few feature additions:



Ville <b>fixed up display fifo underruns</b> all over the place: FDI modeset fixes for Haswell/Broadwell, correctly detecting fused-of VGA on the same,&nbsp; and disabling fifo underrun reporting in some places where we've learned that underruns just happen - mostly around starting up the display pipeline.



Next up is <b>improved runtime PM wakelock debugging</b> from Imre Deak, with efforts from other folks trying to fix up various issues. Unfortunately this turned up so many little buglets that we had to disable the reporting again, at least for now. But at least we'll now have a very clear list of things to address, and a reliable way to audit any failures, to finally be able to enable runtime PM by default hopefully soon.



There's also been lots of <b>fixes for PSR and FBC</b> from Rodrigo and Paulo respectively. PSR enabled by default missed the 4.5 merge window by just a hair - it's already enabled for 4.6. FBC is also pretty close, the last bit Paulo is working is untangling the locking issues. FBC is sitting between GEM and KMS and FBC code gets called by both subsystems. And that's an easy recipe for deadlocks, which Paulo is now working to resolve.



There's also fixes included from Tvrtko Ursulin, Lukas Wunner and Chris Wilson to remedy some long-standing regressions in the <b>fbdev framebuffer setup code</b>. In GEM finally Dave Gordon <b>fixed issues with the page dirty</b> tracking. And Chris Wilson <b>fine-tuned the request polling logic</b> to avoid needlessly wasting CPU cycles.



Imre, Patrik and others have done a lot of work to <b>fix up various issues in the DMC</b> firmware loader for Skylake and the DC5/6 support. It works well now on that platform, and could reenable the overall display power well support again on Skylake. But there's still plenty of issues on Broxton unfortunately.



Since bugfixes have been highly prioritized over feature work this time around there's only very little progress on atomic modesettting and specifically atomic watermark updates. But 4.5 includes a few more prep patches from Maarten Lankhorst and Matt Roper.



There have been some real features though still: Alex Goins from nvidia implementd proper <b>sync for page-flipping dma-buf</b> backed framebuffers, benefitting setups where nVidia renders buffers that the Intel driver displays.



Finally there's also been the usual amount of internal refactoring to prepare the code for the future and keep it maintainable. Jani Nikula <b>rewrote the VBT parsing</b> code.&nbsp; And Ander started to <b>rework the DP detection</b> code as the first step of a large DP support revamp. And finally ther's been a bit of <b>enabling for Kabylake</b> too, but it's not yet complete.



And of courese there's been a lot more smaller things, again mostly bugfixes.