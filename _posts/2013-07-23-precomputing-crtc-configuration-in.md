---
layout: post
title: Precomputing the CRTC Configuration in drm/i915
date: '2013-07-23T00:03:00.000-07:00'
author: sima
tags:
- In-Depth Tech
modified_time: '2013-07-23T01:08:07.778-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-4481518305873487209
blogger_orig_url: http://blog.ffwll.ch/2013/07/precomputing-crtc-configuration-in.html
---


Like I've briefly mentioned when explaining the [new modeset
code](/2012/08/new-modeset-code.html) merged into [kernel 3.7](/2012/10/neat-drmi915-stuff-for-37.html) one of the
goals was to facility atomic modesetting and fastboot. Now the big new
requirement for both atomic modesetting and fastboot is that we precompute the
entire output configuration before we start touching the hardware. And this
includes things like pll settings, internal link configuration and watermarks.
For atomic modesetting we need this to be able to decide up-front whether a
configuration requested by userspace works, or whether we don't have enough
bandwidth, display plls or lack some other resource. For fastboot we need to be
able to compute and track the full display hardware state in software so that we
can compare a requested configuration from userspace with the boot-up state
taken over from the BIOS. Otherwise we might end up with suboptimal settings
inherited from the firmware or worse, we'd try to reuse some resources still in
use by the firmware configuration (like when the BIOS and the linux driver would
pick different display plls). 

Now the new modeset code merged into 3.7 already added such state tracking and
precomputation for the output routing between crtcs and connectors. Over the
past months, starting with the just released 3.10 linux kernel we've added new
infrastructure and tons of code to track the low-level hardware state like
display pll selection, FDI link configuration and tons of other things. Read on
below for an overview of how the new magic works. 

<!--more-->

## Pipe Configuration Computation


So let's first look at how the new <code>struct intel_crtc_config</code> is used in the modeset code. The overall logic is similar to how the output routing works: First we create a new pipe configuration in a staging area. Once we have done that and know that the configuration requested by userspace is possible we shut down any pipes and ports that need to be changed or disabled. Then we commit the staged pipe configuration by writing it into the <code>struct intel_crtc</code>, this is done at the same time when we commit all the staged output routing pointers (and other modeset state like dpms) to the curretn state pointers. With that everything is in place to set the new mode into the hardware and enable the new pipes and ports again. 

The difference is that since the amount of data we have to keep track of is just a pointer to the entire pipe configuration per crtc we explicitly pass it around in the preparation phase. This is different from the output routing where every object has both routing pointers for both the current state and a second set of pointers with the <code>new_</code> prefix for the staged routing state. 

Currently our code doesn't yet support atomic modesets, so we only ever update the configuration of one display pipe. In the prepare phase we allocate a new pipe configuration structure and then fill it out with the desired state. The code generally calls functions and callbacks involved in this <code>compute_config</code>. The first step is to prefill it with information that encoder <code>-&gt;compute_config</code> callback needs. This includes the requested mode from userspace, a copy of the requested mode called adjusted mode which encoders can change for e.g. upscaling to a built in panel. Other state prefilled is the pipe bpp value, which is computed from the pixel depth of the main framebuffer. There's also some code to handle default values for the port clock, pixel multiplier and similar things which only a few encoders need to change. 

The next step is to loop over all encoders which will use the pipe (using the staged output routing pointers). Typically encoders can change the adjusted mode and set the desired panel fitter state (for eDP and LVDS). They adjust clock selection parameters like the pixel multiplier or the required port clock, e.g. the fixed DP lane clocks or 12 bpc HDMI output which needs a 1.5x faster port clock than the actual pixel clock. Encoders also set various state only used internally like whether the port is on the PCH (and so needs an FDI link). In some special cases encoders also set the display pll state or at least some values to steer the pll computation algorithms later on. 

Eventually the goal is that all the pll state computation happens in the prepare phase before we start to touch the hardware. This is important for atomic modesets where we want to know upfront whether enough plls are available for a given configuration. But right now the code transformation is still in progress and the pll selection happens still later on in the actual modeset phase, and only for a few case do we already precompute the desired pll configuration even. 

Once the encoders have had their say the last stage of the pipe configuration computation deals with general pipe state. This includes things like for example the FDI link configuration or in the future the pll state computation. This is done in <code>intel_crtc_compute_config</code>. This is also the function which checks for global constraints (like shared FDI links or eventually sufficient display plls). If the current configuration doesn't work out, but it might be possible with some reduced settings then this function returns <code>RETRY</code>. The overall pipe configuration computation function <code> intel_modeset_pipe_config</code> then restarts at the second phase where encoder callbacks are called. There are also some special flags in the pipe configuration structure to make sure encoders adhere to these limits. E.g. HDMI only supports either 8 bpc or 12 bpc modes and tries to pick the port bpc setting by rounding up. Now if we have a bandwidth constraint and the crtc logic limited a 12 bpc pipe to 10 bpc then we would loop forever since we would flip-flop between 12 bpc and 10 bpc. Hence why we need a flag for this case which the HDMI encoder's <code>compute_config</code> callback checks. 

