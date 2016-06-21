---
layout: post
title: GEM Overview
date: '2011-05-04T12:11:00.000-07:00'
author: danvet
tags:
- In-Depth Tech
modified_time: '2013-07-21T06:58:51.484-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-1414489045975317504
blogger_orig_url: http://blog.ffwll.ch/2011/05/gem-overview.html
---

This is the script of a short intro I've given at the Linaro@UDS memory
management summit in Budapest in spring 2011. Yep, it's a bit old ...

<a name='more'></a>

The core idea of GEM is to identify graphic buffer objects with 32bit ids. The
reason being "X runs out of open fds" (KDE easily reaches a few thousand).

The core design principle behind GEM is that the kernel is in full control of
the allocation of these buffer objects and is free to move the around in any way
it sees fit. This is to make concurrent rendering by multiple processes possible
while userspace can still assume that it is in sole possession of the gpu - GEM
means "graphics execution manager".

Below some more details on what GEM is and does, what it does _not_ do and how
it relates to other graphic subsystems.

### GEM does ...

- lifecycle management. Userspace references are associated with the drm fd and
  get reaped on close (in case userspace forgets about them).

- per-device global names to exchange buffers between processes (eg dri2). These
  names are again 32bit ids. These global ids do not count as userspace
  references and don't prevent a buffer from being reaped.

- It implements very few generic ioctls:
  - flink for creating a global name for a buffer object
  - open for getting a per-fd handle to a buffer object with a global name
  - close for dropping a per-fd handle.


- a little bit of kernel-internal helpers to facilitate mmap (by blending
  multiple buffer objects into the single drm device address space) and a few
  other things.

That's it, i.e. GEM is very much meant to be as simple as possible.

### Driver-specific GEM ioctls

The generic GEM stuff is obviously not very useful. So drivers implement

quite a bit driver-specific ioctls, like:

- buffer creation. In recent kernels there is some support to create dumb
  scanout objects for KMS. But they're only really useful for boot-splashs and
  unaccelerated dumb KMS drivers. Creating buffers usable for rendering is only
  possible with driver specific ioctls.

- command submission. An important part is mapping abstract buffer ids to actual
  gpu address (and rewriting batchbuffers with these). In the future,  with
  support for virtual gpu address spaces this might change.

- tiling management. The kernel needs to know this to correctly tile/detile
  buffers when moving them around (e.g. evicting from vram).

- command completion signalling and gpu/cpu synchronization.

There are currently two approaches for implementing a GEM driver:

- roll-your-own, used by drm/i915 (and sometimes getting flaked for NIH).

- ttm-base: radeon &amp; nouveau.

### GEM does not ...

This still leaves out a few things that I've seen mentioned as
ideas/requirements here and elsewhere:

- cross-device buffer sharing and namespaces (see below) and
- buffer format handling and mediation between different users (except
  tiling as mentioned above). The reason here is that gpus are a mess

And one of the worst parts is format handling. Better keep that out
of the kernel ...


### KMS (kernel mode setting)

KMS is essentially just a port of the xrandr api to the kernel as an ioctl

interface:

- crtcs feed (possible multiple) outputs and get their data from a framebuffer
  object. A major part of KMS is also the support for vsynced-pageflipping of
  framebuffers.

- Internally there's some support infrastructure to simplify drivers (all the
  drm_*_helper.c code).

- framebuffers are created from a opaque driver-specific 32bit id and a format
  description. For GEM drivers these ids name GEM objects, but that need not be:
  The recently merged qemu kms driver does not implement  gem and has one unique
  buffer object with id 0.

- as mentioned above there newly is a generic ioctl to create an object suitable
  as a dumb scanout (plus some support to mmap it).

- currently KMS has no generic support for overlays (there are driver-specific ioctls in i915 and vmgfx, though). Jesse Barnes has posted an RFC to remedy this: <a href="http://www.mail-archive.com/dri-devel@lists.freedesktop.org/msg10415.html">http://www.mail-archive.com/dri-devel@lists.freedesktop.org/msg10415.html</a>

### GEM and PRIME

PRIME is a proof-of-concept implementation from Dave Airlie for sharing GEM
objects between drivers/devices: Buffer sharing is done with a list of struct
page pointers. While being shared, buffers can't be moved anymore.  No further
buffer description is passed along in the kernel, format/layout mediation is to
be handled in userspace.

Blog-post describing the initial design for sharing buffers between an
integrated Intel igd and a discrete ATI gpu: <a
href="http://airlied.livejournal.com/71734.html">http://airlied.livejournal.com/71734.html</a> 

Other code using the same framework to render on an Intel igd and display the
framebuffer on an usb-connected displayport: <a
href="http://git.kernel.org/?p=linux/kernel/git/airlied/drm-testing.git;a=shortlog;h=refs/heads/udl-v2">http://git.kernel.org/?p=linux/kernel/git/airlied/drm-testing.git;a=shortlog;h=refs/heads/udl-v2</a>

### GEM/KMS and fbdev

There's some minimal support to emulate an fbdev with a gem/kms driver.
Resolution can't be changed and it's unaccelerated. There's been some muttering
once in a while to better integrate this with either a kms kernel console driver
or by routing fbdev resolution changes to kms.

But the main use case is to display a kernel oops, which works. For everything
else there's X (or an EGL client that understands kms).
