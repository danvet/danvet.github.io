---
layout: post
title: Neat drm/i915 stuff for 3.15
date: '2014-04-01T04:03:00.000-07:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2014-04-01T04:03:13.886-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-5069398136577877315
blogger_orig_url: http://blog.ffwll.ch/2014/04/neat-drmi915-stuff-for-315.html
---

So the release of the [3.14 linux kernel](https://www.blogger.com/%3Cdegasus%3E%20but%20when%20implementing%20running%20based%20on%20walking%20needs%20some%20major%20api%20changes%20:/) already happended and I'm a bit late for our regular look at what cool stuff will land in the 3.15 merge window for the Intel graphics driver.



<!--more-->

The first big thing is Ben Widawsky's support for <b>per-process address spaces</b>. Since a long time we've already supported the per-process gtt page tables the hardware provides, but only with one address space. With Ben's work we can now manage multiple address spaces and switch between them, at least on Ivybridge and Haswell. Support for Baytrail and Broadwell for this feature is still in progress. This finally allows multiple different users to use the gpu concurrently without accidentally leaking information between applications. Unfortunately there have been a few issues with the code still so we had to disable this for 3.15 by default.



Another really big feature was the much more <b>fine-grained display power domain</b> handling and a lot of <b>runtime power management infrastructure</b> work from Imre and Paulo. Intel gfx hardware always had lots of automatic clock and power gating, but recent hardware started to have some explicit power domains which need to be managed by the driver. To do that we need to keep track of the power state of each domain, and if we need to switch something on we also need to restore the hardware state. On top of that every platform has it's own special set of power domains and how the logical pieces are split up between them also changes. To make this all manageable we now have an extensive set of display power domains and functions to handle them as abstraction between the core driver code and the platform specific power management backend. Also a lot of work has happened to allow us to reuse parts of our driver resume/suspend code for runtime power management. Unfortunately the patches merged into 3.15 are all just groundwork, new platforms and features will only be enabled in 3.16.



Another long-standing nuisance with our driver was the take-over from the firmware configuration. With the [reworked modesetting infrastructure](http://blog.ffwll.ch/2012/08/new-modeset-code.html) we could take over the output routing, and with the experimental i915.fastboot=1 option we could eschew the initial modeset. This release Jesse provided another piece of the puzzle by allowing the driver to <b>inherit the firmware framebuffer</b>. There's still more work to do to make fastboot solid and enable it by default, but we now have all the pieces in place for a smooth and fast boot-up.



There&nbsp; have been <b>tons of patches for Broadwell</b> all over the place. And we still have some features which aren't yet fully enabled, so there will be lots more to come. Other smaller features in 3.15 are <b>improved support for framebuffer compression</b> from Ville, again more work still left. <b>5.4GHz DisplayPort</b> support, which is a required to get 4k working. Unfortunately most 4k DP monitors seem to expose two separate screens and so need a adriver with working MST (multi-stream support) which we don't yet support. On that topic: The i915 driver now uses the <b>generic DP aux helpers</b> from Thierry Redding. Having shared code to handle all the communication with DP sinks should help a lot in getting MST off the ground. And finally I'd like to highlight the <b>large cursor support</b>, which should be especially useful for high-dpi screens.





And of course there's been fixes and small improvements all over the place, as usual.
