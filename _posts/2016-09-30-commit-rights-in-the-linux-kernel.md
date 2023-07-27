---
layout: post
title: Commit Rights in the Linux Kernel?!
date: '2016-09-30T06:32:00.001+01:00'
author: sima
tags: 
- Maintainer-Stuff
- Conferences
---

Since about a year we're running the Intel graphics driver with a new process:
Besides the two established maintainers we've added all regular contributors as
committers to the main feature branch feeding into -next. This turned out into a
tremendous success, but did require some initial adustments to how we run things
in the first few months.

I've presented the new model here at Kernel Recipes in Paris, and I will also
talk about it at Kernel Summit in Santa Fe. Since [LWN](https://lwn.net/) is
present at both I won't bother with a full writeup, but leave that to much
better editors. **Update**: [LWN on kernel maintainer
scalability](https://lwn.net/Articles/703005/).

Anyway, there's a [video
recording](https://www.youtube.com/watch?v=gZE5ovQq9g8&feature=youtu.be&list=PLQ8PmP_dnN7L5OVT95uXJAE78qcGCcDVm)
and the [slides](/slides/kernel-recipes-2016-maintainer.odp). Our [process is
also
documented](https://01.org/linuxgraphics/gfx-docs/maintainer-tools/drm-intel.html) -
scroll down to the bottom for the more interesting bits around what's
expected of committers.

On a related note: At XDC, and a bit before, Emma Anholt started a discussion
about improving our patch submission process, especially for new contributors.
He used the Rust community as a great example, and [presented about it at
XDC](https://youtu.be/KIHrjgZJHZA?t=18480). Rather interesting to hear
his perspective as a first-time contributor confirm what I learned in LCA this
year in Emily Dunham's awesome talk on [Life is better with Rust's community
automation](https://www.youtube.com/watch?v=dIageYT0Vgg).
