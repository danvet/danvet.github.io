---
layout: post
title: Neat drm/i915 stuff for 3.14
date: '2014-01-15T09:29:00.001-08:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2014-01-15T09:29:24.124-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-7722245516833600559
blogger_orig_url: http://blog.ffwll.ch/2014/01/neat-drmi915-stuff-for-314.html
---

<a href="http://blog.ffwll.ch/search/label/Kernel%20RelNotes">Kernel v3.13</a> is nearing its release, so it's time at our regular look at what the next version will bring to the Intel gfx driver.

<a name='more'></a>

This time around there was a lot of infrastructure work for improved power management. Imre Deak implemented code to <b>manage display power wells</b> in a more generic way so that the current Haswell support is also ready for other platforms. Unfortunately new platform support hasn't landed yet. But for Haswell we now have working <b>runtime D3</b> support, contributed by Paulo Zanoni. For Baytrail/Valleyview Deepak S provided improvements to the <b>GT power well&nbsp; force-wake </b>code - on these mobile platforms it is split up between the render engine and more auxiliary engines like the blitter and the video decode hardware.



Back to the topic of display and power management Ville Syrjälä contributed a lot of patches to fix little issues in the <b>watermark computation</b> code and als for the <b>framebuffer compression</b> support. The motivation for this is Ville's atomic pageflip feature work which not only needs to update all the scanout targets (primary plane, sprites, ...) but also all derived state perfectly synchronized to the next vblank. And especially the watermark updates have proven to be really tricky to get right. Finally Jani Nikula has completely <b>rewritten our low-level backlight</b> code. This has already fixed a few bugs, and hopefully this new code will give us fewer headaches in the future than the old one. Note that this is just about the raw backlight interface provided by Intel's integrated gpu, almost all machines use some other PWM or at least have some fanciness (written in ACPI scripts) on top of that. So don't expect your backlight to magically work perfectly, this is just a tiny piece of the puzzle.



Looking at GEM related features the big thing is the <b>reset statistics ioctl</b> from Mika Kuoppala. This is the kernel code to implement the GLX_ARB_robustness extension, and the further building blocks on top of that. This extension is shipping in Mesa 10.0.x releases and is necessary to implement browser support for WebGL in a sane way. Thjs release cycle has also seen more patches from Ben Widawsky <b>towards full ppgtt support</b>, but the final bits didn't make the cut due to a few regressions that popped up at the last minute.



For platform support the big thing is that we've removed the prelimary hardware support tag for <b>Broadwell</b>, which means 3.14 and later kernels will use the real i915.ko driver by default. There's still a few features missing for full-blown Broadwell kernel support, but all the basics to boot the system and so avoid users getting stuck with black screens are there. <b>Baytrail</b> also has seen lots of smaller improvements, like a vga hotplug fix from Imre, DSI improvements from Deepak and tons of other patches from Jesse Barnes, Ville and Chon Lee Ming.



One last piece new in 3.14 is the <b>deprecation of the legacy UMS support</b>. We've kept this code on live support since a few years already, but now it's getting in the way of some of the plans for improving the driver load and teardown code. So long-term we want this gone. For now there's still a kernel config option to keep the code around. Another new&amp;nifty <b>kernel option disables the legacy fbdev support</b>. This is not something for your regular desktop linux system yet since the boot splash and a bunch of other things rely on it. And you also need to manually make sure that the vga driver doesn't load. But it does allow us to rip out all the legacy linux framebuffer support code, which is beneficial for embedded systems. And with efforts like <a href="http://dvdhrm.wordpress.com/2012/08/11/kmscon-linux-kmsdrm-based-virtual-console/">David Herrmann's kmscon</a> also where the future is heading towards in general.