The eventual goal is to use the pipe configuration structure to decide which pipes need to be changed. So the logic will be that we compute the pipe configuration for all pipes that will be enabled after the modeset. Once that's done we can compare the new pipe state with the current configuration and only disable and re-enable pipes which have actually changed. This will be useful for atomic modeset where userspace might only request a change in one crtc, but due to global limits we also need to change a second crtc. It should also allow us to have robust fastboot support as long as the pipe configuration can accurately reflect any hardware state left behind by the BIOS. 


## Configuration State Read-Out and Cross-Checking


But of course that requires us to read out the pipe configuration state from the hardware when taking over from the BIOS. And once we have that state read-out code it's really good to reuse it to cross-check the hardware state with the software tracking after each modeset. For the output routing this helped to uncover tons of bugs, both in the state tracking in software and the hardware read-out code. For the pipe configuration this was even more the case, mostly due to the much higher complexity of the pipe state. 

Reading out the pipe configuration state is a bit tricky since a bunch of the settings stored in the <code>struct intel_crtc_configuration</code> are associated with the output port, at least on some platforms. One example is the pixel multiplier (on the few old platforms where that's stored in the port), which in turn affects how the adjusted mode's dot clock is computed from the display pll state. To solve this conundrum we have a bunch of callbacks currently which need to be used in a specific order. Unfortunately we can't share that code between the hardware state readout logic run at boot-up and resume time and the modeset hardware cross check logic. There's also been the oddball bug because the sequence didn't quite match up at first. I expect that in the future we'll need to refactor this part of the pipe configuration infrastructure and move the entire logic into per-platform callbacks. 

But for now we first call the per-platform <code>-&gt;get_pipe_config</code> callback, which reads out all the state associated with planes, pipes and transcoders. Then we call the <code>-&gt;get_config</code> callback of each encoder connected to a given pipe. This currently fills in the sync polarity mode flags and pixel multiplier. Finally we call <code>-&gt;get_clock</code> to compute the actual dot clock of the adjusted mode from the read-out values. 

Since we don't read out the pipe state when the pipe is disabled there has been no need yet to sanitize it beyond what the existing code already does by simple disable crtcs which aren't in use. We've stumbled though already over cases where the hardware readout can't accurately reflect state, either due to hardware bugs or since our software tracking is less flexible than what the hardware can do (and what BIOS try to use). For those cases we set special quirk flags in the read-out pipe configuration so that the state comparison code can act accordingly. 

Comparing state configurations is done in <code>intel_pipe_config_compare</code>. Currently this is only used for the state cross check code run after every modeset. But eventually we want to use this code to decide whether a crtc needs to go through a full modeset sequence or whether we can optimize things with a special fastpath. This is important for fastboot support where e.g. disabling the panel fitter doesn't require a complete shutdown of the pipe. Quirk flags will also play an important role for fastboot support: If e.g. the panel fitters aren't associated to the pipes we expect them to be used with then we can set a corresponding quirk flag on all affected pipes. This will ensure that the configuration comparison fails and we do a full modeset at boot-up. 

In the future we might need to abstract away the configuration comparison with a per-platform callback to properly encode platform-specific constraints for fastpaths. But since the current experimental fastboot code doesn't explicitly deal with pipe configuration structures yet there's not yet a need for more complexity. 


## Handling Shared Display PLLs


One really special piece of pipe configuration state is handling display plls. The reason for that is that they can be shared between different pipes. This is unlike other global resources like panel fitters where sharing isn't possible, or reference clocks where sharing is the default but there's also no way to have conflicts. 

Now for display plls on the PCH we've had a bit of code to reference count and manage them. But unfortunately it took us a lot of iterations to have anything remotely working and we still ended up with tons of bugs. And since other platforms gear up for pll sharing too I've decided that more abstraction, which will afford us much more paranoia through systematically cross-checking state is required. 

Therefore we've added new objects to track the state of shared display plls in kernel version 3.11. Callbacks to read out the pll state and to enable/disable it provide the required abstraction between the generic refcounting and cross-checking code and the platform specific hardware state. The cross-checking not only compares whether the pll state is what we expect it to be, but also checks that the relationship between plls and crtcs is sound. For example we cross-check the reference count for all active crtcs using a given pll. 

Like when developing the new modeset infrastructure or when implementing the pipe configuration computation code the cross-checking helped a lot in catching bugs. And it already caught a sloppy firmware implementation which left an unused but enabled pll behind, so now we also have a bit of code to sanitize the pll state. 


## Conclusion


Overall the drm/i915 driver gained quite a bit of fanciness with the addition of the pipe configuration tracking - the new infrastructure already allows us to pull off a few neat new tricks. But there's still a lot to do to fully reap the benefits for atomic modesets and solid fastbooting. One thing is clear though: The paranoid hardware cross check code really helps to catch oddball bugs and gives us a much better reassurance that the state tracking is correct than any amount of laborious QA managed to do beforehand. Which means that development pace can greatly speed up since we depend far less upon the big linux community to test our code in the upstream kernel. 
