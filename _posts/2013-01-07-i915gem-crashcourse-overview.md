---
layout: post
title: 'i915/GEM Crashcourse: Overview'
date: '2013-01-07T08:57:00.001-08:00'
author: danvet
tags:
- In-Depth Tech
modified_time: '2013-07-21T06:58:51.502-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-7367005160029620308
blogger_orig_url: http://blog.ffwll.ch/2013/01/i915gem-crashcourse-overview.html
---

Now that the entire series is done I've figured a small overview would be in
order.

[Part 1](/2012/10/i915gem-crashcourse.html) talks about the
different address spaces that a i915 GEM buffer object can reside in and where
and how the respective page tables are set up. Then it also covers different
buffer layouts as far as they're a concern for the kernel, namely how tiling,
swizzling and fencing works.

[Part 2](/2012/11/i915gem-crashcourse-part-2.html) covers
all the different bits and pieces required to submit work to the gpu and keep
track of the gpu's progress: Command submission, relocation handling, command
retiring and synchronization are the topics.

[Part 3](/2012/11/i915gem-crashcourse-part-3.html) looks at
some of the details of the memory management implement in the i915.ko driver.
Specifically we look at how we handle running out of GTT space and what happens
when we're generally short on memory.

Finally [part 4](/2013/01/i915gem-crashcourse-part-4.html) discusses coherency and caches and how to most efficiently transfer between the gpu coherency domains and the cpu coherncy domain under different circumstances.

Happy reading! 

<b>Update:</b> There's now also a new article with a few [questions and answers
about some details in the i915 gem code](/2013/05/i915gem-q.html).
