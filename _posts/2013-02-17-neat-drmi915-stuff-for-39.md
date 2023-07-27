---
layout: post
title: Neat drm/i915 stuff for 3.9
date: '2013-02-17T11:01:00.002-08:00'
author: sima
tags:
- Kernel RelNotes
modified_time: '2013-07-21T06:50:08.046-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-4898738363561615597
blogger_orig_url: http://blog.ffwll.ch/2013/02/neat-drmi915-stuff-for-39.html
---


Now that [3.8](/2012/11/neat-drmi915-stuff-for-38.html) is winding down it's
time to look at what 3.9 will bring for the drm/i915 driver:

<!--more-->

Let's first look at bit at the drm core changes: The headline item this time
around is the
[reworked kernel modeset locking](/2013/02/new-kernel-modesetting-locking.html).
Finally the kernel doesn't stall for a few frames while probing outputs in the
background! The other core drm changes revolve around the fb helper code. Both
in the [i915 modeset rework](/2012/08/new-modeset-code.html) which landed in 3.7
and the locking rework I've noticed a few things to fix up and clarify. And as
the icing on the cake we now have rather nice
[kerneldoc](http://cgit.freedesktop.org/~airlied/linux/commit/?h=drm-next&id=207fd32970b1def91b11ae28f6bebffc792db714)
for the driver interfaces.  Within i915.ko itself we've started with merging
some of that fastboot patches from Chris Wilson, namely <b>stolen memory
support</b>. This is required to wrap the framebuffer allocated by the firmware.
This is one of the many pieces needed to avoid the initial modeset at boot-up if
the BIOS configuration is suitable, and so shaves off a bit of boottime. To make
suspend/resume quicker and more important flicker-free Jesse Barnes implemented
<b>vt-switchless resume</b> support. With that you won't see that annoying
console cursor blinking any more when resuming, but directly go to your desktop
(or probably screensaver, asking for your password). The core power management
and console pieces are merged, but the i915 enabling will likely only be merged
for 3.10 since a few bugs surfaced in the patch review. 

For general robustness of our GEM implementation we've clarified the various
<b>gpu reset</b> state transitions. This should prevent applications from
crashing while a gpu reset is going on due to the kernel leaking that transitory
state to userspace. Ville Syrjälä also started to fix up our handling of
<b>pageflips across gpu hangs</b> so that compositors no longer get stuck after
a reset. Unfortunately not all of his patches made it into 3.9. Somewhat related
is Mika Kuoppala's work to fix bugs across the <b>seqnqo wrap-around</b>. And to
make sure that those bugs won't pop up again he also added some testing
infrastructure. 

On the modeset feature front 3.9 has proper support for the <b>"RBG broadcast
range/reduced color range"</b> mode from Ville Syrjälä for HDMI/DP. Paulo Zanoni
implemented support for the manually-controlled <b>display power well on
Haswell</b> - this should reduce power consumption when only the internal eDP
panel is enabled. 

Driver-internally Ville <b>removed the IS_DISPLAYREG hack on Valleyview</b>
which was added to kickstart early enabling. The hardware engineers moved around
all the display related hardware registers which required us to add an offset.
But since some of those display registers conflict with GT registers the
original approach with the IS_DISPLAYREG check in the read/write functions
became didn't work. And people often forgot to add new registers to that table,
making it a big maintenance hassle. But now all this offset handling has been
moved to the register accessing functions/definitions themselves, which also
paves the way for future hardware enabling. 

On the GEM side of userspace-visible features the big item is the
<b>no-relocations and execbuffer LUT</b> extensions from Chris Wilson. As long
as buffers are not moved around between execbuffer calls, this should completely
slash the in-kernel relocation overhead. Unfortunately due to some bogons in the
libdrm design we can't implement support for the new mode for it. So for now
only the SNA acceleration backend from Chris has support for these new execbuf
modes. 

The other big refactoring in the driver is the <b>massive GTT cleanup</b> from
Ben Widawsky. On gen6 and later we now no longer rely on the fake intel-agp
driver but have moved all the code to i915.ko. Also, all the gtt and ppgtt
functions are now nicely abstracted behind a set of interface function tables.
This will greatly help for some of the anticipated work in the address space
handling, like real per-process address spaces. 
 
Finally there's the usual amount of small cleanups and bug fixes, though this time around there was also a major regression involved: The infamous <b>ilk/gen4 relocation regression</b> which killed tons of machines when putting them under decent amounts of I/O and memory pressure. But we've [fixed that](https://bugs.freedesktop.org/show_bug.cgi?id=55984), too. And as an upside the shrinker handling should now be a notch more robust, so you can drive your systems even harder against the memory limit wall until you unleash the wrath of the OOM gods ... 
