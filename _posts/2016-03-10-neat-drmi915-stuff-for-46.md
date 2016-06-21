---
layout: post
title: Neat drm/i915 stuff for 4.6
date: '2016-03-10T01:51:00.000-08:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2016-03-10T02:27:44.724-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-6153199226047072843
blogger_orig_url: http://blog.ffwll.ch/2016/03/neat-drmi915-stuff-for-46.html
---

The <a href="http://blog.ffwll.ch/2016/01/neat-drmi915-stuff-for-45.html">4.5 release </a>is close, it's time to look at what's in store for the next kernel's merge window in the Intel graphics driver.

<a name='more'></a>

Headline features for sure are that <b>FBC and PSR are enabled by default</b>. And this time around I'm really hopeful that it will stick, since Paulo&amp;Rodrigo have done a stellar job hunting down all the corner cases, writing testcases for them all and in general making sure we have a really solid foundation for display power saving features. There's still some oddball cornercases, which means it's not yet enabled everywhere and on all platforms, but like I said: Looking really good, and the culmination of over 1 year of effort to get the code infrastructure fixed up and solid.



Another project <b>ongoing is atomic display support</b>, with again lots of work from Maarten and Matt and others to move things forward. Specifically Maarten adapted the load detect logic to atomic and removed a pile of legacy structures no longer needed in preparation of next steps. Matt continued to work on atomic display fifo watermark updates. Another area that has seen a <b>lot of work in the background is runtime PM</b>. Mika, Imre and Ville have massively improved the debugging infrastructure we have to track down rare bugs in our power status tracking code. They also merged lots of fixes in this area. Unfortunately we're not yet at a point where we can enable overall runtime PM for the device by default.



More on the feature side is <b>pixel clock limit checks </b>for all outputs and platforms from Mika Kahola. This is related to the work to also enable dynamic display clock scaling on Gen9, but that part is still being worked on. Ville worked a lot on how <b>offsets and alignment are handled for display planes</b>, all in preparation to support rotated multi-planar formats like NV12. Again a feature where a lot of hard work is required to make the final patch to enable it all look really simple.



On the plain hardware and platform enabling side Jani implemented support for version 3 of the <b>VBT DSI descriptions</b>, which should extend DSI panel support to all current hardware. Which includes the Surface 3.



Finally on the <b>GEM side there have been mostly small fixes</b> and imrpovements under the hood. Tvrtko decoupled the internal engine representation from the userspace ABI defines. He also restructed the CS irq handler code, and started to fix up some locking issues in execlist. Chris tracked down some coherency issues in the execlist interrupt handler. Dave Gordon finally started to somewhat untangle the execlist initialization logic and some of the confusion in how all the different software structures connect.



One real feature work by Alex Dai though was enabled <b>ADS for GuC</b>, which is some means to hand additional metadata to the GuC firmware scheduler. But since GuC based command submission isn't enabled yet, this doesn't have a direct impact.



And of course there have been bugfixes all over the place, as usual.