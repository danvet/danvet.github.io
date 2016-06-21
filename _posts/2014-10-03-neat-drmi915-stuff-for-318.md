---
layout: post
title: Neat drm/i915 stuff for 3.18
date: '2014-10-03T08:47:00.000-07:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2014-11-02T06:31:49.294-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-4075400659743837174
blogger_orig_url: http://blog.ffwll.ch/2014/10/neat-drmi915-stuff-for-318.html
---

Since Dave Airlie moved the feature cut-off of the drm-next tree roughly one month ahead it is already time for our regular look at what's ahead. Even though <a href="http://blog.ffwll.ch/2014/08/neat-stuff-for-317.html">the 3.17 features</a> aren't even released yet.



<a name='more'></a>On the modeset side of things we now have the final pieces for <b>plane rotation support</b> from Sonika Jindal and Ville. The DisplayPort code has also seen lots of improvements, with updated training values in preparation of the latest eDP standard (Sonika Jindal) and support for <b>DP training</b> pattern 3 (Ville). DSI panels now support burst mode (Shobhit) and hdmi conformance has been improved with some fixes from Clint Taylor.



For eDP panels we also have <b>improved panel power sequencing code</b>, mostly to fix issues on Cherryview, from Ville. Ville has also contributed fixes to the VDD handling code, which is used to temporarily enable panel power. And the backlight code learned to handle the bl_power setting so that the backlight can be turned off completely without upsetting the panel's power sequencing, contributed by Jani.



Chris Wilson has also been fairly busy on the modeset code: 3.18 includes his patches to <b>cache EDIDs</b> for a single probe call - unfortunately the full caching solution to keep the EDID around between multiple probe calls isn't merged yet. And <b>pageflips have now improved error detection and recovery </b>logic: In case something goes wrong we shouldn't end up stuck any longer waiting for a pageflip to complete that has been lost by either the hardware or the driver.



Moving on to platform specific work there's been lots of <b>preparations for Skylake</b>, most of it from Damien and Sonika. The actual intial platform enabling is delayed for 3.19 though. On the other end of the timeline Ville <b>fixed up i830M modeset support</b> on a rainy w/e in his vacation, and 3.18 now has all that code. And there has been a lot of <b>Cherryview fixes</b> all over.



Cherryview also gained support for power wells and hence runtime pm (Ville). And for platform agnostic feature a lot of the <b>preparation for DRRS </b>(dynamic refresh rate switching) is merged, hopefully the actual feature patches from Vandana Kannan will land in 3.19.



Moving on the render side of the driver there's been a lot of patches to beat the <b>full ppgtt support into shape</b>. The context code has been cleaned up, lifetime handling for ppgtt address spaces is fixed and bad interactions with secure batches are now also rectified. Enabling full ppgtt missed the feature cutoff by a hair though, but it's already enabling for the following release.



Basic support for <b>execlists command submission</b> from Ben Widawsky, Oscar Mateo and Thomas Daniel was also merged. This is the fancy new way to submit commands available on Gen8 and subsequent platforms. It's not yet enabled by default, but since it's a requirement for a lot of cool new features keep an eye on what's going on here. There is also a lot of work going on underneath to enable all this new code in GEM, like preparing to switch away from sequence numbers to tracking gpu progress more abstractly using the driver's request structures.



And this time around there is also some cool stuff going on in the drm core worth of a shout-out: The <b>vblank handling code is massively revamped</b>, hopefully plugging all the small races, inconsistencies and inefficiencies in that code. And thanks to David Herrmann it is finally possible to write a<b> drm driver without the drm midlayer</b> getting completely in the way of a proper driver load and unload sequence! Unfortunately i915 can't be converted right away since the legacy usermodesetting code crucial relies on this midlayer functionality. But that's now well deprecated and hopefully can be removed in one of the next releases.