---
layout: post
title: Atomic Modeset Support for KMS Drivers
date: '2014-11-02T10:49:00.002-08:00'
author: danvet
tags:
- In-Depth Tech
modified_time: '2014-11-03T14:23:30.264-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-8822981517130755067
blogger_orig_url: http://blog.ffwll.ch/2014/11/atomic-modeset-support-for-kms-drivers.html
---


So I've just reposted my <a href="http://article.gmane.org/gmane.comp.video.dri.devel/117376">atomic modeset helper series</a>, and since the main goal of all that work was to ensure a smooth and simple transition for existing drivers to the promised atomic land it's time to elaborate a bit. The big problem is that the existing helper libraries and callbacks to driver backends don't really fit the new semantics, so some shuffling was required to avoid long-term pain. So if you are a driver writer and just interested in the details then read for what needs to be done to support atomic modeset updates using these new helper libraries.
<!--more-->

### Phase 1: Reworking the Driver Backend Functions for Planes


The first phase is reworking the driver backend callbacks to fit the new world. There are two big mismatches between the new atomic semantics and legacy ioctl interfaces:
<ul><li>The primary plane is no longer tied to the CRTC. Instead it is possible to enable the CRTC without any planes (resulting in a black screen) or only overlay planes. And the primary plane can be enabled/disabled and moved without changing the mode (of course only if the hardware actually supports it). But the existing CRTC helper library used to implement modesets only provides the single <code>crtc-&gt;mode_set</code> driver callback which always implicitly enables the primary plane, too.</li><li>Atomic updates of multiple planes isn't supported at all. And worse the code to check whether a plane update will work out is smashed into the same callback that does the actual plane update, defeating the check/commit distinction used in atomic interfaces.</li></ul>Both issues are addressed by adding new driver backend callbacks. Furthermore a few transitional helper functions are provided to implement the legacy entry points in terms of these new callbacks. That way the driver backend can be reworked without the additional hassle of needing to deal with all the atomic state object handling and check/commit semantics.



The first step is to rework the <code>-&gt;disable/update_plane</code> hooks using the transitional helper implementations <code>drm_plane_helper_update/disable</code>. These need the following new driver callbacks:

<ul><li><code>-&gt;atomic_check</code> for both CRTCs and planes. This isn't strictly required, but any state checks implemented in the current <code>-&gt;update_plane</code> hook must be moved into the plane's -&gt;atomic_check callback. The CRTC's callback will likely be empty for now.</li><li><code>-&gt;atomic_begin</code> and <code>-&gt;atomic_flush</code> CRTC callbacks. These wrap the actual plane update and should do per-CRTC work like preparing to send out the flip completion event. Or ensure that the plane updates are actually done atomically by e.g. setting/clearing GO bits or latching the update through some other means. Or if the hardware does not provide any support for synchronized updates, use vblank evasion to ensure all updates happen on the same frame.</li><li><code>-&gt;prepare_fb</code> and <code>-&gt;cleanup_fb</code> hooks are also optional. These are used to setup the framebuffers, e.g. pin their backing storage into memory and set up any needed hardware resources. The important part is that besides the <code>-&gt;atomic_check</code> callbacks <code>-&gt;prepare_fb</code> is the only new callback which is allowed to fail. This is important to make asynchronous commits of atomic state updates work. The helper library guarantees that for any successful call of <code>->prepare_fb</code> it will call <code>->cleanup_fb</code> - even when something else fails in the atomic update.</li><li>Finally there's <code>-&gt;atomic_update</code>. That's the function which does all the per-plane update, like setting up the new viewport or the new base address of the framebuffer for each plane.</li></ul>With this it's also easy to implement universal plane support directly, instead of with the default implementation which doesn't allow the primary plane to be disabled.  Universal planes are a requirement for atomic and need to be implemented in phase 1, but testing the primary plane support is also a good preparation for the next step: 
The new <code>crtc-&gt;mode_set_nofb</code> callback must be implement, which just updates the CRTC timings and data in the hardware without touching the primary plane state at all. The provided helpers functions <code>drm_helper_crtc_mode_set</code> and <code>drm_helper_crtc_mode_set_base</code> then implement the callbacks required by the CRTC helpers in terms of the new <code>-&gt;mode_set_nofb</code> callback and the above newly implemented plane helper callbacks.

### Phase 2: Wire up the Atomic State Object Scaffolding 


