---
layout: post
title: Neat drm/i915 stuff for 3.10
date: '2013-04-18T14:29:00.003-07:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2013-07-21T06:50:00.722-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-5100670050437314612
blogger_orig_url: http://blog.ffwll.ch/2013/04/neat-drmi915-stuff-for-310.html
---

So kernel <a href="http://blog.ffwll.ch/2013/02/neat-drmi915-stuff-for-39.html">3.9</a> should be releasing really soon and so it's time for our regular look at what 3.10 brings to drm/i915:



<a name='more'></a>For enthusiast (i.e. people who like to see their hw burn down in flames ...) the <b><a href="http://blog.ffwll.ch/2013/03/overclocking-your-intel-gpu-on-linux.html">improved overclocking</a></b> support is certainly the interesting bit. Thanks to Ben Widawsky's patches we now correctly detect the gpu turbo limit and set the non-turbo frequency as the default limit to avoid hanging systems right on boot. So GPU overclocking should now work on Sandybridge and Ivybridge - apparently something changed on Haswell. Related is Chris Wilson's patch to <b>tune the Haswell turbo support</b> properly - Haswell has new frequency domains and so needs a different control table to ramp up the ring frequency when the GPU is busy.



A big pain relief, at least on affected systems, is Ebgert Eich's <b>hotplug irq storm mitigation</b> work. With the <a href="http://blog.ffwll.ch/2013/02/new-kernel-modesetting-locking.html">modeset locking rework</a> from 3.9 this should get rid of the last reason to boot with the <code>drm_kms_helper.poll=0</code> option. Finally no longer a sluggish cursor and delayed screen updates!



Another neat improvement is the <b>vt-switchless suspend/resume</b> support from Jesse Barnes. This goes back to the user modesetting days, where the kernel forced a vt-switch to the vga linux console to make sure the X server properly saved and afterwards again restored the display state. With kernel modesetting that's pretty pointless and results in an ugly console cursor appearing quickly on resume (usually you can't spot it on suspend ...). But those days are past now.



<b>Display-less gpu support </b>is probably not something many people will care about. But Ben Widawsky's patches allows us to run on special Ivybridge server configurations, where all the display parts of the gpu are fused of and only the GT is used for e.g. video transcoding. Which is pretty cool since for a long time Intel graphics was only about display things, with a comparetively puny gpu attached to it. Times are changing ...



For driver internals the big change is the introduction of the <b>pipe configuration tracking</b>. This is pretty much just a continuation of the <a href="http://blog.ffwll.ch/2012/08/new-modeset-code.html">modeset rework started in 3.7</a> and serves two main purposes: First this will allow us to precompute the desired display pipe configuration before we start to touch the hardware. This is required to make atomic modeset operations actually useful: Userspace can then just ask the kernel which configurations work very quickly and more important without causing any flickering, before deciding on a given setup. The second reason is that for fastboot we need to track the display state left behind by the BIOS precisely - integrating that tracking into the established hw state readout and cross-check support will make sure that this tracking is actually reliable.



3.10 only contains the basic infrastructure though and moves only a few basic attributes over to it (like the pipe bpp values, dither settings, color space conversion and a few internal states used by different platforms to keep track of enabled chip functions). Our aim for 3.11 is to tackle the really big things like clock sharing, output mode and especially dotclock reconstruction and so lay the groundwork for solid fastboot support.



Again purely driver-internal was the massive <b>low-level GTT interface rework</b> from Ben Widawsky. This will help a lot in finally implementing real per-process gpu address spaces on Intel hardware, and it should also simplify enabling of future hardware platforms since that has now a clearly-defined and well-separated interface. Related is Imre Deak's little cleanup to abstract all our scatter-gather list walking with a <code>for_each_sg_page</code> iterator.



Finally I want to point out the <b>pageflip improvements</b> from Ville Syrjälä. Compositors should now no longer get stuck after a gpu hang, gen2-4 received vblank interrupt and pageflip completion interrupt handling fixes and modesetting or panning operations should now also be able to survive gpu hangs without resulting in deadlocks. And the best part is that we have full coverage for all these corner cases in our kernel test suite, so these bugs should be gone for good.



Last but not least there's been the usual big pile of small&amp;large improvements all over: More vlv patches, backlight improvements and tons of bugfixes all over. For amusement maybe take a look at Chris Wilson's <b>bring a bigger gun</b> <a href="http://cgit.freedesktop.org/~danvet/drm-intel/commit/?h=drm-intel-next-queued&amp;id=6ef2ba0d558e55312af8406093c62bd61216b991">coherence fix</a>.