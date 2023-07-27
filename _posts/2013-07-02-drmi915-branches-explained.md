---
layout: post
title: drm/i915 Branches Explained
date: '2013-07-02T06:14:00.000-07:00'
author: sima
tags:
- Maintainer-Stuff
modified_time: '2014-05-26T00:13:51.456-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-6661539345779075349
blogger_orig_url: http://blog.ffwll.ch/2013/07/drmi915-branches-explained.html
---

Since there've been questions I'll try to explain the various branches in the
drm-intel.git repository a bit.

<!--more-->

There are two main branches I merge patches into, namely <b>drm-intel-fixes</b> and <b>drm-intel-next-queued</b>. drm-intel-fixes contains bugfixes for the current -rc upstream kernels, whereas drm-intel-next-queued (commonly shortened to dinq) is for feature work. The first special thing is that contrary to subsystem trees I don't close dinq while the upstream merge window is open - stalling QA and our own driver feature and review work for usually more than 2 weeks is a bit too disruptive. But the linux-next tree maintained by upstream forbids this, hence why I also have the for-linux-next branch: Usually it's a copy of the drm-intel-next-queued branch safe when the merge window is open, then it's a copy of the drm-intel-fixes branch. This way we can keep the process in our team going without upsetting Stephen Rothwell. 

Usually once the merge-window opens I have a mix of feature work (which then obviously missed the deadline) and bugfixes oustanding (i.e. not yet merged to Dave Airlie's drm branches). Therefore I split out the bugfixes from dinq into -fixes and rebase  the remaining feature work in dinq on top of that.



Now for our own nightly QA testing we want a lightweight linux-next just for graphics. Hence there's the <b>drm-intel-nightly</b> branch which currently merges together drm-intel-fixes and -next-queued plus the drm-next and drm-fixes branches from Dave Airlie. We've had some ugly regressions in the past due to changes in the drm core which we've failed to detect for months, hence why we now also include the drm branches.



Unfortunately not all of our QA procedures are fully automated and can be run every night, so roughly every two weeks I try to stabilize dinq a bit and freeze it as <b>drm-intel-next</b>. I also push out a snapshot of -nightly to <b>drm-intel-testing</b>, so that QA has a tree with all the bugfixes included to do the extensive and labour-intensive manual tests. Once that's done (usually takes about a week) and nothing bad popped up I send a pull request to Dave Airlie for that drm-intel-next branch.



There's also still the for-airlied branch around, but since I now use git tags both for -fixes and -next pull requests I'll probably remove that branch soon. 

Finally there's the <b>rerere-cache</b> branch. I have a bunch of scripts which automatically create drm-intel-nightly for me, using git rerere and a set of optional fixup patches. To be able to seamlessly do both at home with my workstation and on the road with my laptop the scripts also push out the latest git rerere data and fixup branches to this branch. When I switch machines I can then quickly sync up all the maintainer branch state to the next machine with a simple script.



Now if you're a developer and wonder on which branch you should base your patches on it's pretty simple: If it's a bugfix which should be merged into the current -rc series it should be based on top of drm-intel-fixes. Feature work on the other hand should be based on drm-intel-nightly - if the patches don't apply cleanly to plain dinq I'll look at the situation and decide whether an explicit backport or just resolving the conflicts is the right approach. Upstream maintainers don't like backmerges, hence I try to avoid them if it makes sense. But that also means that dinq and -fixes can diverge quite a bit and that some of the patches in -fixes would break things on plain dinq in interesting ways. Hence the recommendation to base feature development on top of -nightly.



Addendum: All these branches can be found in the [drm-intel git repository](http://cgit.freedesktop.org/drm-intel/).



<b>Update:</b> The [drm-intel repository
move](/2014/02/new-drmi915-git-repository.html), so I've
updated the link. 
