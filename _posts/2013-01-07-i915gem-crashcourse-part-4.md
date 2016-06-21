---
layout: post
title: i915/GEM Crashcourse, Part 4
date: '2013-01-07T07:56:00.001-08:00'
author: danvet
tags:
- In-Depth Tech
modified_time: '2013-07-21T06:58:51.488-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-1825930399780826244
blogger_orig_url: http://blog.ffwll.ch/2013/01/i915gem-crashcourse-part-4.html
---


In the previous installment we've taken a closer look at some <a href="http://blog.ffwll.ch/2012/11/i915gem-crashcourse-part-3.html">details of the gpu memory management</a>. One of the last big topics now still left are all the various caches, both on the gpu (both render and display block) and the cpu, and what is required to keep the data coherent between them. Now one of the reasons gpus are so fast at processing raw amounts of data is that caches are managed through explicit instructions (cutting down massively on complexity and delays) and there are also a lot of special-purpose caches optimized for different use-cases. Since coherency management isn't automatic, we will also consider the different ways to move data between different coherency domains and what the respective up- and downsides are. See the <a href="http://blog.ffwll.ch/2013/01/i915gem-crashcourse-overview.html">i915/GEM crashcourse overview</a> for links to the other parts of this series.  
<!--more--> 

## Interactions with CPU Caches

 
The first thing to look at is how gpu-side caches work together with the cpu caches. Traditionally the intel gfx has been sitting right on top of the memory controller. For maximum efficiency it therefore did not take part in the coherency protocol with the cpu caches, but had direct read/write access to the underlying memory. To ensure that data written by the cpu could be read by the gpu coherently, we need to flush cpu caches and further the chipset write queue on the separate memory controller. Only once the data is guaranteed to have reached physical memory can it be read by the gpu. 
 
For readbacks, e.g. after the gpu has rendered a frame, we again need to invalidate all the cpu caches - the gpu writes don't snoop cpu caches and so the cpu cache might still contain stale data. On intel hw this is again done by the clflush instruction, which (only on Intel cpus though) guarantees that clean cachelines will simply be dropped and not written back to memory. This is important, since otherwise data in memory written by the gpu could be overwritten by stale cache contents from the cpu. 
 
Now on newer platforms (Sandybridge and later) where the gpu is integrated with the cpu on the same die, the render portion of the gpu is sitting on top of the last level caches, together with all the cpu cores. Current architecture is that the cpu cores (including L2 caches), gpu (again including gpu specific caches), memory controller and the big L3 cache are all connected with a coherent ring interconnect. Since then the gpu can profit from the big L3 cache on the die, giving a decent speedup for gpu workloads. Hence it is now beneficial for gpu reads/writes to be coherent with all the cpu caches - explicitly flushing cachelines is cumbersome and the last-level cache can easily hide the snooping latencies. 
 
The odd thing out on this architecture is the display unit. To preserve the most power in idle situation we want to be able to turn off power for cpu cores, the gpu render block and the entire L3 and interconnect (called uncore). Which means the always-on display block can only read from memory directly and can't be coherent with caches. To allow this we can specify the caching mode in the gtt pagetables and so mark render targets which will be used as scanout sources uncacheable. But that also implies that they're no longer coherent with cpu cache contents, and so we need to again engage in manual cacheline flushing like on the platforms without a shared last level cache. 
 

## Transferring Data Between CPU and GPU Coherency Domains

 
The oldest way to move data between the gpu and cpu in the i915 GEM kernel driver was with the <code>set_domain</code> ioctl. If required, it flushed the cpu caches for all the cachelines for the entire memory address range of the object. Which is really expensive, so we need more efficient ways to move date from/to the gpu, especially on platforms without a coherent last level cache. Note that all objects start out in the cpu domain, since freshly-allocated pages aren't necessarily flushed. Hence it is important that userspace keeps around a cache of already flushed objects, to amortize the expensive initial cacheline flushing. Also note that the <code>set_domain</code> ioctl is also used to synchronize cpu access with outstanding gpu rendering, and so still used in that role for coherent objects. 
  
For this the kernel and older hw support a snooping mode, where the gpu asks the cpu to clear any dirty cachelines before reading/write. The upside is that we don't blow through cpu time flushing cachelines, and the gpu seems to be more efficient at moving data out of and into cpu caches anyway. The downside is, at least on older generations, that we can't use such snooped areas for rendering, but only to offload data transfers from/to the cpu to gpu. But since there are now downsides for the cpu in accessing snooped gem buffer objects they are very interesting for mixed sw/hw rendering. Another upside is that the uploads can be streamed and are asynchronously done, so avoiding stalls. Downloads can obviously also be done asynchronously using the gpu, but additional unrelated queued rendering could introduce stalls since the cpu often needs the data right away. 
  
Hence the tradeoffs for picking the gpu for copieding data from/to snoopable buffers compared to using the cpu to transfer data is tricky: For uploads it's usually only worth it to use the gpu to avoid stalls, since our cpu uploads paths are fast. For downloads on platforms without a shared last-level cached gpu copies using snoopable buffers beats the cpu readbacks hands-down. 
 
