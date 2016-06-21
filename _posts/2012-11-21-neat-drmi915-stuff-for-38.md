---
layout: post
title: Neat drm/i915 stuff for 3.8
date: '2012-11-21T08:34:00.001-08:00'
author: danvet
tags:
- Kernel RelNotes
modified_time: '2013-07-21T06:50:38.540-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-2719478283665178832
blogger_orig_url: http://blog.ffwll.ch/2012/11/neat-drmi915-stuff-for-38.html
---


So <a href="http://blog.ffwll.ch/2012/10/neat-drmi915-stuff-for-37.html">kernel 3.7</a> hasn't even shipped yet, but we're already lining up all the ducks for 3.8. And since feature wise I don't expect anything massive any more on top (since the feature merge period will close rsn) I've figured I might do the overview as well a bit earlier: 
<a name='more'></a>
 Now really big part is that <b>Haswell display support</b> is finally in decent shape - big kudos to Paulo Zanoni for this awesome work. As I've said already in my 3.7 overview, one of the reasons for the reworked modeset code is the Haswell display block, specifically how the new DP DDI encoders work. Paulo has split up the major parts into three blocks: First laying the ground with the basic DP code, then using that infrastructure to enable eDP panels. The tricky part was getting external displayport connectors to work, since on earlier platforms we have two independant encoders for DP and HDMI modes each, but now on Haswell the encoding part is actually shared. Which required some decent code-reorganization in our DP/HDMI encoder code to extract the output-specific functions into DP/HDMI-specific connectors, and then merge the DP and HDMI encoders into a <b>unified DDI encoder for Haswell</b>. Earlier platforms still have a separate encoder for DP and HDMI. 
 
 But that's not all for Haswell: Paulo also beat the <b>Haswell VGA</b> into shape - we've used a few clever tricks to get the initial support going, but that didn't scale. So again we've separate the Haswell code from older generations and implemented the required tricks. All these changes are because the display pipe on Haswell is much more flexible than on earlier generations: We have a display pipe on the cpu chip, a transcoder on the cpu chip to route the data to the different DDI ports (which work as universal display ports capable of DP, HDMI, but also FDI transfers to feed the PCH display pipe). And finally the transcoder on the PCH. On Haswell we can route the display data rather flexibly between all these differents parts, whereas on e.g. Ivybridge we've only had 3 of each (for the 3 pipes supported in total), with a fixed linking between them. 
 
There are also quite some small bits here&there to fix up oversights in our Haswell support where reused code from older generations isn't correctly updated yet. And Haswell has a very nice feature to help in that: The hardware can tell us when a register write didn't arrive anywhere since no block claimed that specific range. Damien Lespiau fixed an aweful lot of these cases, where code from previous generations was still used on Haswell, but wasn't updated for the new registers in all cases. Also on the topic of new hw enabling is <b>DP support for Valleyview</b> from Vijay Purushothaman et al.  
 
Another big area is panel and backlight support. Jani Nikula added some code for a <b>unified panel handling</b> between LVDS and eDP - this should help nicely for enabling some nice features in the future. Some, like connector properties to control the panel fitter scaling for eDP are already implemented thanks to this unified code. For eDP we're now also trying really hard to set up the panel without the BIOS' help - in some cases (unfortunately not all of them) this should fix black screen problems.  
 
Again to prepare for future enabling Ben Widawsky <b>killed the agp layer</b> for gen6+. This is a horrible in-between layer dating back to the old user-modesetting driver, and isn't really required on modern platforms any more. But we have to keep it around for backwards-compatibility (breaking userspace ABI in the kernel is verboten). But at least for new chips we don't need to jump through these hoops any more, which will greatly simplify some of the upcoming features in this area. 
 
 In 3.8 we'll also have a slightly <b>improved DP helper code</b> in the drm core, with some common sink handling code extracted from i915.ko and radeon.ko. I'm planning to extract much more common code in this area, especially the DP aux transfer code is practically screaming for this. 
 
 And as usual, tons of bugfixes, workarounds and general code improvements all over the place!  

 <b>Update:</b> I've forgotten a rather nice feature implemented by Rob Clarke and Imre Deak in the drm core: Pageflip completion and vblank timestamps are now sampled with CLOCK_MONOTONIC, finally making our buffer swap support OML_sync compliant. But the real deal is that vblank-driven rendereres like Wayland now no longer stall for a few frames when the realtime clock jumps!  
 
 <b>Update 2:</b> Another feature I've missed is the <b>secure batchbuffer support</b> from Chris Wilson, which can be used by the X driver to implement hopefully working vsync blits support without a compositor on Sandybridge and Ivybridge. So Xv on plain X should now no longer tear with a recent X driver version. 
