---
layout: post
title: i915/GEM Crashcourse, Part 2
date: '2012-11-01T07:09:00.000-07:00'
author: sima
tags:
- In-Depth Tech
modified_time: '2013-07-21T06:58:51.486-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-1685551924522514150
blogger_orig_url: http://blog.ffwll.ch/2012/11/i915gem-crashcourse-part-2.html
---


After the [previous installement](/2012/10/i915gem-crashcourse.html) previous
installement this part will cover command submission to the gpu. See the
[i915/GEM crashcourse overview](/2013/01/i915gem-crashcourse-overview.html) for
links to the other parts of this series.  

<!--more-->

## Command Submission and Relocations 


As I've alluded already, gpu command submission on intel hardware happens by sending a special buffer object with rendering commands to the kernel for execution on the gpu, the so called batch buffer. The ioctl to do this is called <code>execbuf</code>. Now this buffer contains tons of references to various other buffer objects which contain textures, render buffers, depth&amp;stencil buffers, vertices, all kinds of gpu specific things like shaders and also quite some state buffers which e.g. describe the configuration of specific (fixed-function) gpu units.
 
The problem now is that userspace doesn't control where all these buffers are - the kernel manages the entire GTT.&nbsp; And the kernel needs to manage the entire GTT, since otherwise multiple users of the same single gpu can't get along. So the kernel needs to be able to move buffers around in the GTT when they don't all fit in at the same time, which means clearing the PTEs in the relevant pagetables for the old buffers that get kicked out and then filling them again with entries pointing at new the buffers which the gpu now requires to execute the batch buffer. In short userspace needs to fill the batchbuffer with tons of GTT addresses, but only the kernel really knows them at any given point.
 
This little problem is solved by supplying a big list of relocation entries along with the batchbuffer and a list of all the buffers required to execute this batch. To optimize for the common case where buffers don't move around, userspace prefills all references with the GTT offsets from the last command submission (the kernel is so kind to tell userspace the updated offset after successful submission of a batch). The kernel then goes through that relocation list, checks whether the offsets that userspace presumed are still correct. And if that's not the case, it updates the buffer references in the batch and so relocates the referenced data, hence the name.
 
A slight complication is that the gpu data structures can be several levels deep, e.g. the batch points at the surface state, which then points at the texture/render buffers. So each buffer in the command submission list has a relocation list pointer, but for most buffers it's just NULL (since they just contain data and don't reference any other buffers).
 
