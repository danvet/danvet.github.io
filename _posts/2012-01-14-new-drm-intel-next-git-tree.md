---
layout: post
title: New drm-intel-next Git Tree
date: '2012-01-14T05:33:00.000-08:00'
author: danvet
tags:
- Maintainer-Stuff
modified_time: '2013-07-21T06:56:15.364-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-8123378769638523809
blogger_orig_url: http://blog.ffwll.ch/2012/01/new-drm-intel-next-git-tree.html
---

[This is a verbatim copy of the announcement that went out to intel-gfx &amp;
Co. today.]

Because Keith is routinely really busy with all kinds of things, notably
gathering fixes for drm-intel-fixes, the patch merge process for the next
release cycle sometimes falls behind. To support him and improve things I've
been volunteered to take over handling the -next tree.

<!--more-->

The main aim is to shift the drm/i915 -next merge process massively ahead with the goals to:

<ul><li>Reduce pressure to merge questionable patches into -rc kernels because the -next tree is not yet open for patches.</li><li>Allow our QA at Intel and also the community to actually test things before  they land in mainline. The lack of such testing has severly bitten us in the  past few releases.</li><li>Refocus -fixes on handling regressions with absolute top priority (as it  should).</li><li>And generally get a steady and predictable patch-flow towards mainline back  into gears.

</li></ul>

I plan to run this -next tree with a few simple rules:

<ul><li>I'll open the drm/i915 -next tree around -rc1 (maybe earlier in the future)  and cut regular new trees about every 2nd week or so. 2 weeks should be enough  for both our qa and the community to give it some decent testing.</li><li>I intend to send out the previous -next to Dave Airlie (assuming it tests ok)  so that he has a good check on the stuff that's going on in i915-land. A few  other people also asked for a better overview of what's going on on drm/i915 -  I'll plan to announce every new -next tree with a short mail (maybe together  with the pull request to Dave for the old one).</li><li>-next will close about 1-2 weeks before the merge opens. No new features after  that point for a given release cycle.</li><li>To make this really work we also need to cut down on what can go into -fixes.  drm/i915 unfortunately has the reputation that deserves it a bunch of  draconian rules (which are stricter than drm/* in general):<ul><li>  Only fixes for serious issues and regressions after the -next tree went to    Dave.</li><li>After -rc2 regression fixes only. It simply happend why too often that an    "obvious" patch blew up late in the -rc release cycle, my patches included.  - After -rc4/5 only reverts and disable patches. Imo it's way too late to play    games by then, the real fix should go straight to -next (which will close    only a few weeks afterwards already).</li></ul></li><li>Regressions have top priority, if they get neglected due to ongoing work  headed for -next I'll refuse to merge the patches.

</li><li>We have a test-suite in the intel-gpu-tools package for the kernel. Expect me  to be annoying about patches that should have testcases for them but don't.  This includes new features, but also bugs that can be reproduced with a  reasonable amount of effort.</li><li>To avoid me pushing utter crack I will only merge my own patches after they  have gathered sufficient review on intel-gfx. Please yell at me at the top of  your voice (and in public) if I violate this.</li><li>The main discussion list for patches to drm/i915 is  <a href="mailto:intel-gfx@lists.freedesktop.org">intel-gfx@lists.freedesktop.org</a> - I don't keep up with lkml usually.</li><li>I'll reserve my human right to act like an incompetent arrogant fool once in a  while.</li></ul>

Last but not least, the new tree is available at



<a href="http://cgit.freedesktop.org/%7Edanvet/drm-intel">http://cgit.freedesktop.org/~danvet/drm-intel</a>

git://people.freedesktop.org/~danvet/drm-intel



drm-intel-next is the main branch, but I expect at least a for-airlied branch for pull requests and maybe other branches with proposed patches to show up.



Comments, flames and suggestions highly welcome.



Yours, Daniel



PS: Quick version for people with too short attentation spans:



<ul><li>-next will open around -rc1, a new tree should show up about every second  week. -next trees that are tested will regurarly be forwarded to Dave.</li><li>No stuff in -fixes that should go into -next instead.</li><li>-next will close for features about 1-2 weeks ahead of the upstream merge window.</li><li>Regressions have top priority.</li><li>Implementing proper tests is required.</li><li>Hit me hard if I break these rules for my own patches.</li><li>Sometimes I'll screw things up badly.

</li></ul>Now grab the tree from



git://people.freedesktop.org/~danvet/drm-intel