With the completion of phase 1 all the driver backend functions have been adapted to the new requirements of the atomic helper library. The goal of phase 2 is to get all the state object handling needed for atomic updates into place. There are three steps to that:
<ul><li>The first is fairly simply and consists in just wiring up all the state reset, duplicate and destroy functions for planes, CRTCs and connectors. Except for really crazy cases the default implementations from the atomic helper library should be good enough, at least to get started. With this there will always be an atomic state object stored in each object's <code>-&gt;state</code> pointer.</li><li>The second step is patching up the state objects in legacy code paths to make sure that we can partially transition to atomic updates. If your driver doesn't have any transition checks for plane updates (i.e. doesn't ever look at the old state to figure out whether an change is possible) then all you need to do is keep the framebuffer pointers and reference counts in balance with <code>drm_atomic_set_fb_for_plane</code>. The transitional helpers from phase 1 already do this, so usually the only place this is needs to be manually added is in the <code>-&gt;page_flip</code> callback.</li><li>Finally all <code>-&gt;mode_fixup</code> callbacks need to be audited to not depend upon any state which is only set in the various CRTC helper callbacks and not tracked by the atomic state objects. This isn't required for implementing the legacy interfaces using atomic updates, but this is important to correctly implement the check/commit semantics. Especially when the commit is done asynchronously. This really is a corner-case though, but any such code must be moved into <code>-&gt;atomic_check</code> hooks and rewritten in terms of the atomic state objects.</li></ul>

### Phase 3: Rolling out Atomic Support


With the driver backend changes from phase 1 and the state handling changes from phase 2 everything is ready for the step-by-step rollout of atomic support. Presuming nothing was missed this just consists of wiring up the -&gt;atomic_check and -&gt;atomic_commit implementations from the atomic helper library. And then replacing all the legacy entry pointers with the corresponding functions from the atomic helper library to implement them in terms of atomic.
The recommended order is to start with planes first, then test the -&gt;set_config functionality. Page flips and properties are best done later since they likely need some additional work:

<ul><li>The atomic helper library doesn't provide any default asynchronous commit support, since driver and hardware requirements seem to be too diverse. At least until we have a few proper implementations and can use them to extract a good set of helper functions. Hence drivers must implement basic async commit support using the building blocks provided (and other drm and kernel infrastructre like flip queues, fence callbacks and work items - hopefully soonish also vblank callbacks).</li><li>Property values need to be moved into the relevant state object first before the corresponding implementations can be wired up. As long as the driver doesn't yet support the full atomic ioctl this can be done at leisure, but must be completed before the ioctl can be enabled. To do so drivers need to subclass the relevant state structure and reimplement all the state handling functions rolled out in phase 2.</li></ul>
Besides these two complications (which might require a bit of work depending upon the driver) this is all that's needed for full atomic modeset and pageflip support.

### Follow-up Driver Cleanups


But there's of course quite a bit of cleanup work possible afterards!
The are some big differences between the old CRTC helper modeset logic and the new one (using the same callbacks, but completely rewritten otherwise) in the atomic helper library:

<ul><li>The encoder/bridge/CRTC enabling/disabling sequence for a given modeset configuration is now always the same. Which means unused CRTC won't be disabled any more only after everything else is set up, but together with all the other blocks before enabling anything from the new configuration. Also, when an output pipeline changes the helper library will always force a full modeset of the entire pipeline.

This reduces combinatorial complexity a lot and should especially help with shared resources (like PLLs) - no longer can a modeset spuriously fail just because the old CRTC hasn't released its PLL before the new one was enabled.</li><li>Thanks to the atomic state tracking the helper code won't lose track of the software state of each object any more. Which means disabled functions won't be disabled more than once. So all code in the driver-backend which checks the current state and acts accordingly can be flattened and replaced by WARNings.</li></ul>
These are all lessons learned from the <a href="http://blog.ffwll.ch/2012/08/new-modeset-code.html">i915 modeset rewrite</a>. The only thing missing in the atomic helpers compared to i915 is the <a href="http://blog.ffwll.ch/2013/07/precomputing-crtc-configuration-in.html">state readout and cross-checking support</a> - everything else is there. But even that can be easily implemented by adding hardware state readout callbacks and using them in the various state reset functions (to reconstruct matching software state) and also to cross-check state.



The other big cleanup task is to stop using all the legacy state variables and switch all the driver backend code to only look at the state object structures. The two big examples here are <code>crtc-&gt;mode</code> and the <code>plane-&gt;fb</code> pointer.

### So What Now?


With all that converting drivers should be simple and can be done with a series of not-too-invasive refactorings. But my patch series doesn't yet contain the actual atomic modeset ioctl. So what's left to be done in the drm core?
<ul><li>Per-plane locking is still missing - currently all plane-related changes just lock all CRTCs. Which is a bit too much, and in cases like the cursor plane or for page flips actually a regression compared to the legacy code paths.</li><li>The atomic ioctl uses properties for everything, even for the standard properties inherited from the legacy ioctls. All the code for parsing properties exists already in Rob Clark's patch series, but needs to be rebased and adpated to the slightly different interfaces this latest iteration of the internal atomic interface has.</li><li>The fbdev emulation needs to grow proper atomic check/commit support. This is both a good kernel-internal validation of the atomic interface and would finally allow us to get multi-pipe configuration for fbcon to work correctly. But before we can do this we need a driver with multiple CRTCs, shared resource constraints and proper atomic support to be able to even test this.</li><li>There's still some room for more helpers, for example pretty much all drivers have some sort of vblank driver callback and work item infrastructure. That's better done as a helper in the core vblank handling code.</li><li>And finally we need the actual ioctl code.</li></ul>So still a few things to do, besides adding atomic support to all drivers. 



<b>Update:</b> The explanation for how to implement state readout and cross checking was a bit confused, so I reworded that.
