---
layout: post
title: "ELCE Lyon: Everything Great About Upstream Graphics"
tags:
- Conferences
---
At ELC Europe in Lyon I held a nice little presentation about the state of
upstream graphics drivers, and how absolutely awesome it all is. Of course with
a big focus on SoC and embedded drivers. [Slides](/slides/elce-2019-upstream.pdf)
and the [video
recording](https://www.youtube.com/watch?v=kVzHOgt6WGE)

<iframe width="560" height="315" src="https://www.youtube.com/embed/kVzHOgt6WGE" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Key takeaways for the busy:

* The upstream DRM graphics subsystem really scales down to tiny drivers now,
  with the smallest driver coming in at just around 250 lines (including
  comments and whitespace), 10'000x less than the biggest!

* Batteries all included - there's modular helpers for everything. As a rule of
  thumb even minimal legacy fbdev drivers ported to DRM shrink by a factor of
  2-4 thanks to these helpers taking care of anything that's remotely
  standardized in displays and GPUs.

* For shipping userspace drivers go with a dual-stack: Open source GL and Vulkan
  drivers for those who want that, and for getting the kernel driver merged into
  upstream. Closed source for everyone else, running on the same userspace API
  and kernel driver.

* And for customer support backport the entire subsystem, try to avoid
  backporting an individual driver.

In other words, world domination is assured and progressing according to plan.
