---
layout: post
title: Neat drm/i915 stuff for 3.12
date: '2013-09-02T07:01:00.002-07:00'
author: sima
tags:
- Kernel RelNotes
modified_time: '2013-09-02T07:01:44.091-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-7576649732790701078
blogger_orig_url: http://blog.ffwll.ch/2013/09/neat-drmi915-stuff-for-312.html
---

[The linux kernel 3.11](/2013/06/neat-drmi915-stuff-for-311.html) will be released
soonish and it's time for our regular look at what the next merge window will
bring in for the intel GPU driver.

<!--more-->

For Haswell enabling there are two new power saving features in 3.12: The first is <b>pc8+ support</b> from Paulo Zanoni. This is a new, deeper system sleep state for the new Haswell ULT platforms where the PCH is integrated onto the same package - pc stands for "package C state". Since waking up from this will still be much faster than waking from a full resume this will help for always-on use-cases like tablets. This support required a lot of changes in many parts of the driver since we're essentially doing part of what the firmware has done for us at suspend/resume time, but now we do this ourselves at runtime. Most of the changes have been to setup/restore additional reference clocks and to improve our handling of interrupts - the added benefit of this power saving mode compared to full device (or even system suspend) is that hotplugging still works.



The other Haswell power saving feature is <b>panel self refresh</b> support (PSR
for short), polished for merging by Rodrigo Vivi. Note that we still have a few
issues with properly tracking frontbuffer rendering (used by X without a
pageflippping compositor) without completely killing the power saving advantages
on more usual setups. Right now enabling PSR will cause render issues on such
legacy X setups and hence is disabled by default.

On the other end in the power-vs-performace scale we've tuned our Haswell/Iris
support. Ben Widawsky enabled the giant <b>eLLC cache</b> (the seperate 128 MB
cache chip on some Iris packages). And Chris Wilson enabled the <b>write-through
scanout buffer caching mode</b>, again an Iris-only feature used to improve the
effectiveness of the eLLC cache.

Looking at general improvements we've finally moved the <b>gpu error state into
a sysfs file</b> (the old debugfs file will stay around). So now with this
official interface we can support users better even on production systems where
debugfs really shouldn't be mounted. This was all made possible by Mika
Kuoppala's improvements, which mostly landed in 3.12 already.

Under the hood there has been lots of improvements to the codebase. Damien
Lespiau, based on work from Paulo Zanoni finally <b>converted our HDMI infoframe
code</b> over to the common helpers introduced into the kernel half a year ago.
Code sharing, especially for sink handling code is always nice. Also in our
display code we've rid ourselves of the last remnants of the old crtc helper
based modesetting code. This completes the [massive modesetting
rework](/2012/08/new-modeset-code.html) started in kernel
3.8.

But we don't stand still and the next big thing is atomic pageflipping. One of the really tricky bits there is to update the <b>display watermarks</b> atomically, even when we completely switch the plane, sprite, cursor, ... configuration. Hence why Ville Syrjälä has written lots of patches to convert our existing code into a shape more suitable for such atomic updates.



Another display area where big changes are on the horizon is fastboot support. We've further improved the pipe configuration tracking introduced in the last kernel. Now we keep track of enough state that with a few small hacks from Jesse Barnes we finally have <b>fastboot support</b> merge. But due to these hacks this is still hidden behind a module option. Still, thanks to the pipe configuration tracking and checking we should be able to get decent testcoverage on these bits now. And hopefully we can plug the few misssing pieces in our state tracking and enable fastboot as a default option in the next kernel. 



More on the GEM side of things Ben Widawsky has written support for real per-process gpu address spaces. It's a really big feature requiring changes all over the driver. The big change is the <b>introduction of VMA objects</b> (short for virtual memory area) to track the different mappings of the same underlying objects into different gpu address spaces. Most of this preparatory work is already merged into 3.12 and we should have real per-process address space support in 3.13. Together with the already merged render nodes support from David Herrmann the PPGTT code from Ben Widawsky will give us fully drm client insulation on recent Intel platforms. That should be a really nice feature for background GPU workloads like video transcoding or OpenCL.



Finally Jesse Barnes contributed a set of patches to properly <b>reserve the graphics stolen memory</b> region. Stolen memory is a carveout region set up by the firmware for exclusive usage by the graphics driver. Since the i915 GEM driver is designed to fully exploit the unified memory architecture of Intel gpus we've never really used this memory much. But recently we've started to put it to better use (since otherwise it's just wasted) and noticed that at least on some systems the BIOS doesn't mark this range correctly as reserved. Which allows the linux kernel to place various pci mmio resources and other nasty stuff in this range, resulting in potential issues and system hangs. Thus far this wasn't a real issue since we haven't really used stolen memory for much at all. But thanks to Jesse's patches we can now move ahead.



Of course there have been tons of other small improvements and bugfixes all over the place: The valleyview code received quite some polish. The interrupt code received some polish after the big changes in the previous kernel (and also to prepare for pc8+) and we have tons of little bugfixes for some of the more evil testcase we've added and also a few bugfixes for special setups and platforms. 
