---
layout: post
title: New Kernel Modesetting Locking
date: '2013-02-07T06:42:00.000-08:00'
author: sima
tags:
- In-Depth Tech
modified_time: '2013-07-21T06:58:51.506-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-816932904033519461
blogger_orig_url: http://blog.ffwll.ch/2013/02/new-kernel-modesetting-locking.html
---


Now that my kernel modesetting lockig rework has [landed](http://cgit.freedesktop.org/~airlied/linux/commit/?h=drm-next&amp;id=735dc0d1e29329ff34ec97f66e130cce481c9607) in Dave's drm-next tree and is gearing up for inclusion into 3.9 I've figured it's time to also post my little intro here: 

The aim of this locking rework is that ioctls which a compositor should be might call for every frame (set_cursor, page_flip, addfb, rmfb and getfb/create_handle) should not be able to block on kms background activities like output detection. And since each EDID read takes about 25ms (in the best case), that always means we'll drop at least one frame. 

The solution is to add per-crtc locking for these ioctls, and restrict background activities to only use the global lock. Change-the-world type of events (modeset, dpms, ...) need to grab all locks. 
<!--more-->
Two tricky parts arose in the conversion: 

<ul><li>A lot of current code assumes that a kms fb object can't disappear while holding the global lock, since the current code serializes fb destruction with it. Hence proper lifetime management using the already created refcounting for fbs need to be instantiated for all ioctls and interfaces/users. </li><li>The rmfb ioctl removes the to-be-deleted fb from all active users. But unconditionally taking the global kms lock to do so introduces an unacceptable potential stall point. And obviously changing the userspace abi isn't on the table, either. Hence this conversion opportunistically checks whether the rmfb ioctl holds the very last reference, which guarantees that the fb isn't in active use on any crtc or plane (thanks to the conversion to the new lifetime rules using proper refcounting). Only if this is not the case will the code go through the slowpath and grab all modeset locks. Sane compositors will never hit this path and so avoid the stall, but userspace relying on these semantics will also not break.  </li></ul>

All these cases are exercised by the newly [subtests](http://cgit.freedesktop.org/xorg/app/intel-gpu-tools/commit/?id=f56384c2c0264989acae7b8c7efb5030a6c94caf">added</a> <a href="http://cgit.freedesktop.org/xorg/app/intel-gpu-tools/commit/?id=df41e1a6bbe75f047e83bb543e0b0564ed10862b) for the i-g-t kms_flip, tested on a machine where a full detect cycle takes around 100 ms. It works, and no frames are dropped any more with these patches applied. kms_flip also contains a special case to exercise the above-describe rmfb slowpath. 
 
For anyone curious about some of the details of this rework or ARM drm driver writers needing to port out-of-tree stuff I suggest to just read through the patches - I've tried really hard to properly document things in commmit messages and note about the implications of the new design. 
 
The downside of this all is that we can now enable some really paranoid inter-frame jitter checks and vblank counter timestamp checks in the kms_flip testcase. After all, no frames should be dropped any longer. But it turns out that a lot of the different platforms still have small issues here and there with races and other inconsistencies like completing a page flip immediately right after a modeset. 
 
So there's still plenty of work to do until we have perfect pageflip support. And there's also a few funny issues like racing gpu hangs against pageflips, client crashes against pageflips or trying to flip overlays and cursor on the same vblank as the underlying framebuffer. Ville Syrjälä and other guys at Intel and elsewhere are working on this. So stay tuned. 