The i915 GEM driver exposes a set of ioctls to allow userspace to read/write linear buffers which try to pick the most efficient way to transfer the data from/to the gpu coherency domain, called <code>pwrite</code> and <code>pread</code>. We'll later on look at some of the trade-offs and tricks employed, but for now only discuss how reads/writes which only partially cover a cacheline need to be handled if we write through the cpu mappings. Reads are simple, every cacheline we touch needs to be flush before reading data written by the gpu. For writes we only need to flush before writing if the cacheline is partially covered - stale data from cpu caches in that cacheline could otherwise be written to memory. For writes covering entire cachelines we can avoid this. Hence userspace tries to avoid such partial writes. Obviously we also need to flush the cacheline after writing to it, to move the data out to the gpu. 
 
All this flushing doesn't apply when the buffer is coherent, and generally on platforms with a last-level cache writing through the cpu caches is the most efficient way to copy data with the cpu (the gpu can obviously also copy things around, since it doesn't pay a penalty for coherent access on such platforms). Going through the GTT to e.g. avoid manually tiling and swizzling a texture is much slower. The reason for that is that GTT writes land directly in main memory and don't go into the last-level cache - hence we're limited by main memory bandwidth and can't benefit from the fast caches. It does though issue a snoop to ensure that everything is still coherent. 
 
On platforms without a shared last-level cache it's a completely different story though - we have to manually flush cpu caches, which is a rather expensive operation. GTT writes on the other hand bypass all caches and land directly in main memory. Which means that on these platforms GTT writes are generally 2-3x faster than writes through the cpu caches - a notch slower if the GTT area written to is tiled. GTT reads are in all cases extremely slow since the GTT I/O range is only write-combined, but uncached. 
 
Now one of the clever tricks that the pwrite ioctl implements is that on platforms where GTT writes are faster it tries to first move the buffer object into the mappable part of the GTT (if the object isn't there yet and this can be done without blocking for the gpu). This way the upload benefits from the faster GTT writes, which easily offsets any costs in unbinding a few other buffer objects and rewriting a bunch of PTEs. Especially since on recent kernels and most platforms GTT PTEs are mapped with write-combining. 
 
Summarizing the data transfer topic, we either have full coherency (where cpu cached memory mappings are the most efficient way to transfer data, with no explicit copying involved). Or we can copy data with the gpu (using snooped buffer objects), or with the cpu (either through memory mappings of the GTT or through the pwrite/pread ioctls). The kernel also supports explicit cpu cache flushing through the <code>set_domain</code> ioctl, which allows userspace to use cpu-cached memory maps even on platforms without coherency. But they're abysmally slow and hence not used in well-optimized code. 
 

## GPU Caches

 
Up to now we've only looked at cpu caches and presumed that the gpu is an opaque black box. But like already explained, gpus have tons of special-purpose caches and assorted tlbs, which need to be invalidated and flushed. First we will go trough the various gpu caches and look at how and when they're flushed. Then we'll look at how gpu cache domains are implemented in the kernel. 
 
Display caches are very simple: Data caches are always fifos, so never need to be flushed. And the assorted TLBs are in general invalidated on every vblank. The cpu mmio window into the GTT is also rather simple: No data caches need to be explicitly managed, only the tlb needs to be explicitly flush when updating PTEs by writing to a magic register afterwards. 
 
Much more complicated are the various special-purpose caches of the render core. Those all get flushed either explicitly (by setting the right bit in the relevant flush instruction) or implicitly as a side-effect of batch/ringbuffer commands. Important to note is that most of these caches are not fully coherent (safe for the new L3 gpu cache on Ivybridge), so writes do not invalidate cachelines in cpu caches, even when the respective gtt PTE has the snoop/caching bits set. Writes only invalidate the cpu caches once a cacheline gets evicted from the gpu caches. Hence the command streamer instruction which flush caches also are coherency barriers. 
 
One of the cornerstones of the original i915 GEM design was that the kernel explicitly keeps track of all the caches (render, texture, vertex, ...) an object is coherent in. It separately kept track of read and write domains, to optimize away flush for r/w caches if no cacheline should be dirty. But the code ended up being complicated and fragile, and thanks to tons of workarounds we need to flush most caches on many platforms way more often than necessary. Furthermore, userspace inserted it's own flushes to support e.g. rendering to textures, so the kernel wasn't even tracking properly which caches are clean. On top of that, the explicit tracking of dirty objects on the <code>flushing_list</code> to coalesce cache flushing operations added unnecessary delays, hurting workloads with tight gpu/cpu coupling. 
 
In the end this all ended up being a prime example of premature optimization. So for most parts the kernel now ignores the gpu domain values in relocations entries and instead unconditionally invalidates all caches before start a new batch and then flushes all write caches after the batch completes, but before the seqno is written and signaled with the CS interrupt. The big exception is workarounds, e.g. on Sandybridge writes from the command streamer used mostly for GL queries need global GTT entries shadowing the PPGTT PTEs. But all the in-kernel complexity to track dirty objects and gpu domains is mostly ripped out. The last remaining thing is to collapse all the different GEM domains into a simple boolean to track whether an object is cpu or gpu coherent. 
 

##  Odds&amp;Sods 

 
This now concludes our short tour of the i915.ko GEM implementation for submitting commands to the gpu. Two bigger topics are not covered though: Resetting the gpu when is stuck somewhere and how fairness is ensured by throttling command submission. Both are results of the cooperative nature of command submission on current intel gpus - batchbuffers cannot be preempted. Hence interactive responsiveness and fairness requires cooperation from userspace. And a stuch batchbuffer can only be stopped by resetting the entire gpu due to lack of preemption support. 
 
In both areas new ideas and improvements are floating around, so I've figured it best to cover them when improvements to the gpu reset code or the way we schedule batchbuffers land. 
