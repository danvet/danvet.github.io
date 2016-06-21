---
layout: post
title: Neat stuff for 3.17
date: '2014-08-07T07:29:00.000-07:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2014-08-07T08:36:55.470-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-3223799358281583581
blogger_orig_url: http://blog.ffwll.ch/2014/08/neat-stuff-for-317.html
---

So with the <a href="http://blog.ffwll.ch/2014/06/neat-drmi915-stuff-for-316.html">3.16 kernel</a> out of the door it's time to look at what's queued up for the Intel graphics driver in 3.17.

<a name='more'></a>

This release features the <b>universal plane</b> support from Matt Roper, all enabled already by default. This is prep work for atomic modesetting and pageflipping support: Since a while we support additional (overlay) planes in the DRM core and the i915 driver, but there have always been two implicit planes directly attached to the CRTC: The primary plane used by the SetCrtc and PageFlip functions, and the optional cursor support. But with the atomic ioctl these implicit planes it's easier to handle everything as an explicit plane, so Matt's patches split them away into separate real plane objects. This is a nice cleanup of the kms api in general since a lot of SoC hardware has unified plane hardware, where cursor, primary plane and any overlays are fully interchangeable. So we already expose this to userspace, if it sets the corresponding feature flag.



Another big feature on the display side is the <b>improved PSR</b> support, which is now enabled by default on Haswell and Broadwell. The tricky bit with PSR (and also with FBC) and the reason we didn't yet enable this by default is correctly support legacy frontbuffer rendering (for example for X). The hardware provides a bit of support to do that, but it doesn't catch all possible frontbuffer rendering and has a lot of other limitations. To finally fix this for real we've added <b>accurate frontbuffer tracking</b> in software. This should finally allow us to enable a lot of display power saving features by default like PSR on Baytrail, FBC (on all platforms) and DRRS (dynamic refresh rate switching).



On actual platform display enabling we have lots of improvements all over: Baytrail MIPI <b>DSI support</b> has greatly stabilized, <b>backlight and power sequencer</b> fixes, <b>mmio based flips</b> to work around issues with stalls and hangs for blitter ring based flips and plenty of other work. The core drm pieces for <b>plane rotation support</b> have also landed, unfortunately the i915 parts didn't make the cut for 3.17.



Another big area, as usual, has been general power management improvements. We now support <b>runtime PM for DPMS Off</b> and not just when the output is completely disabled. This was fairly invasive work since our current modesetting code assumed that a DPMS Off/On cycle will not destroy register state, but that's exactly what runtime PM can do. On the plus side this reorganization greatly cleaned up the code base and prepared the driver for atomic modesetting, which requires a similar separation between state computation and actual hw state updating like this feature.



Jesse Barnes implemented <b>S0ix support</b> for system suspend/resume. Marketing has some crazy descriptions for this, but essentially this means that we use the same power saving knobs for system suspend as for runtime PM - the entire machine is still running, just at a very low power state. Long-term this should simplify our system suspend code a bit since we can just reuse all the code used to implement runtime PM.



Moving on to the render side of the gpu there have been again improvements to the rps code. Chris Wilson further <b>tuned the rps boost</b> logic, and Ville and Deepak implemented <b>rps support for Cherrytrail</b>.

Jesse contributed <b>ppgtt support for Baytrail</b> which will be a lot more interesting once we enable full ppgtt again (hopefully in 3.18).



For <b>Broadwell semaphores</b> support from Ben and Rodrigo was merged, but it looks like we need to disable that again due to stability issues. Oscar Mateo also implemented a large pile of <b>interrupt handling improvements </b>which hopefully address the small races and bugs we've had in the past on some platforms. There's also a lot of refactoring patches to <b>prepare for execlist</b> support from Oscar. Excelists are the new way of submitting work to the gpu, first supported on Broadwell (but not yet mandatory). The key feature compared to legacy ringbuffer submission is that we'll finally be able to preempt gpu tasks.



And as usual there have been tons of bugsfixes and improvements all over. Oh and: User mode setting has moved one step further on the path to deprecation and is now fully disabled. If no one complains about this we can finally rip out all that code in one of the next kernel releases.