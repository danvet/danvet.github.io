---
layout: post
title: On Getting Your API Right
date: '2011-01-06T10:42:00.000-08:00'
author: danvet
tags: 
modified_time: '2011-01-06T10:42:24.725-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-1596341627763159524
blogger_orig_url: http://blog.ffwll.ch/2011/01/on-getting-your-api-right.html
---

Ben Widawsky is writing hardware contexts support for the i915 drm module. The resulting API <a href="http://lists.freedesktop.org/archives/intel-gfx/2010-December/009059.html">discussion</a> showed that things are not simple and probably in need of an iteration or two. Now linux kernel rules mandate that an ioctl once merged must be supported (almost) forever. So one can still see all the evolutionary steps of gpu command submission:



<b>cmdbuffer:</b> The kernel simply copies the commands given by userspace into the hardware ringbuffer. Bad, because the kernel needs to copy everything. And it can't use hardware features to check the commands because commands in the ringbuffer are assumed to be trusted.



<b>batchbuffer:</b> Userspace hands in a GTT adress and the kernel executes it with an MI_BATCHBUFFER command from the ringbuffer. No copying, and the hardware does some rudimentary security checking (the newer your chip, the more). But handing in a fixed GTT address requires that userspace is in full control of the graphics memory. Which fails with GEM - if you neglect some rather hacky porting of libXvMC to DRI2/GEM ...



<b>execbuffer:</b> Due to GEM, userspaces references buffer objects only by handles. So the new command submission path takes the id of a buffer containing the commands. Plus a list of buffer objects used by this command stream (textures, shaders, color- and depthbuffer, ...) and a list of relocations so the kernel can fix up buffer addresses in the command stream. As an optimization, the kernel passes back the current GTT address of all buffers used. This way the kernel only needs to fix up address in when buffers have moved around since the last time being used.



<b>execbuffer2:</b> Unfortunately the structures used by execbuf don't have reserved fields, so the next extension required an new ioctl. Older hardware (still being sold in Atom based netbooks) requires the setup of so called buffer fences (not to be confused with execution fences) for tiled buffers. But only for certain commands. There are only 16 fences available (on newer hardware, 8 on older) so this severely restricts performance. Especially since fences are not required by the 3D engine. So execbuf2 was born with a flag to explicitly designate buffers that need fences.



Due to plenty of reserved fields in execbuf2 <b>multiple ringbuffer suppor</b>t didn't need a new ioctl. But to keep things funny, there are two versions: First inter-ring synchronization must have been done by userspace, in later versions the kernel takes care of this.



And now hardware context support adds its own complications. And likely requires another interface for command submission. Or two, because Intel hardware moves towards a per-process GTT, possibly handing back full control over the memory layout to userspace - somewhat similar to the DRI1 interfaces ...



And for anyone forced to design an API, read these <a href="http://ozlabs.org/~rusty/index.cgi/tech/2008-03-30.html">two</a> <a href="http://ozlabs.org/~rusty/index.cgi/tech/2008-04-01.html">posts</a> by Rusty Russells.