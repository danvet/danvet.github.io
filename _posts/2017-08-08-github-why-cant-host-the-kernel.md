---
layout: post
title: Why Github can't host the Linux Kernel Community
author: danvet
tags:
- Maintainer-Stuff
---
A while back at the awesome [maintainerati](https://maintainerati.org/) I
chatted with a few great fellow maintainers about how to scale really big open
source projects, and how github forces projects into a certain way of scaling.
The linux kernel has an entirely different model, which maintainers hosting
their projects on github don't understand, and I think it's worth explaining why
and how it works, and how it's different.

Another motivation to finally get around to typing this all up is the [HN
discussion](https://news.ycombinator.com/item?id=13444560) on my ["Maintainers
Don't Scale" talk](/2017/01/maintainers-dont-scale.html), where the top comment
boils down to "... why don't these dinosaurs use modern dev tooling?". 
A few top kernel maintainers vigorously defend mailing lists and patch
submissions over something like github pull requests, but at least some folks from
the graphics subsystem would love more modern tooling which would be much easier
to script. The problem is that
github doesn't support the way the linux kernel scales out to a huge number of
contributors, and therefore we can't simply move, not even just a few
subsystems. And this isn't about just hosting the git data, that part obviously
works, but how pull requests, issues and forks work on github.

<!--more-->

## Scaling, the Github Way

Git is awesome, because everyone can fork and create branches and hack on the
code
very easily. And eventually you have something good, and you create a pull
request for the main repo and get it reviewed, tested and merged. And github is
awesome, because it figured out an UI that makes this complex stuff all nice&easy
to discover and learn about, and so makes it a lot simpler for new folks to
contribute to a project.

But eventually a project becomes a massive success, and no amount of tagging,
labelling, sorting, bot-herding and automating will be able to keep on top of
all the pull requests and issues in a repository, and it's time to split things
up into more manageable pieces again. More important, with a certain size and
age of a project different parts need different rules and processes: The shiny
new experimental library has different stability and CI criteria than the main
code, and maybe you have some dumpster pile of deprecated plugins that aren't
support, but you can't yet delete them: You need to split up your
humongous project into sub-projects, each with their own flavour of process and
merge criteria and their own repo with their own pull request and issue tracking.
Generally it takes a few tens to few hundreds of full time contributors until
the pain is big enough that such a huge reorganization is necessary.

Almost all projects hosted on github do this by splitting up their monorepo
source tree into lots of different projects, each with its distinct set of
functionality. Usually that results in a bunch of things that are considered the
core, plus piles of plugins and libraries and extensions. All tied together with
some kind of plugin or package manager, which in some cases directly fetches
stuff from github repos.

Since almost every big project works like this I don't think it's necessary to
delve on the benefits. But I'd like to highlight some of the issues this is
causing:

- Your community fragments more than necessary. Most contributors just have the
  code and repos around that they directly contribute to, and ignore everything
  else. That's great for them, but makes it much less likely that duplicated
  effort and parallel solutions between different plugins and libraries get
  noticed and the efforts shared. And people who want to steward the overall
  community need to deal with the hassle of tons of repos either managed through
  some script, or git submodules, or something worse, plus they get drowned in
  pull requests and issues by being subscribed to everything. Any kind of
  concern (maybe you have shared build tooling, or documentation, or whatever)
  that doesn't neatly align with your repo splits but cuts across the project
  becomes painful for the maintainers responsible for that.

- Even once you've noticed the need for it, refactoring and code sharing have
  more bureaucratic hurdles: First you have to release a new version of the core
  library, then go through all the plugins and update them, and then maybe you
  can remove the old code in the shared library.  But since everything is
  massively spread around you can forget about that last step.

  Of course it's not much work to do this, and many projects excel at making
  this fairly easy. But it is more effort than a simple pull request to the one
  single monorepo. Very simple refactorings (like just sharing a single new
  function) will happen less often, and over a long time that compounds and
  accumulates a lot of debt. Except when you go the node.js way with repos for
  single functions, but then you essentially replace git with npm as your source
  control system, and that seems somewhat silly too.

- The combinatorial explosion of theoretically supported version mixes becomes
  unsupportable. As a user that means you end up having to do the integration
  testing. As a project you'll end up with blessed combinations, or at least
  de-facto blessed combinations because developers just close bug reports with
  "please upgrade everything first". Again that means defacto you have a
  monorepo, except once more it's not managed in git. Well, except if you use
  submodules, and I'm not sure that's considered git ...

- Reorganizing how you split the overall projects into sub-projects is a pain,
  since it means you need to reorganize your git repositories and how they're
  split up. In a monorepo shifting around maintainership just amounts to
  updating OWNER or MAINTAINERS files, and if your bots are all good the new
  maintainers get auto-tagged automatically. But if your way of scaling means
  splitting git repos into disjoint sets, then any reorg is as painful as the
  initial step from a monorepo to a group of split up repositories. That means
  your project will be stuck with a bad organizational structure for too long.
 
## Interlude: Why Pull Requests Exist

The linux kernel is one of the few projects I'm aware of which isn't split up
like this. Before we look at how that works - the kernel is a huge project and
simply can't be run without some sub-project structure - I think it's
interesting to look at why git does pull requests: On github pull request is the
one true way for contributors to get their changes merged. But in the kernel
changes are submitted as patches sent to mailing lists, even long after git has
been widely adopted.

But the very first version of git supported pull requests. The audience of these
first, rather rough, releases was kernel maintainers, git was
written to solve Linus Torvalds' maintainer problems. Clearly it was needed and
useful, but not to handle changes from individual contributors: Even today, and
much more back then, pull requests are used to forward the changes of an entire
subsystem, or synchronize code refactoring or similar cross-cutting change
across different sub-projects. As an example, the [4.12 network pull request
from Dave S. Miller](https://lkml.org/lkml/2017/5/2/508), [committed by
Linus](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8d65b08debc7e62b2c6032d7fe7389d895b92cbc):
It contains 2k+ commits from 600 contributors and a bunch of merges for pull
requests from subordinate maintainers. But almost all the patches themselves are
committed by maintainers after picking up the patches from mailing lists, not by
the authors themselves. This kernel process peculiarity that authors generally
don't commit into shared repositories is also why git tracks the committer and
author separately.

Github's innovation and improvement was then to use pull requests for
everything, down to individual contributions. But that wasn't what they were
originally created for.

## Scaling, the Linux Kernel Way

At first glance the kernel looks like a monorepo, with everything smashed into
one place in [Linus' main
repo](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/). But that's very far from it:

- Almost no one who's using linux is running the main repo from Linus Torvalds.
  If they run something upstream-ish it's probably one of the [stable
  kernels](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git/).
  But much more likely is that they run a kernel from their distro, which
  usually has additional patches and backports, and isn't even hosted on
  [kernel.org](https://git.kernel.org/), so would be a completely different
  organization. Or they have a kernel from their hardware vendor (for SoC and
  pretty much anything Android), which often have considerable deltas compared
  to anything hosted in one of the "main" repositories.

- No one (except Linus himself) is developing stuff on top of Linus' repository.
  Every subsystem, and often even big drivers, have their own git repositories,
  with their own mailing lists to track submissions and discuss issues
  completely separate from everyone else.

- Cross-subsystem work is done on top of the [linux-next integration
  tree](https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/),
  which contains a few hundred git branches from about as many different git
  repositories.

- All this madness is managed through the
  [MAINTAINERS](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/MAINTAINERS)
  file and the get_maintainers.pl script, which for any given snippet of code can
  tell you who's the maintainer, who should review this, where the right git
  repo is, which mailing lists to use and how and where to report bugs. And it's
  not just strictly based on file locations, it also catches code patterns to
  make sure that cross-subsystem topics like device-tree handling, or the
  kobject hierarchy are handled by the right experts.

At first this just looks like a complicated way to fill everyone's disk space
with lots of stuff they don't care about, but there's a pile of compounding
minor benefits that add up:

- It's dead easy to reorganize how you split things into sub-project, just
  update the MAINTAINERS file and you're done. It's a bit more work than it
  really needs to be, since you might need to create a new repo, new mailing
  lists and a new bugzilla. That's just an UI problem that github solved with
  this neat little *fork* button.

- It's really really easy to reassign discussions on pull requests and issues
  between sub-projects, you simply adjust the Cc: list on your reply.
  Similarly, doing cross-subsystem work is much easier to coordinate, since the
  same pull request can be submitted to multiple sub-projects, and there's just
  one overall discussions (since the Msg-Ids: tags used for mailing list
  threading are the same for everyone), despite that the mails are archived in a
  pile of different mailing list archives, go through different mailing lists and
  land in a few thousand different inboxes. Making it easier to discuss topics
  and code across sub-projects avoids fragmentation and makes it much easier to
  spot where code sharing and refactoring would be beneficial.

- Cross-subsystem work doesn't need any kind of release dance. You simply change
  the code, which is all in your one single repository. Note that this is
  strictly more powerful than what a split repo setup allows you: For
  really invasive refactorings you can still space out the work over multiple
  releases, e.g. when there's so many users that you can just change them all at
  once without causing too big coordination pains.

  A huge benefit of making refactoring and code sharing easier is that you don't
  have to carry around so much legacy gunk. That's explained at length in the
  kernel's [no stable api
  nonsense](https://dri.freedesktop.org/docs/drm/process/stable-api-nonsense.html#stable-kernel-source-interfaces)
  document.

- It doesn't prevent you from creating your own experimental additions, which is
  one of the key benefits of the multi-repo setup. Add
  your code in your own fork and leave it at that - no one ever forces you to push the code back,
  or push it into the one single repo or even to the main organization, because
  there simply is no central repositories. This works really well, maybe too
  well, as evidenced by the millions of code lines which are *out-of-tree* in
  the various Android hardware vendor repositories.

In short, I think this is a strictly more powerful model, since you can always
fall back to doing things exactly like you would with multiple disjoint
repositories. Heck there's even kernel drivers which are in their own
repository, disjoint from the main kernel tree, like the proprietary Nvidia
driver. Well it's just a bit of source code glue around a blob, but since it
can't contain anything from the kernel for legal reasons it is the perfect
example.

### This looks like a monorepo horror show!

Yes and no.

At first glance the linux kernel looks like a monorepo because it contains
everything. And lots of people learned that monorepos are really painful,
because past a certain size they just stop scaling.

But looking closer, it's very, very far away from a single git repository.
Just looking at the upstream subsystem and driver repositories gives you a few
hundred. If you look at the entire ecosystem, including hardware vendors,
distributions, other linux-based OS and individual products, you easily have a
few thousand major repositories, and many, many more in total. Not counting
any git repo that's just for private use by individual contributors.

The crucial distinction is that linux has one single file hierarchy as the
shared namespace across everything, but lots and lots of different repos for all
the different pieces and concerns. It's a monotree with multiple repositories,
not a monorepo.

### Examples, please!

Before I go into explaining why github cannot currently support this workflow,
at least if you want to retain the benefits of the github UI and integration, we
need some examples of how this works in practice. The short summary is that it's
all done with git pull requests between maintainers.

The simple case is percolating changes up the maintainer hierarchy, until it
eventually lands in a tree somewhere that is shipped. This is easy, because the
pull request only ever goes from one repository to the next, and so could be
done already using the current github UI.

Much more fun are cross-subsystem changes, because then the pull request flow
stops being an acyclic graph and morphs into a mesh. The first step is to get
the changes reviewed and tested by all the involved subsystems and their
maintainers. In the github flow this would be a pull request submitted to
multiple repositories simultaneously, with the one single discussion stream
shared among them all. Since this is the kernel, this step is done through patch
submission with a pile of different mailing lists and maintainers as
recipients.

The way it's reviewed is usually not the way it's merged, instead one of the
subsystems is selected as the leading one and takes the pull requests, as long
as all other maintainers agree to that merge path. Usually it's the subsystem
most affected by a set of changes, but sometimes also the one that already has
some other work in-flight which conflicts with the pull request. Sometimes also
an entirely new repository and maintainer crew is created, this often happens
for functionality which spans the entire tree and isn't neatly contained to a
few files and directories in one place. A recent example is the [DMA mapping
tree](http://git.infradead.org/users/hch/dma-mapping.git/commit/2e7d1098c00caebc8e31c4d338a49e88c979dd2b),
which tries to consolidate work that thus far has been spread across drivers,
platform maintainers and architecture support groups.

But sometimes there's multiple subsystems which would both conflict with a set
of changes, and which would all need to resolve some non-trivial merge
conflict. In that case the patches aren't just directly applied (a rebasing pull
request on github), but instead the pull request with just the necessary
patches, based on a commit common to all subsystems, is merged into *all*
subsystem trees. The common baseline is important to avoid polluting a subsystem
tree with unrelated changes. Since the pull is for a specific topic only, these
branches are commonly called *topic branches*.

One example I was involved with added code for audio-over-HDMI support, which
spanned both the graphics and sound driver subsystems. The same commits from the
same pull request where
both [merged into the Intel graphics
driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2844659842017c981f6e6f74aca3a7ebe10edebb)
and also [merged into the sound
subsystem](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3c95e0c5a6fa36406fe5ba9a2d85a11c1483bfd0).

An entirely different example that this isn't insane is the [only other relevant
general purpose large scale OS project](https://www.microsoft.com/windows/) in
the world *also* decided to have a monotree, with a commit flow modelled similar
to what's going on in linux. I'm talking about the folks with such a huge tree
that they had to write an entire new [GVFS](https://github.com/Microsoft/GVFS)
virtual filesystem provider to support it ...

## Dear Github

Unfortunately github doesn't support this workflow, at least not natively in the
github UI. It can of course be done with just plain git tooling, but then you're
back to patches on mailing lists and pull requests over email, applied manually.
In my opinion that's the single one reason why the kernel community cannot
benefit from moving to github. There's also the minor issue of a few top
maintainers being extremely outspoken against github in general, but that's a
not really a technical issue. And it's not just the linux kernel, it's all huge
projects on github in general which struggle with scaling, because github
doesn't really give them the option to scale to multiple repositories, while
sticking to with a monotree.

In short, I have one simple feature request to github:

> Please support pull requests and issue tracking spanning different repos of a
> monotree.

Simple idea, huge implications.

### Repositories and Organizations

First, it needs to be possible to have multiple forks of the same repo in one
organization. Just look at [git.kernel.org](https://git.kernel.org/), most of
these repositories are not personal. And even if you might have different
organizations for e.g. different subsystems, requiring an organization for each
repo is silly amounts of overkill and just makes access and user managed
unnecessarily painful. In graphics for example we'd have 1 repo each for the
userspace test suite, the shared userspace library, and a common set of tools
and scripts used by maintainers and developers, which would work in github. But
then we'd have the overall subsystem repo, plus a repository for core subsystem
work and additional repositories for each big drivers. Those would all be forks,
which github doesn't do. And each of these repos has a bunch of branches, at
least one for feature work, and another one for bugfixes for the current release
cycle.

Combining all branches into one repository wouldn't do, since the point of
splitting repos is that pull requests and issues are separated, too.

Related, it needs to be possible to establish the fork relationship after the
fact. For new projects who've always been on github this isn't a big deal. But
linux will be able to move at most a subsystem at a time, and there's already
tons of linux repositories on github which aren't proper github forks of each
another.

### Pull Requests

Pull request need to be attached to multiple repos at the same time, while
keeping one unified discussion stream. You can already reassign a pull request
to a different branch of repo, but not at multiple repositories at the same
time. Reassigning pull requests is really important, since new contributors will
just create pull requests against what they think is the main repo. Bots can
then shuffle those around to all the repos listed in e.g. a MAINTAINERS file for
a given set of files and changes a pull request contains. When I chatted with
githubbers I originally suggested they'd implement this directly. But I think as
long as it's all scriptable that's better left to individual projects, since
there's no real standard.

There's a pretty funky UI challenge here since the patch list might be different
depending upon the branch the pull request is against. But that's not always a
user error, one repo might simple have merged a few patches already.

Also, the pull request status needs to be different for each repo. One
maintainer might close it without merging, since they agreed that the other
subsystem will pull it in, while the other maintainer will merge and close the
pull. Another tree might even close the pull request as invalid, since it
doesn't apply to that older version or vendor fork. Even more fun, a pull
request might get merged multiple times, in each subsystem with a
different merge commit.

### Issues

Like pull requests, issues can be relevant for multiple repos, and might need to
be moved around. An example would be a bug that's first reported against a
distribution's kernel repository. After triage it's clear it's a driver bug
still present in the latest development branch and hence also relevant for that
repo, plus the main upstream branch and maybe a few more.

Status should again be separate, since once push to one repo the bugfix isn't
instantly available in all of them. It might even need additional work to get
backported to older kernels or distributions, and some might even decide that's
not worth it and close it as *WONTFIX*, even thought the it's marked as
successfully resolved in the relevant subsystem repository.

## Summary: Monotree, not Monorepo

The Linux Kernel is not going to move to github. But moving the Linux way of
scaling with a monotree, but mutliple repos, to github as a concept will be
really beneficial for all the huge projects already there: It'll give them a
new, and in my opinion, more powerful way to handle their unique challenges.
