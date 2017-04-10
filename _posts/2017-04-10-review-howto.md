---
layout: post
title: Review, not Rocket Science
author: danvet
tags:
- Maintainer-Stuff
---
About a week ago there where 2 articles on LWN, the first coverging [memory
management patch review](https://lwn.net/Articles/718212/) and the second
covering [the trouble with making review
happen](https://lwn.net/Articles/718411/). The take away from these two articles
seems to be that review is hard, there's a constant lack of capable and willing
reviewers, and this has been the state of review since forever. I'd like to
counter pose this with our experiences in the graphics subsystem, where we've
rolled out a well-working review process for the Intel driver, core subsystem
and now the co-maintained small driver efforts with success, and not all that
much pain.

> tl;dr: require review, no exceptions, but document your expectations

Aside: This is written with a kernel focus, from the point of view of a
maintainer or group of maintainers trying to establish review within their
subsystem. But the principles really work anywhere.

<!--more-->

## Require Review

When review doesn't happen, that generally means no one regards it as important
enough. You can try to improve the situation by highlighting review work more,
and giving less focus for top committer stats. But that only goes so far, in the
end when you want to make review happen, the one way to get there is to require
it.

Of course if that then results in massive screaming, then maybe you need to
start with improving the recognition of review and value it more. Trying to put
a new process into place over the persistent resistance is not going to work.
But as long as there's general agreement that review is good, this is the easy
part.

## No Exceptions

The trouble is that there's a few really easy way to torpedo reviews before you
event started, and they're all around special priviledges and exceptions. From
one of the LWN articles:

> ... requiring reviews might be fair, but there should be one exception: when
> developers modify their own code.

Another similar exception is often demanded by maintainers for applying their
own patches to code they maintain - in the Linux kernel only about 25% of all
maintainer patches have any kind of review tag attached when they land. This is
in contrast to other contributors, who always have to get past at least their
direct maintainer to get a patch applied.

There's a few reasons why having exceptions for the original developer of some
code, or a maintainer of a subsystem, is a really bad idea:

- Doing review (or at least full review) only for new contributors and people
  external to the subsystem makes it look like it's just an elaborate hazing
  ritual, until you're welcomed into the inner cabal. That tends to not go down
  too well with new folks, and not many want to help keep such a system running
  by actively contributing review. End result is that the pool of reviewers will
  stay limited to the old guard.

- Forcing even established contributors to go through review is a great
  opportunity for them to teach the art of a good review to someone new. Even
  when you write perfect code, eventually you'll be gone, and then someone else
  needs to have understand your code. Not subjecting your own patches to review
  by others and new contributors drops one of the best mentoring opportunities
  on the floor we have in open source.

- And really, unicorns who always write perfect code don't exist. At least I
  haven't seen them yet ...

On the flip side, requiring review from all your main contributors is a really
easy way to kickstart a working review economy: Instantly you both have a big
demand for review. And capable reviewers who are very much willing to trade a
bit of review for getting reviews on their own patches.

Another easy pitfall is maintainers who demand unconditional NAck rights for the
code they maintain, sometimes spiced up by claiming they don't even need to
provide reasons for the rejection. Of course more experienced people know more
about the pitfalls of a code base, and hence are more likely to find serios
defects in a change. But most often these rejections aren't about clear bugs,
but more design dogmas once established (and perhaps no longer valid), or just
plain personal style preferences. Again, this is a great way to prevent review
from happening:

- Anyone without NAck priviledges can only do second class review, and their
  review is then of course much less valued. Which means it won't happen.
  Note this isn't about experience, see above, review from new folks can still
  be useful, but only about who's been around for longer, or who has the commit
  powers. Valuing only review from the old guard makes sure you won't train new
  reviewers.

- It also curbs a working review economy, since non-maintainers can't unblock
  patches.  If only maintainers can provide real review, then you can't trade
  reviews. Worse, non-maintainer reviewers might direct the author into a
  direction that the maintainer doesn't approve of, wasting everyone's time.

And again, I haven't seen unicorns who write perfect code yet, neither have I
seen someone who's review feedback was consistently impeccable.

## But Document your Expectations

Training reviews through direct mentoring is great, but it doesn't scale.
Document what you expect from a review as much as possible. This includes
everything from coding style, to how much and in which detail code correctness
should be checked. But also related things like documentation, test-cases, and
process details on how exactly, when and where review is happening.

And like always, executable documentation is much better, hence try to script as
much as possible. That's why build-bots, CI bots, coding style bots, and all
these things are possible - it frees the carbon-based reviewers from wasting
time on the easy things and instead concentrate on the harder parts of review
like code design and overall architecture, and how to best get there from the
current code base. But please make sure your scripting and automated testing is
of high-quality, because if the results need interpretation by someone
experienced you haven't gained anything. The kernel's <code>checkpatch.pl</code>
coding style checker is a pretty bad example here, since it's widely accepted
that it's too opinionated and it's suggestions can't be blindly followed.

As some examples we have the [dim ingloriuos maintainer
scripts](https://01.org/linuxgraphics/gfx-docs/maintainer-tools/dim.html) and
some fairly extensive [documentation on what is expected from reviewers for
Intel graphics driver
patches](https://01.org/linuxgraphics/gfx-docs/maintainer-tools/drm-intel.html#committer-guidelines).
Contrast that to the comparetively lax [review guidelines for small drivers in
drm-misc](https://01.org/linuxgraphics/gfx-docs/maintainer-tools/drm-misc.html#small-drivers).
At Intel we've also done [internal trainings on review best practices and
guidelines](http://blog.ffwll.ch/2014/08/review-training-slides.html). Another
big thing we're working on on the automation front is [CI crunching through
patchwork
series](https://patchwork.freedesktop.org/project/intel-gfx/series/?ordering=-last_updated)
to properly regression test new patches before they land.
