---
layout: post
title: "Upstream Graphics: Too Little, Too Late"
tags:
- Maintainer-Stuff
- Conferences
---
Unlike the tradition of my past few talks at Linux Plumbers or Kernel
conferences this time around in Lisboa I did not start out with a rant proposing
to change everything. Instead I celebrated roughly 10 years of upstream graphics
progress and [finally achieving paradise, like in my ELCE
talk](/2019/12/elce-lyon-everything-great.html). But that was all just prelude
to a few bait-and-switches later fulfill expectations on what's broken this time
around in upstream, totally, and what needs to be fixed and changed, maybe.

The [LPC video recording](https://www.youtube.com/watch?v=S1I34t5RpnI) is now
released, [slides](/slides/lpc-2019-upstream.pdf) are uploaded. If neither of
that is to your taste, read below the break for the written summary.

<iframe width="560" height="315" src="https://www.youtube.com/embed/S1I34t5RpnI" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<!--more-->

## Mission Accomplished

10 or so years ago upstream graphics was essentially a proof of concept of the
promises to come. Kernel display modeset just landed, finally bringing a
somewhat modern display driver userspace API to linux. And GEM landed, bringing
proper GPU memory management and multi client rendering. Realistically a lot
needed to be done still, from rendering drivers for all the various SoC, to 
an atomic display API that can expose all the features, not just what was needed
to light up a linux desktop back in the days. And lots of work to improve the
codebase and make it much easier and quicker to write drivers.

There's obviously still a lot to do, but I think we've achieved that - for full
details, check out my [ELCE talk about everything great for upstream
graphics](/2019/12/elce-lyon-everything-great.html).

Now despite all this justified celebrating, there is one sticking point still:

## NVIDIA

The trouble with team green from an open source perspective - for them it's a
great boon - is that they own the GPU software stack in two crucial ways:

* NVIDIA defines how desktop GL works. Not so relevant anymore, and at least the
  core profile is a solid spec and has fully open source test suite from Khronos
  by now. But the compatibility profile, which didn't throw out all the legacy
  features from the GL1.x days in the 90s, does not have any of the interactions
  with all the new features specced out and covered with tests - NVIDIA's binary
  driver is that standard, and that since roughly 20 years.

* More relevant today is CUDA, not quite as long as desktop GL, but for a market
  that's growing at a rather brisk pace. CUDA is the undisputed king of the
  general purpose GPU compute hill. Anything and everything that matters runs on
  top of it, often exclusively.

Together these are a huge software moat around the high margin hardware
business. All an open stack would achieve is filling in that moat and inviting
competition to eat the nice surplus. In other words, stupid to even attempt,
vendor lock-in just pays too well.

Now of course the reverse engineered nouveau driver still exists. But if you
have to pay for reverse engineering already, then you might as well go with
someone else's hardware, since you're not going to get any of the CUDA/GL
goodies.

And the business case for open source drivers indeed exists so much that even
paying for reverse engineering a full stack is no problem. The result is a
vibrant community of hardware vendors, customers, distros and consulting shops
who pay the bills for all the open driver work that's being done. And in
userspace even "upstream first" works - releases happen quickly and often
enough, with sufficiently smooth merge process that having a vendor tree is
simply not needed. Plus customer's willingness to upgrade if necessary, because
it's usually a well-contained component to enable new hardware support.

Because without a solid business case behind open graphics drivers, they're just
not going to happen, viz. NVIDIA.

## Not Shipping Upstream

Unfortunately the business case for "upstream first" on the kernel side is
completely broken. Not for open source, and not for any fundamental reasons, but
simply because the kernel moves too slowly, is too big, drivers aren't well
contained enough and therefore customer will not or even can not upgrade. For
some hardware upstreaming early enough is possible, but graphics simply moves
too fast: By the time the upstreamed driver is actually in shipping distros,
it's already one generation behind. And missing almost a year of tuning and
performance improvements. Worse it's not just new hardware, but also GL and
Vulkan versions that won't work on older kernels due to missing features,
fragementing the ecosystem further.

This is entirely unlike the userspace side, where refactoring and code sharing
in a cross-vendor shared upstream project actually pays off. Even in the short
term.

There's a lot of approaches trying to paper over this rift with the linux
kernel:

* Stable kernel ABI for driver modules, so that you can upgrade the core kernel
  and drivers independently. Google Android is very much aiming this solution at
  their huge vendor tree problem. Traditionally enterprise distros do the same.
  This works, safe that stable kernel-internal ABI isn't a notion that's [very
  popular with kernel maintainers
  ...](https://www.kernel.org/doc/html/latest/process/stable-api-nonsense.html)

* If you go with an "upstream first" approach to shipping graphics drivers you
  first need to polish your driver, refactor out common components, and push it
  to upstream.  Only to then pay a second team to re-add all the crap so you can
  ship your driver on all the old kernels, where all the helpers and new common
  code don't exist.

* Pay your distro or OS vendor to just backport the new helpers before they even
  have landed in an upstream release. Which means instead of a backporting team
  for the driver on your payroll you now pay for backporting the entire
  subsystem - which in many cases is cheaper, but an even harder sell to
  beancounters. And sometimes not possible because other driver teams from
  competitors might not be on board.

Also, there just isn't a single LTS kernel. Even upstream has multiple, plus
every distro has their own flavour, plus customers love to grow their own
variety trees too. Often they're not even coordinated on the same upstream
release. Cheapest way to support this entire madness is to completely ignore
upstream and just write your own subsystem. Or at least not use any of the
helper libraries provided by kernel subsystems, completely defeating the
supposed benefit of upstreaming code.

No matter the strategy, they all boil down to paying twice - if you want to
upstream your code. And there's no added return for the doubled bill. In
conclusion, upstream first needs a business case, like an open source graphics
stack in general. And that business case is very much real, except for
upstreaming, it's only real in userspace.

In the kernel, "upstream first" is a sham, at least for graphics drivers.

Thanks to Alex Deucher for reading and commenting on drafts of this text.
