---
layout: post
title: Neat drm/i915 Stuff for 3.20
date: '2015-02-11T15:20:00.000-08:00'
author: sima
tags:
- Kernel RelNotes
modified_time: '2015-02-11T15:20:25.645-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-746164648252083417
blogger_orig_url: http://blog.ffwll.ch/2015/02/neat-drmi915-stuff-for-320.html
---

[Linux 3.19](/2014/12/neat-drmi915-stuff-for-319.html) was just
released and my usual overview of what the next merge window will bring is more
than overdue. The big thing overall is certainly all the work around [atomic
display updates](/2015/01/update-for-atomic-display-updates.html), but read on
for what else all has been done.

<!--more-->

Let's first start with all the driver internal rework to support atomic. The big
thing with atomic is that it requires a clean split between code that checks
display updates and the code that commits a new display state to the hardware.
The corallary from that is that any derived state that's computed in the
validation code and needed int the commit code must be stored somewhere in the
state object. Gustavo Padovan and Matt Roper have done all that work to
<b>support atomic plane updates</b>. This is the code that's now in 3.20 as a
tech preview. The big things missing for proper atomic plane updates is async
commit support (which has already landed for 3.21) and support to adjust
watermark settings on the fly. Patches for from Ville have been around for a
long time, but need to be rebased, reviewed and extended for recently added
platforms.

On the modeset side Ander Conselvan de Oliveira has done a lot of the necessary
work already. Unfortunately converting the modeset code is much more involved
for mainly two reaons: First there's a lot more derived state that needs to be
handled, and the existing code already has structures and code for this.
Conceptually the code has been prepared for an atomic world since the [big
display rewrite](/2012/08/new-modeset-code.html) and the introduction of [CRTC
configuration structures](/2013/07/precomputing-crtc-configuration-in.html). But
<b>converting the i915 modeset code to the new DRM atomic structures and
interface</b> is still a lot of work. Most of these refactorings have landed in
3.20. The other hold-up is shared resources and the software state to handle
that. This is mostly for handling display PLLs, but also other shared resources
like the display FIFO buffers. Patches to handle this are still in-flight.

Continuing with modeset work Jani Nikula has reworked the <b>DSI support to use
the shared DSI helpers</b> from the DRM core. Jani also reworked the DSI to in
preparation for <b>dual-link DSI support</b>, which Gaurav Singh implemented.
Rodrigo Vivi and others provided a lot of patches to <b>improve PSR support</b>
and enable it for Baytrail/Braswell. Unfortunately there's still issues with the
automated testcase and so PSR unfortunately stays disabled by default for now.
Rodrigo also wrote some nice DocBook <b>documentation for fbc</b>, another step
towards fully documenting i915 driver internals.

Moving on to platform enabling there has been a lot of work from Ville on
<b>Cherryview</b>: RPS/gpu turbo and pipe CRC support (used for automated
display testing) are both improved. On <b>Skylake</b> almost all the basic
enabling is merged now: PM patches, enabling mmio pageflips and fastboot support
from Damien have landed. Tvrtko Ursulin also create the infrastructure for
<b>global GTT views</b>. This will be used for some upcoming features on
Skylake. And to simplify enabling future platforms Chris Wilson and Mika
Kuoppala have completely <b>rewritten the forcewake domains handling</b> code.

Also really important for Skylake is that the <b>execlist support</b> for gen8+
command submission is finally in a good enough to be used by default - on
Skylake that's the only support path, legacy ring submission has been
deprecated. Around that feature and a few other closely related ones a lot of
code was refactoring: John Harrison implemented the <b>conversion from raw
sequence numbers to request objects</b> for gpu progress tracking. This as is
also prep work for landing a gpu scheduler. Nick Hoath removed the special
execlist request tracking structures, simplifying the code. The <b>engine
initialization code was also refactored</b> for a cleaner split between software
and hardware initialization, leading to robuster code for driver load and system
resume. Dave Gordon has also reworked the code tracking and computing the ring
free space. On top of that we've also <b>enabled full ppgtt</b> again, but for
now only where execlists are available since there are still issues with the
legacy ring-based pagetable loading.

For generic GEM work there's the really nice support for <b>write-combine cpu
memory mappings</b> from Akash Goel and Chris Wilson. On Atom SoC platforms
lacking the giant LLC bigger chips have this gives a really fast way to upload
textures. And even with LLC it's useful for uploading to scanout targets since
those are always uncached. But like the special-purpose uploads paths for LLC
platforms the cpu mmap views do not detile textures, hence special-purpose
fastpaths need to be written in mesa and other userspace to fully exploit this.
In other GEM features the shadow batch copying code for the command parser has
now also landed.

Finally there's the redesign from Imre Deak to <b>use the component framework
for the snd-hda/i915 interactions</b>. Modern platforms need a lot of
coordination between the graphics and sound driver side for audio over hdmi, and
thus far this was done with some ad-hoc dynamic symbol lookups. Which results in
a lot of headaches to get the ordering correctly for driver load or system
suspend and resume. With the component framework this depency is now explicit,
which means we will be able to get rid of a few hacks. It's also much easier to
extend for the future - new platforms tend to integrate different components
even more.
