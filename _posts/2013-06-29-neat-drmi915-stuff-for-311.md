---
layout: post
title: Neat drm/i915 stuff for 3.11
date: '2013-06-29T07:46:00.001-07:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2013-07-21T06:48:11.116-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-5299773283244896906
blogger_orig_url: http://blog.ffwll.ch/2013/06/neat-drmi915-stuff-for-311.html
---

<a href="http://blog.ffwll.ch/2013/04/neat-drmi915-stuff-for-310.html">Kernel 3.10</a> will be release soon, so it's time for our regular look at what the next kernel will bring. Purely on statistics the drm-intel-next pull for 3.11&nbsp; is one of the biggest pull request with 314 patches thus far. Read on after the break for the details.



<!--more-->

The big item is surely that <b>ValleyView</b> (now called <b>Baytrail</b>) support is now solid enough that we've lifted the preliminary hardware support tag for it and enabled it by default. Support for this SoC platform should be fairly complete now with rc6, render turbo and all outputs (with the exception of MIPI/DSI panels all nicely working now. Most of the code is from Jesse Barnes with a bunch of patches from Ville Syrjälä and also from the Bangalore display guys.



For <b>Haswell</b> there's a big effort to <b>enable power saving features</b>: Paulo Zanoni fixed up our <b>watermark code</b> and enabled the <b>intermediate pixel storage</b> feature. Rodrigo Vivi enabled <b>framebuffer compression</b>. Both changes greatly increase deep package C state residency and hence power consumption on idle systems when the display is enabled. Furthermore the <b>audio</b> guys around Haihao Xiang fixed up the interaction of the intel alsa driver with our driver when the manual <b>display power well</b> is shut down, further improving power consumption figures.



On the Haswell render side there's finally support for the <b>VECS engine</b>.<b> </b>This engine is used by libva for some post-processing steps.



There are also some further <b>hotplug improvements</b> from Egbert Eich.<b> </b>Chris Wilson has some further ideas to cut down on needless reprobing (and so speed up boot and resume times), but I've decided that we should test the current changes (especially the hotplug storm detection code merged in 3.10) a bit more - hardware is simply too broken in this area to play fast&amp;loose.



Mika Kuoppala worked a lot on our hangcheck &amp; reset code and improved our context <b>support to enable the GL_ARB_robustness</b> extension. Unfortunately the new ioctl wasn't merged for 3.11 since one of the last prep patches blew up in a bad way and so had to be dropped. But Mika also <b>fixed our long-standing out-of-memory issues with dumping the gpu error state</b> - finally no bug report should be blocked any more because the reporter can't grab the error dump!



Some of the context work also prepares for real per-process gpu address space. Ben Widawsky is working on that and also submitted quite a few <b>patches for our GTT code</b> to prepare for this. Hopefully 3.12 will have the first cut of that code, and if Daniel Herrman's rendernode GSOC project also lands we'd finally have much improved security between different gpu clients.



Another improvement which probably fewer people will notice is that we've revamped and improved our code to handle <b>non-24bpp modes</b> - we can finally drive DP screens at 30bpp (although there's still work to do, as always), and also fixed up a few other corner cases. This is all part of the larger effort to reorganize our modesetting code. The big change in 3.11 though is the <b>addition of the pipe config structure</b> to track the per-crtc modesetting state. We now also have code to <b>cross-check the hardware state</b> with the software state after each modeset (similarly to how we already cross-check the output routing). This will help greatly to catch regression earlier, especially for some of the big upcoming features like fastboot or atomic modeset.



In 3.11 though we've just merged preparatory patches to convert as much as possible of our crtc state to the new infrastructure: Panel fitter state, pipe bpp and dithering modes, display pll sharing (and some of the dpll hw state) and the basic output mode properties are now all tracked in the pipe configuration structure. For those curious about what's the design behind this new code framework and how it'll work out eventually: Stay tuned to this blog, I'm working on a more in-depth coverage.



And of course there's been tons of little fixes, improvements and code refactoring all over the place.
