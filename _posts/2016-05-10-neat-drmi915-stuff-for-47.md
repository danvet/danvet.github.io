---
layout: post
title: Neat drm/i915 Stuff for 4.7
date: '2016-05-10T14:52:00.001-07:00'
author: danvet
tags: 
modified_time: '2016-05-11T08:02:31.939-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-3142001204592303032
blogger_orig_url: http://blog.ffwll.ch/2016/05/neat-drmi915-stuff-for-47.html
---

The <a href="http://blog.ffwll.ch/2016/03/neat-drmi915-stuff-for-46.html">4.6 release</a> is almost out of the door, it's time to look at what's in store for 4.7.

<a name='more'></a>Let's first look at the epic saga called atomic support. In 4.7 the <b>atomic watermark update support for Ironlake through Broadwell</b> from Matt Roper, Ville Syrjälä and others finally landed. This took about 3 attempts to get merged because there's lots of small little corner cases that caused regressions each time around, but it's finally done. And it's an absolutely key piece for atomic support, since Intel hardware does not support atomic updates of the watermark settings for the display fetch fifos. And if those values are wrong tearings and other ugly things will result. We still need corresponding support for other platforms, but this is a really big step. But that's not the only atomic work: Maarten Lankhorst made the <b>hardware state checker atomic</b>, and there's been tons of smaller things all over to move the driver towards the shiny new.



Another big feature on the display side is <b>color management</b>, implemented by Lionel Landwerlin, and then fixes to make it fully <b>atomic</b> from Maarten. Color management aims for more accurate reproduction of a well definied color space on panels, using a de-gamma table, then a color matrix, and finally a gamma table.



For platform enabling the big thing is support for <b>DSI panels on Broxton</b> from Jani Nikula and Ramalingam C. One fallout from this effort is the <b>cleaned up VBT parsing </b>code, done by Jani. There's now a clean split between parsing the different VBT versions on all the various platforms, now neatly consolidated, and using that information in different places within the driver. Ville also hooked up <b>upscaling/panel fitting for DSI panels</b> on all platforms.



Looking more at driver internals Ander Conselvan de Oliviera<b> </b>and Ville <b>refactored the entire display PLL </b>code on all platforms, with the goal to reuse it in the DP detection code for upfront link training. This is needed to detect the link configuration in certain situations like USB type C connectors. Shubhangi Shrivastava <b>reworked the DP detection</b> code itself, again to prep for these features. Still on pure display topics Ville <b>fixed lots of underrun issues</b> to appease our CI on lots of platforms. Together with the atomic watermark updates this should shut up one of the largest sources of noise in our test results.



Moving on to power management work the big thing is <b>lots of small fixes for the runtime PM support</b> all over the place from Imre Deak and Ville, with a big focus on the Broxton platform. And while we talk features affecting the entire driver: Imre<b> added fault injection to the driver load paths</b> so that we can start to exercise all that code in an automated way.



Finally looking at the render/GEM side of the driver the short summary is that Tvrtko Ursulin and Chris Wilson worked the code all over the place: A cleanup up and <b>tuned forcewake handling </b>code from Tvrtko, f<b>ixes for more userptr corner cases </b>from Chris, a new <b>notifier to handle vmap exhaustion</b> and assorted polish in the related shrinker code, <b>cleaned up and fixed handling of gpu reset corner cases</b>, <b>fixes for context related hard hangs</b> on Sandybridge and Ironlake, large-scale renaming of parameters and structures to realign old code with the newish execlist hardware mode, the list goes on. And finally a rather big piece, and one which causes some trouble, is all the work to <b>speed up the execlist code</b>, with a big focusing on <b>reducing interrupt handling overhead</b>. This was done by moving the expensive parts of execlist interrupt handling into a tasklet. Unfortunately that uncovered some bugs in our interrupt handling on Braswell, so Ville jumped in and fixed it all up, plus of course removed some cruft and applied some nice polish.



Other work in the GT are are <b>gpu hang fixes for Skylake GT3 and GT4</b> configurations from Mika Kuoppala. Mika also provided patches to i<b>mprove the edram handling</b> on those same chips. Alex Dai and Dave Gordon kept working on making GuC ready for prime time, but not yet there. And Peter Antoine improved the MOCS support to work on all engines.



And of course there's been tons of smaller improvements, bugfixes, cleanups and refactorings all over the place, as usual.