Now along with any information required to rewrite references for relocated buffers, userspace also supplies some other information (the read/write domains) about how it wants to use the buffer. Originally this was to optimize cache coherency management (coherency will be covered in detail later on), but nowadays that code is massively simplified since that clever optimized cache tracking is simply not worth it. We do though still use these domain values to implement a workaround on Sandybridge though: Since we use PPGTT, all memory accesses from the batch are directed to go through the PPGTT, with the exception that pipe control writes (useful for a bunch of things, but mostly for OpenGL queries) always go through the global GTT (it's a bug in the hw ...). Hence we need to ensure that we not only bind to the PPGTT, but also set up a mapping in the global GTT. And we detect this situation by checking for a GEM_INSTRUCTION write domain - only pipe control writes have that set.
 
The other special relocation is for older generations, where the gpu needs a fence set up to access tiled buffers, at least for some operations. The relocation entries have a flag for that to signal the kernel that a fence is required. Another peculiarity is that fences can only be set up in the mappable part of the GTT, at least on those chips that require them for rendering. Hence we also restrict the placement of any buffers that require a fence to the mappable part of the GTT.
 
So after rewriting any references to buffers that moved around, the kernel is ready to submit the batch to the gpu. Every gpu engine has a ringbuffer that the kernel can fill with its own commands. First we emit a few preparatory commands to flush caches and set a few registers (which normal userspace batches can't write) to the values that userspace needs. Then we start the batch by emitting a MI_BATCHBUFFER_START command.
 

## Retiring and Synchronization

 
Now the gpu can happily process the commands and do the rendering, but that leaves the kernel with a problem: When is the gpu done? Userspace obviously needs to know this to avoid reading back incomplete results. But the kernel also needs to know this, to avoid unmapping buffers which are still in use by the gpu, e.g. when a render operation requires a temporary buffer, userspace might free that buffer right away after the <code>execbuf</code> call completes. But the kernel needs to delay the unmapping and freeing of the backing storage until the gpu not longer needs that buffer.
 
Therefore the kernel associates a sequence number with every batchbuffer and adds a write of that sequence number to the ringbuffer. Every engine has a hardware status page (HWS_PAGE) which we can use for such synchronization purposes. The upshot of that special status page is that gpu writes to it snoop the cpu caches, and hence a read from it is much faster than reading directly the gpu head pointer register of the ring buffer. We also add a MI_USER_IRQ command after the sequence number (seqno for short) write, so that we don't need to constantly poll when waiting for the gpu.
 
Two little optimizations apply to this gpu-to-cpu synchronization mechanism: If the cpu doesn't wait for the gpu, we mask the gpu engine interrupts to avoid flooding the cpu with thousands of interrupts (and potentially waking it up from deeper sleep states all the time). And the seqno read has a fastpath which might not be fully coherent, and a potentially much more expensive slowpath. This is because of some coherency issues on recent platforms, where the interrupt seemingly arrives before the seqno write has landed in the status page. Since we check that seqno rather often it's good to have an lightweight check which might not give the most up-to-date value, but is good enough to avoid going through more costly slowpaths in the code that handles synchronization with the gpu.
 
So now we have a means to track the progress of the gpu through the batches submitted to the engine's ringbuffer, but not yet a means to prevent the kernel from unmapping or freeing still in-use buffers. For that the kernel keeps a per-engine list of all active buffers, and marks each buffer with the seqno of the latest batch it has been used for. It also keeps a list of all still outstanding seqnos in a per-engine request list. The clever trick now is that the kernel keeps an additional reference on each buffer object that resides on one of the active list - that way a buffer can never disappear while still in use by the gpu, even when userspace removes all it's references. To batch up the active list processing and retiring of any newly completed requests, the kernel runs a regular task from a worker thread to clean things up.
 
To avoid polling in userspace the kernel also provides interfaces for userspace to wait until rendering completes on a given buffer object: The <code>wait_timeout</code> ioctl simply waits until the gpu is done with an object (optionally with a timeout), the <code>set_domain</code> ioctl doesn't have a timeout, but additionally takes a flag to indicate whether userspace only wants to read or also whether it wants to write. Furthermore <code>set_domain</code> also ensures that cpu caches are coherent, but we will look at that little issue later on. The set_domain ioctl doesn't wait for all usage by the gpu to complete if userspace only wants to read the buffer. In that case it only waits for all outstanding gpu writes - the kernel knows this thanks to the separate read/write domains in the relocation entries and keeps track of both the last gpu write and read by remembering the seqnos of the respective batches.
 
The kernel also supports a <code>busy</code> ioctl to simply inquire whether a buffer is still being used by the gpu. This recently gained the ability to tell userspace on which gpu engine an object is busy on - which is useful for compositors that get buffer objects from clients to decide which engine is the most suitable one, if a given operation can be done with more than one engine (pretty much all of them can be coaxed into copying data).
 
With that we have gpu/cpu synchronization covered. But as just mentioned above, the gpu itself also has multiple engines (at least on newer platforms) which can run in parallel. So we need to have some means of synchronization between them. To do that the kernel not only keeps track of the seqno of the last batch an object has been used for, but also of the engine (commonly just called ring in the kernel, since that's what the kernel really cares about).
 
If a batchbuffer then uses an object which is still busy on a different engine, the kernel inserts a synchronization point: Either by employing so called hardware semaphores, which similarly to when the kernel needs to wait for the gpu simply wait for the correct seqno to appear, only using internal registers instead of the status page. Or if that's disabled, simply by blocking in the kernel until rendering completes. To avoid inserting too many synchronization points the kernel also keeps track of the last synchronization point for each ring. For ring/ring synchronization we don't track read domains separately, at least not yet.
  
Note that a big difference of GEM compared to a lot of other gpu execution management frameworks and kernel drivers is that GEM does not expose explicit sync objects/gpu fences to userspace. A synchronization point is always only implicitly attached to a buffer object, which is the only thing userspace deals with. In practice the difference is not big, since userspace can have equally fine-grained control over synchronization by keeping onto all the batchbuffer objects - keeping them around until the gpu is done with them won't waste any memory anyway. But the big upside is that when sharing buffers across processes, e.g. with DRI2 on X or generally when using a compositor, there's no need to also share a sync object: Any required sync state comes attached to the buffer object, and the kernel simply Does The Right Thing. 
 
This concludes the part about command submission, active object retiring and
synchronization. In the next installment we will take a [closer look at how the
kernel manages the GTT](/2012/11/i915gem-crashcourse-part-3.html), what happens
when we run out of space in the GTT and how the i915.ko currently handles
out-of-memory conditions.
