---
layout: post
title: Update for Atomic Display Updates
date: '2015-01-28T09:18:00.000-08:00'
author: danvet
tags:
- In-Depth Tech
modified_time: '2015-01-28T09:18:55.513-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-9029048282758780689
blogger_orig_url: http://blog.ffwll.ch/2015/01/update-for-atomic-display-updates.html
---

Another kernel release is imminent and a lot of things happened since my last
big blog post about [atomic
modeset](/2014/11/atomic-modeset-support-for-kms-drivers.html). Read on for what
new shiny things 3.20 will bring this area.

<!--more-->

### Atomic IOCTL and Properties

The big thing for sure is that the actual atomic IOCTL from Ville has finally landed for 3.20. That together with all the work from Rob Clark to add all the new atomic properties to make it functional (there's no IOCTL structs for any standard values, everything is a property now) means userspace can finally start using atomic. Well, it's still hidden behind the experimental module option <code>drm.atomic=1</code> but otherwise it's all there. There's a few big differences compared to earlier iterations:

<ul><li>Core properties are now fully handled by the core, drivers can only decode their own properties. This should mean that the handling of standardized properties should be more uniform and also makes it easier to extend (e.g. with the standardized rotation property we already have) beyond the paramaters the legacy interfaces provided.</li><li>Atomic props&amp;ioctl are opt-in per file_priv, userspace needs to explicitly ask for it. This is the same idea as with universal plane support and will make sure that old userspace doesn't get confused and fall over when it would see all the new atomic properties. In the same spirit some of the legacy-only properties (like DPMS) will be rejected in the atomic IOCTL paths.</li><li>Atomic modesets are currently not possible since the exact ABI for how to handle the mode property is still under discussion. The big missing thing is handling blob properties, which will also be needed to upload gamma table updates through the atomic interface.</li></ul>Another missing piece that's now resolved is the DPMS support. On a high level this greatly reduces the complexity of the legacy DPMS settings into a&nbsp; simple per-CRTC boolean. Contemporary userspace really wants nothing more and anything but a simple on/off is the only thing that current hardware can support. Furthermore all the bookkeeping is done in the helpers, which call down into drivers to enable or disable entire display pipelines as needed. Which means that drivers can rip out tons of code for the legacy DPMS support by just wiring up <code>drm_atomic_helper_connector_dpms</code>.

### New Driver Conversions and Backend Hooks

Another big thing for 3.20 is that driver support has moved forward greatly: Tegra has now most of the basic bits ready, MSM is already converted. Both still lack conversion and testing for DPMS since that landed very late though. There's a lot of prep work for exynos, but unfortunately nothing yet towards atomic support. And i915 is in the process of being converted to the new structures and will have very preliminary atomic plane updates support in 3.20, hidden behind a module option for now.

And all that work resulted in piles of little fixes. Like making sure that the legacy cursor IOCTLs don't suddenly stall a lot more, breaking old userspace. But we also added a bunch more hooks for driver backends to simplifiy driver code:

<ul><li>For drivers there's often a very big difference between a plane update, and disabling a plane. And with atomic state objects it's a bit more work to figure out when exactly you're doing a on to off transition. Thierry Redding added a new optional <code>-&gt;atomic_plane_disable()</code> hook for drivers which will take care of all those disdinctions.</li><li>Mostly for hysterical raisins going back to the original Xrandr implementations for userspace mode setting the callbacks to enable and disable encoders and CRTCs used by the various helper libraries have really confusing names. And with the legacy helpers all kinds of strange semantics. Since the atomic helpers massively simplify things in this area it made a lot of sense to add a new set of <code>-&gt;enable()</code> and <code>-&gt;disable()</code> hooks, which are preferred if they're set. All the other hooks (namely <code>-&gt;prepare()</code>, <code>-&gt;commit()</code> and <code>-&gt;dpms()</code>) will eventually be deprecated for atomic drivers. Note that <code>-&gt;mode_set</code> is already superseeded by <code>-&gt;mode_set_nofb</code> due to the explicit primary plane handling with atomic updates.</li></ul>Finally driver conversions showed that vblank handling requirements imposed by the atomic helpers are a lot stricter than what userspace tends to cope with. i915 has recently reworked all its vblank handling and improved the core helpers with the addition of <code>drm_crtc_vblank_off()</code> and <code>drm_crtc_vblank_on()</code>. If these are used in the CRTC disable and enable hooks the vblank code will automatically reject vblank waits when the pipe is off. Which is the behaviour both the atomic helpers and the transitional helpers expect.

<br/>One complication is that on load the driver must ensure manually that the vblank state matches up with the hardware and atomic software state with a manual call to these functions. In the simple case where drivers reset everything to off (which is what the reset implementations provided by the atomic helpers presume) this just means calling <code>drm_crtc_vblank_off()</code> somewhen in the driver load code. For drivers that read out the actual hardware state they need to call either <code>_off()</code> or <code>_on()</code> matching on the actual display pipe status.

### Future Work

Of course there's still a few things left to do before atomic display updates can be rolled out to the masses. And a few things that would be rather nice to have, too:

<ul><li>Support for blob properties so that modesets and a bunch of other neat things can finally be done.</li><li>Testing, and lots of it, before we enable atomic by default.</li><li>Thanks to DP MST connectors can be hotplugged, and there are still a lot of lifetime issues surrounding connector handling all over the drm subsystem. Atomic display code is unfortunately no exception.</li><li>And finally try to make the vblank code a bit more generally useful and then implement generic async atomic commit on top of that.</li></ul>It's promising to keep interesting!
