---
layout: post
title: Neat drm/i915 stuff for 4.2
date: '2015-06-01T02:14:00.000-07:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2015-06-26T01:58:24.199-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-6166429885530302954
blogger_orig_url: http://blog.ffwll.ch/2015/06/neat-drmi915-stuff-for-42.html
---

The [4.1 kernel release](http://blog.ffwll.ch/2015/04/neat-drmi915-stuff-for-41.html) is still a few weeks off and hence a bit early to talk about 4.2. But the drm subsystem feature cut-off already passed and I'm going on vacation for 2 weeks, so here we go.

<!--more-->

First things first: No, i915 does not yet support atomic modesets. But a lot of progress has been made again towards enabling it. As I explained last time around the trouble is that the intel driver has grown its own almost-atomic modeset infrastructure over the past few years. And now we need to convert that to the slightly different proper atomic support infrastructure merged into the drm core, which means lots and lots of small changes all over the driver. A big part merged in this release is the <b>removal of the -&gt;new_config</b> pointer by Ander, Matt &amp; Maarten. This was the old i915-specific pointer to the staged new configuration. Removing it required switching all the CRTC code over to handling the staged configuration stored in the struct drm_atomic_state to be compatible with the atomic core. Unfortunately we still need to do the same for encoder/connector states and for plane states, so there's still lots of shuffling pending for 4.2.



There has also been other feature work going on on the modeset side: Ville <b>cleaned&amp;fixed up the CDCLK support</b> in anticipation of implementing dynamic display clock frequency scaling. Unfortunately that part of his patches hasn't landed yet. Ville has also merged patches to <b>fix up some details in the CPT modeset sequence</b>, maybe this will finally fix the remaining "DP port stuck" issues we still seem to have.



Looking at newer platforms the interesting bit is <b>rotation support for SKL</b> from Sonika and Tvrtko. Compared to older platforms skl now also supports 90° and 270° rotation in the scanout engines, but only when the framebuffer uses a special tiling layout (which have been enabled in 4.0). A related feature is <b>support for plane/CRTC scalers on SKL</b>, provided by Chandra. Skylake has also gained support for the new <b>low-power display states DC5/6</b>. For <b>Broxton basic enabling has landed</b>, but there's nothing too interesting yet besides piles of small adjustments all over. This is because Broxton and Skylake have a common display block (similar to how the render block for atom chips was already shared since Baytrail) and hence share a lot of the infrastructure code. Unfortunately neither of these platforms has yet left the preliminary hardware support label for the i915 driver.



There's also a few minor features in the display code worth mentioning: <b>DP compliance testing infrastructure</b> from Todd Previte - DP compliance test devices have a special DP AUX sidechannel protocol for requesting certain test procedures and hence need a bit of driver support. Most of this will be in userspace though, with the kernel just forward requests and handing back results. Mika Kahola has <b>optimized the DP link training</b>, the kernel will now first try to use the current values (either from a previous modeset or set up by the firmware). <b>PSR has also seen some more work</b>, unfortunately it's still not yet enabled by default. And finally there's been lots of cleanups and improvements under the hood all over, as usual.



A big feature is the <b>dynamic pagetable allocation for gen8+</b> from Michel Thierry and Ben Widawsky. This will greatly reduce the overhead of PPGTT and is a requirement for 48bit address space support - with that big a VM preallocating all the pagetables is just not possible any more. The <b>gen7 cmd parser</b> is now finally fixed up and enabled by default (thanks to Rebecca Palmer for one crucial fix), which means finally some newer GL extensions can be used without adding kernel hacks. And Chris Wilson has fine-tuned the cmd parser with a big pile of patches to reduce the overhead. And Chris has <b>tuned the RPS boost</b> code more, it should now no longer erratically boost the GPU's clock when it's inappropriate. He has also written a lot of patches to <b>reduce the overhead of execlist</b> command submission, and some of those patches have been merged into this release.



Finally two pieces of prep work: A few patches from John Harrison to <b>prepare for removing the outstanding lazy request</b>. We've added this years ago as a cheap way out of a memory and ringbuffer space preallocation issue and ever since then paid the price for this with added complexity leaking all over the GEM code. Unfortunately the actual removal is still pending. And then Joonas Lahtinen has implemented <b>partial GTT mmap</b> support. This is needed for virtual enviroments like XenGT where the GTT is cut up between the different guests and hence badly fragmented. The merged code only supports linear views and still needs support for fenced buffer objects to be actually useful.



