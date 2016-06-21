---
layout: post
title: Neat drm/i915 stuff for 3.16
date: '2014-06-10T02:16:00.000-07:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2014-06-10T02:30:51.439-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-1326050572154159414
blogger_orig_url: http://blog.ffwll.ch/2014/06/neat-drmi915-stuff-for-316.html
---

Linus decided to have a bit fun with the 3.16 merge window and the [3.15
release](/2014/04/neat-drmi915-stuff-for-315.html), so I'm a
bit late with our regular look at the new stuff for the Intel graphics driver.

<!--more-->

First things first, Baytrail/Valleyview has finally gained support for <b>MIPI DSI panels</b>! Which means no more ugly hacks to get machines like the ASUS T100 going for users and no more promises we can't keep from developers - it landed for real this time around. Baytrail has also seen a lot of polish work in e.g. the infoframe handling, power domain reset, ...



Continuing on the new hardware platform this release features the first version of our <b>prelimary support for Cherryview</b>. At a very high level this combines a Gen8 render unit derived from Broadwell with a beefed-up Valleyview display block. So a lot of the enabling work boiled down to wiring up existing code, but of course there's also tons of new code to get all the details rights. Most of the work has been done by Ville and Chon Ming Lee with lots of help from other people.



Our modeset code has also seen lots of improvements. The user-visible feature is surely <b>support for large cursors</b>. On high-dpi panels 64x64 simply doesn't cut it and the kernel (and latest SNA DDX) now support up to the hardware limit of 256x256. But there's also been a lot of improvements under the hood: More of Ville's <b>infrastructure for atomic pageflips</b> has been merged - slowly all the required pieces like unified plane updates for modeset, two stage watermark updates or atomic sprite updates are falling into place. Still a lot of work left to do though. And the modesetting infrasfrastucture has also seen a bit of work by the almost complete <b>removal of the <code>-&gt;mode_set</code> hooks</b>. We need that for both atomic modeset updates and for proper runtime PM support.



On that topic: <b>Runtime power management</b> is now enabled for a bunch of our recent platforms - all the prep work from Paulo Zanoni and Imre Deak in the past few releases has finally paid off. There's still leftovers to be picked up over the coming releases like proper runtime PM support for DPMS on all platforms, addressing a bunch of crazy corner cases, rolling it out on the newer platforms like Cherryview or Broadwell and cleaning the code up a bit. But overall we're now ready for what the marketing people call "connected standy", which means that power consumption with all devices turned off through runtime pm should be as low as when doing a full system suspend. It crucially relies upon userspace not sucking and waking the cpu and devices up all the time, so personally I'm not sure how well this will work out really.



Another piece for proper atomic pageflip support is the <b>universal primary plane</b> support from Matt Roper. Based upon his DRM core work in 3.15 he now enabled the universal primary plane support in i915 properly. Unfortunately the corresponding patches for cursor support missed 3.16. The universal plane support is hence still disabled by default. For other atomic modeset work a shout-out goes to Rob Clark who's locking conversion to <b>wait/wound mutexes</b> for modeset objects has been merged.



On the GEM side Chris Wilson massively <b>improved our OOM handling</b>. We are now much better at surviving a crash against the memory brickwall. And if we don't and indeed run out of memory we have much better data to diagnose the reason for the OOM. The <b>top-down PDE allocator</b> from Ben Widawsky better segregates our usage of the GTT and is one of the pieces required before we can enable full ppgtt for production use. And the <b>command parser</b> from Brad Volkin is required for some OpenGL and OpenCL features on Haswell. The parser itself is fully merged and ready, but the actual batch buffer copying to a secure location missed the merge window and hence it's not yet enabled in permission granting mode.



The big feature to pop the champagne though is the <b>userptr</b> support from Chris - after years I've finally run out of things to complain about and merged it. This allows userspace to wrap up any memory allocations obtained by <code>malloc()</code> (or anything else backed by normal pages) into a GEM buffer object. Useful for faster uploads and downloads in lots of situation and currently used by the DDX to wrap X shmem segments. But OpenCL also wants to use this.



We've also enabled a few <b>Broadwell features</b> this time around: <b>eDRAM</b> support from Ben, <b>VEBOX2</b> support from Zhao Yakui and <b>gpu turbo</b> support from Ben and Deepak S.



And finally there's the usual set of improvements and polish all over the place: <b>GPU reset improvements</b> on gen4 from Ville, <b>prep work for DRRS</b> (dynamic refresh rate switching) from Vandana, tons of <b>interrupt and especially vblank handling rework</b> (from Paulo and Ville) and lots of other things.
