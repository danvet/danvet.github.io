---
layout: post
title: i915/GEM Crashcourse, Part 3
date: '2012-11-28T14:36:00.000-08:00'
author: sima
tags:
- In-Depth Tech
modified_time: '2013-07-21T06:58:51.504-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-7810278808977755614
blogger_orig_url: http://blog.ffwll.ch/2012/11/i915gem-crashcourse-part-3.html
---


In previous installments of this series we've looked at
[how the gpu can access memory](/2012/10/i915gem-crashcourse.html) and
[how to submit a workload to the gpu](/2012/11/i915gem-crashcourse-part-2.html). Now we will
look at some of the corner cases in more detail. See the [i915/GEM crashcourse
overview](/2013/01/i915gem-crashcourse-overview.html) for links
to the other parts of this series.  <!--more--> 

## GEM Memory Management Details

 
First we will look at the details of managing the gtt address space. For that the drm/i915 driver employs the <code>drm_mm</code> helper functions, which serves as a chunk allocator from a address space with a fixed size. It allows to constrain the alignment of an allocated block and also supports crazy gpu-specific guard-page constraints through an opaque per-allocation-block color attribute and a callback. Our driver uses that to ensure that so called snoopable and uncached regions on older generations are well-separated (by one page), since the hw prefetcher on some chip functions will fall over if it encounters the wrong type of memory while prefetching into a subsequent page. 
 
Now let's look at how GEM handles memory management corner cases. There are two important ways of running out of resources in GEM: There could be not enough space in the gtt for a given batch, this is the <code>ENOSPC</code> case in the code. Or we could run out of system memory (<code>ENOMEM</code>). One of the nice things of i915 GEM is that we use the kernel's shmemfs implementation to allocate backing storage for gem objects. The big benefit is that we get swap handling for free. A bit a downside is that we don't have much control over how and when swapping happens, and also not really any control over how the backing storage pages get allocated. These downsides regularly result in some noises on irc channels about implementing a gemfs to fix these deficiencies, but with no patches merged thus far. 
 

## Caching GEM Buffer Objects

 
The first complication arises because GEM driver stack has a lot of buffer object caches. The most important object cache is the userspace cache in libdrm. Since setting up a gem object is expensive (allocating the shmemfs backing storage, but also cache flushing operations are required on some generations and setting up memory mappings on first use) it's advisable to keep them around for reuse. And submitting workloads to the gpu tends to require tons of temporary buffers to upload data and assorted gpu-specific state like shaders, so those buffers get recycled quickly.  
 
Now if we run out of memory it would be good if the kernel can just drop the backing storage for these buffers on the floor instead of wasting time trying to swap their data out (or even failing to allocate memory, resulting in OOM-induced hilarity). Hence gem has a <code>gem_madvise</code> ioctl which allows a userspace cache to tell the kernel when a buffer is only kept around opportunistically with the <code>I915_MADV_DONTNEED</code>flag. The kernel is then allowed to reap the backing storage of such marked objects anytime it pleases. It can't though destroy the object tehmselves, since userspace still has a handle to them. Once userspace wants to reuse such a buffer, it mush tell the kernel by setting the <code>I915_MADV_WILLNEED</code> flag through the madvise ioctl. The kernel confirms that the object's backing storage is still around or whether it has been reaped. In the latter case userspace frees the object handle to also release the storage occupied by that in the kernel. If the userspace cache encounters such a reaped object it also walks all its cache buckets to free any other reaped objects. 
 
The kernel itself also tries to keep objects around on the gpu for as long as possible. Any object not used by the gpu is on the <code>inactive_list</code>. This is the list the kernel scans in LRU order when it needs to kick objects out of the gtt to free space for a different set of objects (e.g. for the ENOSPC case). In additions the kernel also keeps a list of objects which are not bound, but for which the backing storage is still pinned (and hence not allowed to be swapped out) on the <code>unbound_list</code>. This mostly serves to mitigate costly cache flushing operations, but the pinning of the backing storage is itself not free. We will talk later about why cache flushing is required in the coherency section in the next installment of this series. 
 

## Handling ENOSPC

 
Running out of gtt space requires us to select some victim buffer objects. Then wait for all access to the selected objects through the gtt to cease and then unbind them from the gpu address space to make room for the new objects. So let's take a closer look at how GEM handles buffer eviction from the gtt when it can't find a suitable free hole. Now setting up and tearing down pagetable entries isn't free, buffer objects vary a lot in size and for some access paths through the gtt we can only use the comparatively small mappable section of it. So just evicting objects from the inactive list in LRU order would be really inefficient, since we could end up unbinding lots of unrelated small objects sitting all over the place in the gtt, until we accidentally free a suitably large hole for a really big object. And if that big object needs to be in the contended mappable part, things are even worse. 
 
