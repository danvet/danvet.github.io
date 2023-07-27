---
layout: post
title: Overclocking your Intel GPU on Linux
date: '2013-03-24T08:59:00.000-07:00'
author: sima
tags:
- In-Depth Tech
modified_time: '2013-07-21T07:00:33.241-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-3197309094902219529
blogger_orig_url: http://blog.ffwll.ch/2013/03/overclocking-your-intel-gpu-on-linux.html
---

First things first: If you damage your hardware and burn down your house, it's
not my problem. You've been warned!

So after a bit of irc discusssions yesterday it turns out that you can overclock
intel gpus by quite a margin. Which makes some sense now that the gfx
performance of intel chips isn't something to be completely ashamed of.

<!--more-->

You need a few ingredients:

- An Sandybridge/Ivybridge Intel GPU (Intel HD 2000/3000/2500/4000 in marketing
  speak).
- A "enthusiast" motherboard which allows you to set gpu overclocking settings
  (higher frequency and voltage).
- Lastet drm-intel-nightly kernel branch from <a
  href="http://cgit.freedesktop.org/~sima/drm-intel">drm-intel git</a> or at
  least a kernel built with <a
  href="https://patchwork.kernel.org/patch/2305081/">this patch</a> from Ben
  Widawsky. The patch should apply to pretty much any recent stable kernel.
  Without that patch gpu turbo support is broken. 
- Preferably a desktop gpu with a big cooling rig. Overclocking on modern intel
  chips essentially just increases the turbo headroom into the unvalidated range
  (with the potential for hangs and corruptions in rendering). And if your
  cooling system can't keep up with the increased heat output the on-chip
  controller will quickly clamp your clocks down to the non-turbo frequency.
- Your favorite gpu benchmark. The reason for that is the thermal throttling
  when running in the turbo range: It happens behind the driver's back and the
  only observable effect is a slower gpu - the current frequency value in sysfs
  is still the same!
- To check whether it all works out you can boot with `drm.debug=0xe` appended
  to your kernel cmdline and check dmesg for the gpu overclocking support debug
  message:

		[98650.411179] [drm:gen6_enable_rps], overclocking supported, adjusting frequency max from 1300MHz to 1300MHz

Yeah, my system doesn't really support overclocking :(

Or check in sysfs in `/sys/class/drm/card0` the various `gt_*_freq_mhz` files.
You can also use those to limit the upper clock, which is useful for figuring
out at which frequency your gpu is still stable at for a given voltage setting.

Limiting the lower clocks is usually not interesting, since it will only affect
how much power the hardware can safe under intermediate loads. And you want to
aggressively reduce heat output to better use the turbo range when needed.

If you play around with this please drop a comment with your results - I only
have boring Intel developer board's around here which don't support
overclocking. So no fun fore me. But freezer on #intel-gfx managed to overclock
his desktop Ivybridge from 1.1GHz to 1.6GHz with only a small voltage increase,
and fps in CS:S increased quite a bit due to that.

Happy overclocking and benchmarking

**Update:** In debugfs you can check the CAGF (current actual gt frequency) field
in the `i915_cur_delayinfo` file for the real frequency, including any effects
due to thermal throttling. 

**Update 2:** Latest kernels (and so also 3.10) will limit the gpu frequency at
boot-up to the non-overclocked range. This should prevent crashes while booting,
but it also means that you need to manually set the overclocked frequency limit.
