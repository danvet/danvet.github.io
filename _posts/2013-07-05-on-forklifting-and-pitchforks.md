---
layout: post
title: On Forklifting and Pitchforks
date: '2013-07-05T02:41:00.000-07:00'
author: sima
tags:
- Maintainer-Stuff
modified_time: '2013-07-21T06:55:34.737-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-2605807993115082017
blogger_orig_url: http://blog.ffwll.ch/2013/07/on-forklifting-and-pitchforks.html
---


So sometimes someone wants the latest drm kernel driver bits, but please on a
ridiculously outdated stable kernel like last month's release. Or much worse,
actually something from last year. 

Ok I'm getting ahead of myself here, usually these things start with a request
for a "minimal" backport to enable <code>$NEW_PLATFORM</code> or some other
really important feature. But the lesson that drm drivers are complex beasts and
that such minimal backports tend to require full validation again and a lot of
work to fix up the fallout is usually learned quickly. So backporting the entire
beast with all drivers it is. 

The next obvious question is why we can't just copy the entire thing, since
reconstructing the git history is rather tedious work, and usually ends up with
a few commits in between that don't quite work correctly. Which means that the
backported git history is usually of little value. The answer to that question
tends to boil down to "we've promised this to the customer ...". So if you're in
such a situation, read on below on how to most efficiently wrestle git to
forklift the history of an entire subsystem onto an older kernel release. 

<!--more-->

First we want to look at how to forklift git history at a high-level. The naive
approach would be to use git rebase. But rebasing linearizes the history, which
means that the baseline patches apply onto will change, which means that any
conflicts resolved in a merge need to instead be resolved when applying patches.
And usually changing one patch will cause conflicts to ripple to subsequent
patches, pretty much guaranteeing that the end-result is broken somewhere. So
linearizing the history is (mostly) out of the question, we need to at least
reconstruct the major merge points between branches, i.e. where conflicts are
resolved. The second option would be git filter-branch, which is slightly better
since it will preserve the large-scale structure of the history. But that
doesn't allow us to fix-up the few places where some drm change depends upon a
patch in the core kernel (pulled in by a merge) or some tree-wide interface
refactoring which for backporting purposes is usually best left out. So we're
left with manually reconstructing the history. 

For preparation the first thing to do is enable git rerere. Then it's also a
good idea to steal all the merge resolutions from your maintainer, since
otherwise you'll have to reconstruct them from merge commits by hand. For my
drm-intel branches they're all getting auto-pushed to my <a
href="http://cgit.freedesktop.org/~sima/drm-intel/log/?h=rerere-cache">rerere-cache
branch</a>. Then we need a branch based on the kernel we want to forklift
everything onto (I tend to call that branch <b>forklift</b>/<baseline>). Plus a
bunch of throwaway branches to prepare different backported branches so that we
can merge them into the main forklift branch, which I usually call
<b>pitchfork</b>s. 

The overall strategy is now to look at the large-scale structure of the history,
i.e. where different parts of the history branched off and where it is merged
back in again. For each such piece we create a new pitchfork branch which is
based off the corresponding backported commit in the forklift branch. Then we
cherry-pick from the haystack (i.e. upstream history) until patches until we hit
a merge commit. Often though, especially when the merge commit only pulled in
bugfixes late in the -rc cycle we can go right ahead an cherry-pick the next
part of the upstream history on top. But if there's a merge commit with conflict
resolutions, or if the next part of the history depends upon the merged changes
we need to merge the pitchfork branch into the main forklift branch before we
can proceed to load up the next pitchfork with new patches. 

If the history of the drm subsystem is completely independent from the core
kernel this will never result in a conflict. At least if all merges are
faithfully reconstructed on the forklift branch. But sometimes a core change
needs to be pulled in, or a tree-wide refactoring needs to not be cheery-picked
(if backporting all the other changes would be too invasive). Luckily this can
only ever happen a points in the history where commits outside of the drm
subsystem are merged in or where a branch is based on such commits. Figuring
those things out before cherry-picking is usually to cumbersome, hence I just
cherry-pick a range of commits and only test at the end, before doing the next
merge. If the patches fail to apply or if compilation fails somehow then (and
only tend) do I analyze what's going on. 

The other fun part of doing this is keeping track of which patches exactly are
merged already - the drm history easily can end up with 5-10 branches running in
parallel which each need to be put onto their own pitchfork and merged back in
at the right time. Using <code>git cherry-pick -x</code> is useful to annotate
which is the upstream commmit, I tend to add similar notes to the reconstructed
merge commits. To figure out the equivalent baseline commit for starting a new
pitchfork branch simply searching for the commit headline is good enough. And
for really complicated sections in the upstream history liberal use of tags to
mark already forklifted branches helps a lot. 

A really useful tool to check whether anything has been missed is to diff the
forklifted branch with the corresponding upstream commit, restricted to
drm-relevant directories like drivers/gpu/drm or include/drm. The diff should
only every contain things specifically ignored for the backport like tree-wide
refactorings.  

Especially if the tree is supposed to be longer-lived it is useful to annotate
the forklifted history with notes about which pieces of the upstream history had
to be backported in addition to subsystem patches and which pieces have been
ignore. I tend to use empty commits for that purpose. Finally a note of caution
for such long-lived forklift branches: It is really tempting to ignore all the
other drivers in the drm subsystem you don't care about and not backport those
patches. But that way comparing trees is more complicated since subdirectories
need to be excluded. And often there are also patches that change an interface
across all drivers, which then require to be fixed up manually. So in my opinion
it's simpler to just drag everything in the drm directories along. 
 