To avoid  trashing the gtt so badly GEM uses a eviction roaster, which is implemented with a few helper functions from the <code>drm_mm</code> allocator library: In a first step it scans through the inactive list and adds objects to the eviction roaster until a big enough free hole can be assembled. Then it walks the list of just scanned objects backwards to reconstruct the original allocator state and while doing that also marks any objects which fall into the selected hole as to-be-freed. In the last step it unbinds all the objects which are marked as to-be-freed, creating a suitable hole with the minimal amount of object rebinding. 
 
If no hole can be found by scanning the inactive list the eviction logic will also scan into the list of objects still being used by the gpu engines. Obviously it then needs to block for rendering to complete before it can unbind these objects. And if that doesn't work out, GEM returns <code>-ENOSPC</code> to userspace which indicates a bug in either GEM or the userspace driver part: Userspace must not construct batchbuffers which cannot fit into the GTT (presuming it's the only user of the gpu), and GEM must ensure that it can move anything else out of the way to execute such a giant batchbuffer. 
 
Now this works if we only need to put a single object into the gtt, e.g. when serving a pagefault for a gtt memory mapping. But when preparing a batchbuffer for command submission we need to make sure that all the buffer objects from an execbuf ioctl call are in the gtt. So we need to make sure that when reserving space for later objects, we don't kick out earlier objects which are also required to run this batchbuffer. To solve this, we need to mark objects as reserved and not evict reserved objects. In i915 GEM this is done by temporarily pinning objects, which every once in a while leads to interesting discussion because the semantics of such temporarily pinned buffers get mixed up with the semantics of buffers which are pinned for an indeterminate time, like scanout buffers. 
 
An funny exercise for the interested reader is to come up with optimal reservations schemes that allocate the gtt space for all buffers of an execbuf call at once (instead of buffer-by-buffer). This is mostly relevant on older generations which have sometimes rather massive alignment constraints on buffers. And create clever approximations of such optimal algorithms, which drm/i915 hopefully implements lready - otherwise patches highly welcome. Another good thing to ponder is the implications of a separate VRAM space (in combination with a GTT) which requires even more costly dma operations to move objects in, but in return has also much higher performance (i.e. the usual discrete gpu setup). One of the big upsides of a UMA (unified memory architecture) design as used on Intel integrated graphics is that there's no headache-inducing need to balance VRAM vs. GTT usage, which needs to take into account dma transfer costs and delays ... 
 

## Handling ENOMEM

 
Handling low memory situations is first and foremost done by the core vm, which tries to free up memory pages. In turn the core vm can call down int GEM (through the shrinker interface) to make more space available. 
 
Within the drm/i915 driver out of memory handling has a few tricks of its own, all of which are due to the rather simplistic locking scheme employed by drm/i915 GEM: We essentially only have one lock to protect all important GEM state, the <code>dev->struct_mutex</code>. Which means that we need to allocate tons of memory when holding that lock, in turn requiring that our memory shrinker callback only trylocks that all-encompassing mutex, since otherwise we might deadlock. We also need to tell the memory allocations functions through GFP flags that they shouldn't endlessly retry allocations and even launch the OOM killer, but instead fail the allocation if there's no memory and return immediately. Since most likely GEM itself is sitting on tons of memory we then try unpin a bunch of objects in such failure cases and retry the allocation ourselves. If that doesn't help, or if there's no buffer object that can be unpinned, we give up and return <code>-ENOMEM</code> to userspace. One side-effect of the trylock in the shrinker is that if any other thread than the allocator is hoggin our GEM mutex, we won't be able to unpin any gem objects, potentially causing a unnecessary OOM situation. 
 
Despite that this is rather hackish approach and not really how memory management is supposed to work, it seems to hold up rather well in reality. And recently (for kernel 3.7 and 3.8) Chris Wilson added some additional clever tricks to make it work for a bit longer when the kernel is tight on free memory. But in the long-term I expect that we need to revamp our locking scheme to be able to reliable free memory in low-memory situations. 
 
Another problem with the current low-memory handling is that the shrinker infrastructure is desinged to handle caches of equally-sized (filesystem) objects. It doesn't have any notion of objects of massively different sizes, which we try by reporting object counts in aggregate number of pages. And it also doesn't expect that when a shrinker runs, tons of memory suddenly shows up in the pagecache - which is what happens when we unpin the buffer object backing storage and give control over it back to shmemfs. 
 
In the next installement we will take a closer look at coherency issues and optimal ways to transfer data between the cpu and gpu. 
