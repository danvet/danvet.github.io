---
layout: post
title: New Modeset Code
date: '2012-08-19T14:04:00.001-07:00'
author: danvet
tags:
- In-Depth Tech
modified_time: '2013-07-21T06:58:51.499-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-6289609224469029433
blogger_orig_url: http://blog.ffwll.ch/2012/08/new-modeset-code.html
---

drm/i915.ko is gearing up to gain a new modeset code, to finally move away from the crtc helper code (in drm/drm_crtc_helper.c) used by (up to now) all kms drivers. As a quick reference I'll detail the motivation and design of the new code a bit here (mostly stitched together from patchbomb announcements and commits introducing the new concepts).

<a name='more'></a>

The crtc helper code has the fundamental assumption that encoders and crtcs can be enabled/disabled in any order, as long as we take care of depencies (which means that enabled encoders need an enabled crtc to feed them data, essentially).



Our hw works differently. We already have tons of ugly cases where crtc code enables encoder hw (or <code>encoder-&gt;mode_set</code> enables stuff that should only be enabled in <code>enocder-&gt;commit</code>) to work around these issues. But on the disable side we can't pull off similar tricks - there we actually need to rework the modeset sequence that controls all this. And this is also the real motivation why I've finally undertaken this rewrite: eDP on my shiny new Ivybridge Ultrabook is broken, and it's broken due to the wrong disable sequence ...



The new code introduces a few interfaces and concepts:

<ul><li>Add new encoder-&gt;enable/disable functions which are directly called from the   crtc-&gt;enable/disable function. This ensures that the encoder's can be   enabled/disabled at a very specific in the modeset sequence, controlled by our   platform specific code (instead of the crtc helper code calling them at a time it deems convenient).</li><li>Rework the dpms code - our code has mostly 1:1 connector:encoder mappings and   does support cloning on only a few encoders, so we can simplify things quite a   bit.</li><li>Also only ever disable/enable the entire output pipeline. This ensures   that we obey the right sequence of enabling/disabling things, trying to be clever here mostly just complicates the code and results in bugs. For cloneable   encoders this requires a bit of special handling to ensure that outputs can   still be disabled individually, but it simplifies the common case. </li><li>Add infrastructure to read out the current hw state. No amount of careful   ordering will help us if we brick the hw on the initial modeset setup. Which could   happen if we just randomly disable things, oblivious to the state set up by   the bios. Hence we need to be able to read that out. As a benefit, we grow a   few generic functions useful to cross-check our modeset code with actual hw   state.</li></ul>With all this in place, we can copy&amp;paste the crtc helper code into the drm/i915 driver and start to rework it:

<ul><li>As detailed above, the new code only disables/enables an entire output pipe. As a preparation for global mode-changes (e.g. reassigning shared resources) it keeps track of which pipes need to be touched by a set of bitmasks.</li><li>To ensure that we correctly disable the current display pipes, we need to know the currently active connector/encoder/crtc linking. The old crtc helper simply overwrote these links with the new setup, the new code stages the new links in <code>-&gt;new_*</code> pointers. Those get commited to the real linking pointers once the old output configuration has been torn down, before the <code>-&gt;mode_set</code> callbacks are called.</li><li>Finally the code adds tons of self-consistency checks by employing the new hw state readout functions to cross-check the actual hw state with what the datastructure think it should be. These checks are done both after every modeset and after the hw state has been read out and sanitized at boot/resume time. All these checks greatly helped in tracking down regressions and bugs in the new code.</li></ul>With this new basis, a lot of cleanups and improvements to the code are now possible (besides the DP fixes that ultimately made me write this), but not yet done:

<ul><li>I think we should create <code>struct intel_mode</code> and use it as the adjusted mode   everywhere to store little pieces like <code>needs_tvclock</code>, pipe dithering values or   dp link parameters.  That would still be a layering violation, but at least we   wouldn't need to recompute these kinds of things in intel_display.c.   Especially the port bpc computation needed for selecting the pipe bpc and   dithering settings in intel_display.c is rather gross.</li><li>In a related rework we could implement <code>-&gt;mode_valid</code> in terms of   <code>-&gt;mode_fixup</code> in a generic way - I've hunted down too many bugs where -&gt;mode_valid   did the right thing, but <code>-&gt;mode_fixup</code> didn't. Or vice versa, resulting in   funny bugs for user-supplied modes.</li><li>Ditch the idea to rework the hdp handling in the common crtc   helper code and just move things to i915.ko. Which would rid us of the   <code>-&gt;detect</code> crtc helper dependencies.</li><li>LVDS wire pair and pll enabling is all done in the <code>crtc-&gt;mode_set</code> function currently. We should be able to move this to   the crtc_enable callbacks (or in the case of the LVDS wire pair enabling, into some encoder callback).</li></ul> Last, but not least, this new code should also help in enabling a few neat features: The hw state readout code prepares (but there are still big pieces missing) for fastboot, i.e. avoiding the inital modeset at boot-up and just taking over the configuration left behind by the bios. We also should be able to extend the configuration checks in the beginning of the modeset sequence and make better decisions about shared resources (which is the entire point behind the atomic/global modeset ioctl). 