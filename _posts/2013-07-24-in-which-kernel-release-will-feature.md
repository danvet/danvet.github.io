---
layout: post
title: In Which Kernel Release Will $FEATURE Land In?
date: '2013-07-24T14:43:00.000-07:00'
author: sima
tags:
- Maintainer-Stuff
modified_time: '2013-07-24T14:43:41.802-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-8642996882842482489
blogger_orig_url: http://blog.ffwll.ch/2013/07/in-which-kernel-release-will-feature.html
---

So I get asked this question quite often and since I'm a terribly lazy person
I'm just going to write this down once and then link to it ...

<!--more-->

So the first thing to know is when future kernel releases will happen. In the
past few years this has settled to a new version roughly every 2½ months, so
this is very predictable. <a
href="http://www.kroah.com/log/blog/2013/07/01/3-dot-10-kernel-development-rate/">Greg
KH keeps some nice statistics</a> about that around. Then the upstream feature
merge window is the (usually) two week period between the first -rc of a new
version and the release of the previous kernel version. In that time all the
subsystem trees get pulled into Linus' upstream branch, so also Dave Airlie's
drm-next tree.

But by the time the upstream merge window opens up subsystem trees should all be
ready, so for drm/i915 driver features (and core drm features) the cut-off date
is earlier: Taking into account the one week delay incurred by our QA team doing
extensive manual testing of my feature pull requests before I send them off to
Dave Airlie and 1-2 weeks of buffer time due to misalignment of our biweekly
drm-intel-next iterations with the upstream merge window our own cut-off is
roughly two weeks before that.

Now this cut-off date is for patches which are ready for inclusion. Which means
they're reviewed, have sufficient coverage through automated tests and survived
a few days of getting beating up in my queue and our QA's nightly test runs.
Even when the patches and test-cases don't need big reworks to be suitable for
merging but only smaller fixups this can easily add 1-2 weeks since review
bandwidth is still in constant short supply. For big&amp;tricky feature work
getting the first round of a patch series into shape can take much longer, and
the time&amp;effort required to do so needs to be individually estimated.

Note that for early hardware enabling we can cut down a few of the intermediate
steps. But only as long as the new hardware support just plugs into existing
code and so has a much reduced risk of breaking existing support. Code
refactoring to prepare for a new platform must go through the usual merge and QA
process to catch regressions in time.

A somewhat related topic to fast&amp;loose for new hardware enabling is
regressions: Upstream is crystal clear that breaking existing setups is never an
option on the table, as long as someone reports the breakage at least. For early
hardware enabling that gives us a bit more leeway since no one yet has the
hardware, but this is deceiving since the development tree is months ahead of
when users actually get released kernels, and widespread availability of SDVs
for OEMs, ISVs and OSVs is surprisingly early. So the only safe place to break
things is just the first kernel released with the basic enabling of a new
platform. And even for that it should not regress compared to using the firmware
modeset code with either the efifb or vesafb drivers.

With that here's the recipe for reliably predicting when a big feature will land
in the kernel:

1. Create an early patch series as an rfc to gather requirements for the
real solution and assess a sensible amount of automated test coverage. Usually
this ends up being the version to throw away (or at least needs to be
substantially reworked).
2. Carefully split up all the work into sufficiently small junks both for easier
reviewing (past 20 patches or so the chore tends to be much worse) and for more
accurate estimation. Anything which takes more than a week to stitch&nbsp;
together is likely way too big.
3. Estimate both developer effort and real world time - regression, important
bugfixes and review duty tend to interfere. And especially for bigger patch
series or complicated subject, even more so in the drm core, review and testing
bandwidth tends to be the limiting factor.
4. Add a risk buffer, both for review delays, new requirements that suddenly pop
up and for tracking down regressions - those just happen. Of course different
parts have different risks associated: A simple interface refactoring should
have almost no risk, whereas adding new features and even more so new userspace
interfaces has much more stringent test and review standards.
5. Take the estimated completion date, i.e. when a feature will have been merged
to drm-intel-next. Then add the above mentioned delays and line that up with the
predicted kernel release dates to figure out in which upstream release something
will land.

In my opinion that's all that's needed for solid kernel feature planning and
lining it up with upstream releases.

For a little example presume that 3.x.0 was just released, and after the initial
review we estimate that we need 5 weeks to beat the code into shape and write
testcases plus 2 weeks trailing at the end to merge everything. Of course
merging would start earlier, so the 2 weeks is just to get the last parts in.
This means the feature is expected to be merged into drm-intel trees by the time
3.(x+1)-rc6 is tagged which for the usual release cycle is just on the edge to
to get into the 3.(x+2) merge window. But since we can't change the whims of
upstream deciding to releasing a week earlier it could very well get delayed to
the 3.(x+3) kernel release. So overall from the point where the in-depth scoping
is done for this example feature to the date where it would ship in a released
kernel could take 7½ months.
