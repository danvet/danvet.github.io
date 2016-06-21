---
layout: post
title: Neat drm/i915 stuff for 3.19
date: '2014-12-16T13:03:00.000-08:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2014-12-16T13:03:06.285-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-6057911954792525149
blogger_orig_url: http://blog.ffwll.ch/2014/12/neat-drmi915-stuff-for-319.html
---

So [kernel version 3.18](http://blog.ffwll.ch/2014/10/neat-drmi915-stuff-for-318.html) is out the door and it's time for our regular look at what's in the next merge window.

<!--more-->First looking at new hardware the big item is <b>basic Skylake support</b>. There are still a few smalls things missing, but mostly it's there now. This has been contributed by Damien, Satheeshakrishna and a lot of other folks. Looking at other platforms there has also been a lot of <b>changes for vlv/chv</b>: <b>Improved</b> <b>backlight code</b>, completely <b>refactored interrupt handling</b> to bring it in line with other platforms, <b>rewritten panel power sequencing</b> code, all from Ville. Rodrigo contributed <b>PSR support</b> for vlv/chv together with a lot of other fixes for PSR. Unfortunately it's not yet again enabled by default.



Moving on to Broadwell and the render side of things, Mika and Arun provided patches to <b>improve the render workaround code</b> and bring the set of workarounds up to date. <b>execlist</b> (the new command submission support on Gen8+) is also being polished with the addition of <b>on-demand pinning</b> of context objects with patches from Thomas Daniel and Oscar Mateo. Finally the RPS/render-turbo code has seen a lot of polish from Imre with a few fixes from Tom O'Rourke.



Otherwise not a lot of really big things happened on the GEM side: Just a few  patches to fix issues in ppgtt (unfortunately still not enabled by  default anywhere due to fun with context switches). And there's a bit of  prep work and reorg all over for new stuff landing hopefully soon.



Looking at overall infrastructure changes the big thing certainly is the <b>preparations for atomic display updates</b>. The drm core/driver interface for atomic and all the helper library code to convert drivers has landed in 3.19, and already some conversions. On the Intel side it's been just prep work under the hood thus far with patches from Ander to <b>precompute display PLL state</b>. The new code to <b>use vblank evades for pagelips</b> has also landed, which is needed for atomic plane updates. And prep patches from Gustavo Padovan started to <b>split the low-level plane update functions into check and commit</b> steps. Lots more patches from different people are in flight and some have been merged for 3.20 already.



Besides these driver internal changes for atomic there has been other work to improve the codebase: Imre <b>reorganized our handlers for suspend, resume </b>and thawing and freezing. Jani <b>reworked the audio and eld code</b> which is the gfx side of the puzzle needed to make audio over HDMI or DP work. Jesse provided patches to <b>track infoframes</b> more accurately, which is needed to correctly fastboot (i.e. without modesets if possible) on external screens.



For older machines Ville has spent a few spare cycles to make them more useful: GPU <b>reset support for gen3/4</b> should mitigate some of the recent chromium crashes on mesa, and the <b>modeset code on i830M might work correctly</b> for the first time, ever.





And of course the usual pile of smaller fixes and improvements all over.



Not directly related to code or features is the start of <b>documenting i915 driver internals</b>: With this release we now have some of the interrupt handling, fifo underrun reporting, frontbuffer tracking and runtime pm support newly document. And there's lots more in-flight, so hopefully soonish this will be fairly useful.