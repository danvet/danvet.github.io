---
layout: post
title: Awesome Atomic Advances
date: '2016-06-06T02:41:00.002-07:00'
author: sima
tags:
- In-Depth Tech
modified_time: '2016-06-06T12:06:56.307-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-6308458049828256007
blogger_orig_url: http://blog.ffwll.ch/2016/06/awesome-atomic-advances.html
---

Also, silly titles. Atomic has taken of for real, right now there's 17 drivers
supporting [atomic
modesetting](/2015/08/atomic-modesetting-design-overview.html)
merged into the DRM subsystem. And still a pile of them each release pending for
review&amp;merging. But it's not just new drivers, there's also been a steady
stream of small improvements over the past year, I think it's time for an
update.



<!--more-->

It seems small, but a big improvement made over the past few months is that
<b>most driver callbacks used by the helper libraries are now optional</b>.
Which means tons and tons of dummy functions and boilerplate code can be removed
from drivers, leading to less clutter and easier to understand driver code.
Aside: Not all drivers have been decluttered, doing that is great starter
project for contributing a first few patches to the DRM subsystem. Many thanks
to Boris Brezillion, Noralf Trønnes and many others for making this happen.

A long standing complaint about the DRM kernel mode setting is that it's too
complicated, especially compared to fbdev when all you have is one dumb
framebuffer and nothing else. And yes, in that case there's really no point in
having distinct CRTC, plane and encoder objects, and now finally Noralf Trønnes
has volunteered to write a <a
href="https://lists.freedesktop.org/archives/dri-devel/2016-May/107452.html"><b>helper
library for simple display pipelines</b></a> to hide all that complexity from
drivers. It's not yet merged but I'm postive it'll land in 4.8. And it will help
to make writing DRM drivers for simple hardware easy and the driver code
clutter-free.

Another piece many dumb framebuffer drivers need is support for manually
uploading new contents to the screen. Often on this simple panels there's no
point in doing page-flipping of entire buffers since a real render engine is
nowhere to be seen. And the panels are often behind a really slow bus, making
full screen uploads to expensive. Instead it's all done by directly drawing into
the frontbuffer, and then telling the driver what changed so that it can shovel
the new bits over the bus to the panel. DRM has full support for this through a
dirty interface and IOCTL, and legacy fbdev also has some support for this. But
the fbdev emulation helpers in DRM never wired these two bits together, forcing
drivers to all type their own boilerplate. Noralf has fixed this by implementing
<a
href="https://cgit.freedesktop.org/drm-intel/commit/?id=eaa434defaca1781fb2932c685289b610aeb8b4b"><b>fbdev
deferred I/O support for the DRM fbdev helpers</b></a>.

A related improvement is <a
href="https://cgit.freedesktop.org/drm-intel/commit/?id=a03fdcb1863297481a4b817c2a759cafcbdfa0ae"><b>generic
support to disable the fbdev emulation</b></a> from Archit Tajena, both through
a Kconfig option and a module option. Most distributions still expect fbdev to
work for the boot splash, recovery console and emergency logging. But some, like
ChromeOS, are entirely legacy-free and don't need any of this. Thus far every
DRM driver had to add implement support for fbdev emulation and disabling it
optionally itself. Now that's all done in the library using dummy stub functions
in the disabled case, again simplifying driver code.

Somehow most ARM-SoC display drivers start out their system suspend/resume
support with a dumb register save/restore. I guess because with simple hardware
that works, and regmap provides it almost for free. And then everyone learns the
lessons why the atomic modeset helpers have a very strict state transition model
the hard way: Display hardware gets upset extremely easily when things are done
in the wrong order, or without the required delays, obeying the depencies
between components and much more. Dumb register restoring does none of that. To
fix this Thierry Redding implemented <a
href="https://cgit.freedesktop.org/drm-intel/commit/?id=1494276000db789c6d2acd85747be4707051c801"><b>suspend/resume
helpers for atomic drivers</b></a>. Unfortunately not many drivers use this
support yet, which is again a nice opportunity to get a kernel patch merged if
you have the hardware for such a driver.

Another big gap in the original atomic infrastructure that's finally getting
close is <a
href="http://thread.gmane.org/gmane.comp.freedesktop.xorg.drivers.intel/91023"><b>generic
support for nonblocking commits</b></a>. The tricky part there is getting the
depency tracking between commits on different display parts right, and I
secretly hoped that with a few examples it would be easier to implement
something that's useful for most drivers. With 17 examples I've finally run out
of excuse to postpone this, after more than 1 year.

But even more important than making the code prettier for atomic drivers and
removing boilerplate with better helpers and libraries is in my opinion explaing
it all, and making sure all drivers work the same. Over the past few months
there's been massive [sphinx-based documentation
toolchain](https://01.org/linuxgraphics/gfx-docs/drm/"><b>improvements to the
driver interface documentation</b></a>. One of the big items there is certainly
[documenting the expected behaviour, return codes and special cases of every
driver
callback](https://01.org/linuxgraphics/gfx-docs/drm/gpu.html#modeset-helper-reference-for-common-vtables).
But there is lots more improved than just this example, so go and read them! And
of course, when you spot an inconsistency or typo, just send in a patch to fix
it. And it's not just contents, but also presentation: Hopefully in 4.8 Jani
Nikula's <a
href="https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1158589.html)
- the above links are already generated using that for a peek at all the new
pretty.

The flip side is testing, and on that front Collabora's effort to&nbsp; convert
all the kernel mode-setting tests in [Tomeu Vizoso's blog on validating changes
to KMS
drivers](https://cgit.freedesktop.org/xorg/app/intel-gpu-tools/">intel-gpu-tools</a>
to be generic and useful on any DRM is progressing nicely. For a bit more
details read up on <a
href="http://blog.tomeuvizoso.net/2016/04/validating-changes-to-kms-drivers-with.html).

Finally it's not all improvements to make it easier to write great drivers,
there's also some new feature work. Lionel Landwerlin added new atomic
properties to <a
href="https://cgit.freedesktop.org/drm-intel/commit/?id=5488dc16fde74595a40c5d20ae52d978313f0b4e"><b>implement
color management support</b></a>. And there's work on-going to implement Z-order
and blending properties, and lots more, but that's not yet ready for merging.
