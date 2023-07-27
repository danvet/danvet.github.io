---
layout: post
title: Neat drm/i915 stuff for 4.3
date: '2015-09-07T02:40:00.000-07:00'
author: sima
tags:
- Kernel RelNotes
modified_time: '2015-09-07T02:40:13.541-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-5631845111107853206
blogger_orig_url: http://blog.ffwll.ch/2015/09/neat-drmi915-stuff-for-43.html
---

[Kernel 4.2](/2015/06/neat-drmi915-stuff-for-42.html) is
released already and the 4.3 merge window in full swing, time to look at what's
in it for the intel graphics driver.



<!--more-->



Biggest thing for sure is that <b>Skylake is finally out of preliminary support</b> and enabled by default. The reason for the long hold-up was some ABI fumble - the hardware exposes the topmost plane both through the new universal plane registers and the legacy cursor registers and because we simply carried the legacy plane code around in the driver we ended up exposing both. This wasn't something big to take care of but somehow was dragged on forever.



The other big thing is that now <b>legacy modesets are done with the new atomic modesetting code </b>driver-internally. Atomic support in i915.ko isn't ready for prime-time yet fully, but this is definitely a big step forward. Besides atomic there's also other cross-platform improvements in the modeset code: Ville fixed up the <b>12bpc support for HDMI</b>, which is now used by default if the screen supports it. Mika Kahola and Ville also implemented dynamic adjustment of the cdclk, which is the main clock source for display engines on intel graphics. And there's a big difference in the clock speeds needed between e.g. a 4k screen and a 720p TV.



Continuing with power saving features Rodrigo again spent a lot of time <b>fixing up PSR</b> (panel self refresh). And Paulo did the same by writing patches to <b>improve FBC </b>(framebuffer compression). We have some really solid testcases by now, unfortunately neither feature is ready for enabling by default yet. Especially PSR is still plagued by screen freezes on some random systems. Also there's been <b>some fixes to DRRS</b> (dynamic refresh rate switching) from Ramalingam. DRRS is enabled by default already, where supported. And finally some improvements to make the frontbuffer rendering tracking more accurate, which is used by all three of these display power saving features.



And of course there's also tons of improvements to platform code. <b>Display PLL code for Sklylake and Valleyview&amp;Cherryview was tuned</b> by Damien and Ville respectively. There's been <b>tons of work on Broxton and DSI support</b> by Imre, Gaurav and others.



Moving on to the rendering side the big change is how tracking of rendering tasks is handled. In the past the driver just used raw sequence numbers emitted by the hardware, but for cross-driver synchronization and reordering tasks with an eventual gpu scheduler more abstraction is needed. A big step is <b>converting over to the i915 request structure</b> completely, done by John Harrison. The next step will be to switch the internal implementation for i915 requests to the cross-driver fences, but that's for future kernels. As a follow-up cleanup John also <b>removed the OLR</b>, which stands for outstanding lazy request. It was a neat little trick implemented years ago to simplify handling error recovery, but which causes tons of pain with subtle bugs. Making requests more explicit in the driver allowed us to finally remove this trick since.



There's also been a pile of platform related features: <b>MOCS programming for Skylake/Broxton</b> (which is used for caching control). <b>Resource streamer support</b> from Abdiel, which is used to offload some of the buffer object tracking for shaders from the cpu to the gpu. And the command parser on Haswell was extended to <b>support atomic instructions</b> in shaders. And finally for Skylake Mika Kuoppala added code to avoid resetting the gpu - in certain cases the hardware would hard-hang the entire system trying to execute the reset. And a dead gpu is still better than a dead system.

