---
layout: post
title: Neat drm/i915 stuff for 3.7
date: '2012-10-02T03:37:00.001-07:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2013-07-21T07:00:05.252-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-4281497260812145729
blogger_orig_url: http://blog.ffwll.ch/2012/10/neat-drmi915-stuff-for-37.html
---

Now that the upstream merge window has started and my last drm-intel-next pull request landed in Dave Airlie's drm tree it's a good time to look at some of the things landing in 3.7:

<!--more-->

The big ticket item is the <b>[modeset-rework](http://blog.ffwll.ch/2012/08/new-modeset-code.html)</b>. On top of that we have some <b>eDP fixes</b> that required the new modeset infrastructure to work properly. Unfortunately the Haswell DP and eDP support did not make it - we've had patches for 3.6, but they've been too ugly to merge. Paulo Zanoni reworked all these patches on top of the new modeset code, unfortunately the new patches missed 3.7.  Also, thanks to these eDP fixes I could finally merge <b>CADL support</b>, this should help to light up the backlight on some obnoxious platforms with a BIOS that tries to get in the way of the kms driver ...



There are also tons of <b>improvements to the Haswell HDMI</b> support and stability fixes for Haswell rendering from Paulo Zanoni. And Ben Widawsky enabled <b>hw context support</b> <b>on Haswell</b> (required for geometry shader support) and a few other more obscure hw features for Haswell.



For power management Ben Widawsky added tons of <b>sysfs interfaces to control gpu frequency scaling</b> on Sandybridge and newer, e.g. to limit the gpu to the lowest frequency for that last ounce of battery runtime. PowerTOP will use this to show a histogram of gpu frequencies, to complete the rc6 histogram support already merged into 3.6.



On the GEM side Chris Wilson provided tons of tuning patches to etch out a bit more performance in corner cases: We now write <b>PTEs with write-combine mappings</b> where the (cacheable) ppgtt ptes are not available. He also <b>optimized the gpu flushing code</b> between batch buffers and reduced the overhead of the gpu-&gt;cpu synchronization on newer platforms. Also merged are his <b>cachable BO support</b> and the <b>unbound BO tracking</b>, both of which should help boost SNA nicely on pre-Sandybridge machines. And we now also have <b>non-blocking gpu waits</b>, which greatly helps interactivity when more than one drm client is using the gpu - which is the case as soon as you run a compositor using OpenGL. Beware benchmarkers though: A side-effect of this is that the gpu is shared more fairly between the different clients - this can result in reduced fps when running a windowized GL app, since Compiz now draws more frames itself, sucking away gpu time from the GL app. End result is though that more rendered frames end up actually visible on the screen. 



A big cleanup under the hood of GEM is the <b>removal of the flushing_list</b>. This concept dates back to the first days of GEM years ago, but turned out to be totally useless. Unfortunately it also complicated the code massively, resulting in countless little bugs - we've first tried to rip this all out at the beginning of this year, but only for 3.7 we've managed to track down all the bugs this uncovered.



Last but not least countless other small improvements and code-cleanups all over the place. Oh, and: In case you have a <b>i830M with a ns2501</b> lvds dvo chip: We now have basic support for that, thanks to Thomas Richter.pz