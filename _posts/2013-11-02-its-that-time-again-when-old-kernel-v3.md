---
layout: post
title: Neat drm/i915 stuff for 3.13
date: '2013-11-02T07:36:00.000-07:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2014-01-17T04:57:04.272-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-2495710684294338878
blogger_orig_url: http://blog.ffwll.ch/2013/11/its-that-time-again-when-old-kernel-v3.html
---

It's that time again when the old [kernel
v3.12](/2013/09/neat-drmi915-stuff-for-312.html) will be released
shortly and we'll take a look at what's in store for the Intel GFX driver in
3.13.

<!--more-->

Formerly known as Valleyview, now called <b>Baytrail</b> has seen a lot of improvements. Jesse Barnes has crawled through bugzilla and fixed a lot of little issues all over the place. Chon Ming Lee also provided a few patches to initialize the chip without the VBIOS' help. Jani Nikula has provided the first cut at <b>MIPI DSI support</b>, but that's still all rather preliminary and still a bit of work to do.



This release has also seen a lot of changes, most of them preparatory work, for new power management features. Jani implemented <b>SWSCI support</b> which is a new kind of OpRegion/ACPI based low power management framework for Intel graphics. Itself it's not terribly interesting, but unfortunately a lot of platform power management code in the ACPI reference implementations for Haswell platforms is linked to SWSCI. So now the power gating for e.g. audio codecs is contingent on the gfx being switched off, which obviously doesn't make too much sense. We can blame this on Windows' power management framework being sub-par, but unfortunately we can't fix it. There's also been a lot of work in handling different power domains better (not all of which managed to get into 3.13): Ville Syrjälä implemented VGA power domains and Imre Deak and Paulo Zanoni <b>reworked the power domain handling</b> in general a lot. This is all preparatory work for power saving features on newer platforms, so stay tuned for more in the next kernel cycle. For the curious, most of the patches have already been posted to intel-gfx.



Another power/performance feature is the <b>gpu boost/deboost</b> logic from Chris Wilson. This will help for interactive gpu workloads where the hardware based boost logic is too slow to adapt. On the flip side the kernel is now also much more aggressive at deboosting the gpu when nothing is going on, which should improve power consumption figures for very light, but spikey workloads.



Damien Lespiau provided basic support for <a href="http://damien.lespiau.name/blog/2013/10/02/hdmi-sterero-3d-kms/"><b>3D/stereo displays</b> on HDMI</a>. Currently there's only a small test application available, but patches to support 3D modes in Wayland are already in the works. Hopefully soon we'll have glxgears spinning in 3D! As a quick aside I've just noticed that in my 3.12 release notes I've forgotten to mention the <b>HDMI 4K</b> support, again from Damien.



Under the hood of the driver we've seen more <b>VMA prep patches for PPGTT</b> from Ben Widawsky. And Ville has again been really busy trying to beat our <b>watermark code</b> into shape for a brave new world where atomic updates of the primary plane and any set of sprites actually works in a reliably way.



Now the one feature that makes me really happy is the <b>display CRC support</b> from Damien and He Shuang. This exposes the hardware checksum features through debugfs. Since these checksums update for each frame displayed and the tap point in the display pipe can be selected to be after all the cursor/plane/sprite blending, color correction and scaling have been applied this will finally allow us to test a lot of the modesetting code in a fully automated way. Ville has already provided a testcase for cursor placement and found a few more bugs while writing it.



And of course there's been countless little bugfixes and improvements to the driver internals: DP fixes from Jani, improved tuning values for Haswell DDI ports from Paulo, the option to compile the driver without CONFIG_FB support in the kernel, small improvements to the hw context support from Ben and Chris and tons of other stuff.



Oh and: We should have a little surprise ready for 3.13, too.



