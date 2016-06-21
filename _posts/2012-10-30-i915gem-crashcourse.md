---
layout: post
title: i915/GEM Crashcourse
date: '2012-10-30T14:37:00.000-07:00'
author: danvet
tags:
- In-Depth Tech
modified_time: '2013-07-21T06:58:51.495-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-4553523947071081750
blogger_orig_url: http://blog.ffwll.ch/2012/10/i915gem-crashcourse.html
---


This is the first part in a short tour of the Intel hardware and what the GEM (graphics execution manager) in the i915 does. See the [overview](http://blog.ffwll.ch/2013/01/i915gem-crashcourse-overview.html) for links to the other parts. 
<!--more-->
GEM essentially deals with graphics buffer objects (which can contain textures, renderbuffers, shaders or all kinds of other state objects and data used by the gpu) and how to run a given workload on the gpu, commonly called command submission (CS), but in the i915.ko driver done with the execbuf ioctl (since the gpu commands themselves reside in a buffer object on Intel hardware). 


## Address Spaces and Pagetables


So the first topic to look at is what kind of different address space we have, which different pieces of hardware can access them, and how we bind various pieces of memory into them (i.e. where the corresponding pagetables are and what they look like). Contrary to discrete gpus Intel gpus can only access system memory, hence the only way to make any memory available to the gpu is by binding a bunch of pages into one of these gpu pagetables, and we don't need to bother us with different kinds of underlying memory as backing storage. 
 
The gpu itself has its own virtual address space, commonly called GTT. On modern chips its 2GB big, and all gpu functions (display unit, render rings and similar global resources, but also all the actual buffer objects used for rendering) access the data they need through it. On earlier generations it's much smaller, down to a meager 32M for the i830M. On Sandybridge and newer platforms we also have a second address space called the per-process GTT (PPGTT for short), which is of the same size. This address space can only be accessed by the gpu engines (and even there we sometimes can't use it), hence scanout buffers must be in the global GTT. The original aim of PPGTT was to insulate different gpu processes, but context switch times are high, and up to Sandybridge the TLBs have errate when using different address spaces. The reason we use PPGTT now - it's been around as a hardware feature even on earlier generations - is that PPGTT PTEs can reside in cachable memory, and so lookups benefit from the big shared LLC cache, whereas GTT PTE lookups always hit main memory. 
 
But before we look at the where the pagetables reside, we also need to consider how the cpu can access graphics data. Since Intel has a unified memory architecture, we can access graphics memory by directly mapping the backing storage in main memory into the cpu address space. For a bunch of reasons that we'll discuss later on it is though useful if the cpu can access the graphics objects through the GTT. For that, the low part (usually the first 256MB) of the GTT can be accessed through the bar 2 pci mmio space of the igd. Hence we can point the cpu PTEs at that mmio window, and then point the relevant GTT PTEs at the actual backing storage in main memory. The chip block that forwards these cpu accesses is usually called System Agent (SA). Note that there's no such window for the PPGTT, that address space can really only be used by the render part of the gpu (usually called GT). This GTT cpu window is called the mappable gtt part in our code (since we can map it from the cpu) and accessed with write-combining (wc) cpu mapping. 
 
Now the pagetables are a bit tricky. In the end they're all in system memory, but there are a few hoops to jump through to get at them. The GTT pagetables has just one level, so with a 4 byte entry size we need 2MB of contiguous pagetable space. The firmware allocates that for us from stolen memory (i.e. a part of the system memory that is not listed in the e820 map, and hence not managed by the linux kernel). But we write these PTEs through an alias in the register mmio bar! The reason for that is to allow the SA to invalidate TLBs. Note though that this only invalidates TLBs for cpu access, any other access to the GTT (e.g. from the GT or the display block) has its own rules for TLB invalidation. Also, on recent generations we need to (depending upon circumstances) manually invalidate the SA TLB by writing to a magic register. To speed up map/unmap operations, we map that GTT PTE aliasing region in the mmio with wc (if this is possible, which means the cpu needs to support PAT). 
 
PPGTT has a two level pagetable. The PTEs have the exact same bit layout as the PTEs in the global GTT, but they reside in normal system memory. The PTE pagetables are split up into pages, each of which contains 1024 entries. So there's no need for a contiguous block of memory and they are hence allocated with the linux page allocator. The PDEs on the other hand are special. We need 512 of&nbsp; them to map the entire 2GB address space. For unknown reasons the hw requires that we steal these 512 entries from the GTT pagetable (hence the useable range of the global GTT shrinks by 2 MB) and write them through the same mmio alias that we write the global GTT PTEs through. 
 
A slight complication is VT-d/DMAR support, which adds another layer to remap the bus addresses that we put into (PP)GTT PTEs to real memory addresses. We simply use the common linux dma api through <code>pci_map/unmap_sg</code> (and variants) to use these, so DMAR support is rather transparent in our code. Well, safe for the minor annoyance that the DMAR hardware interacts badly with the GTT PTE lookups on various platforms and so needs ridiculous amounts of horrible workarounds (down to disabling everything on GM45 since it simply doesn't work). Only on Ivybridge DMAR seems to be truely transparent and no longer requires strange contortions.
 

## Tiling and Swizzling

 
Modern processors go to extreme lengths to paper over memory latency and lack of bandwidth with clever prefetchers and tons of caches. Now graphics usually deals with 2D data and that can result in some rather different access patterns. The pathalogical case is drawing a vertical line (if each horizontal line is contigous in memory, followed by the next one): Drawing one pixel will just access 4 bytes, then the next pixel is a few thousands bytes away. Which means we'll not only incur a cache miss, but also a TLB miss. Which for a 1 pixel line crossing the entire screen means a few hundred to thousands cache and TLB misses. For just 1-2 KB of date. No amount of caches and TLBs can paper over that.
 
So the solution is to rearrange the layout of 2D buffers into tiles of fixed size&amp;height, so that each tile has a size of 4KB. Hence as long as we stay within a tile, we won't incur a TLB miss. Intel hardware has two different tiling layouts:
 
<ul><li>X-tiled, which is 512 bytes wides with 8 rows, where each 512 byte long logical row is contiguous in memory. This is the compromise tiling layout which is somewhat efficient for rendering, but can still be used for scanout, where we access the 2D buffer row-by-row. A too short (or non-contigous) row within a tile would drive power consumption (due to the more random memory access pattern) for scanout through the roof, which hurts idle power consumption.</li><li>Y-tiled, which is 128 bytes times 32 rows. For the comon 32bit pixel layouts this means 32x32 pixels, so nicely symmetric in both x and y direction. For even better performance a row is split up in OWORDS (i.e. 16 bytes) and consecutive OWORDS in memory are mapped to consecutive <i>rows </i>(no columns, which would result in the contigous rows of pixels that the X-tiled layout has). Hence an aligned 4x4 pixel block matches up with a 64 byte cacheline. So in a way a cacheline within a Y-tile works like a smaller microtile.</li></ul> 
There's also a special W-tile layout, which is only used by the separate stencil buffer. Safe when hacking around in the stencil readback code, we can just ignore this (since accessing the stencil buffer is all internal to the render engine). Also, some really old generations have slightly different parameters for the X and Y layouts.
 
An additional complexity comes with dual channel memory. For efficiency reasons we want to read entire 64 byte block (which matches the cachelines size) from a single channel. To load balance between the two channels we therefore load all even cachelines from the first channel and all odd cachelines from the second channels. There are some additional trick to even out things, but this is the gist we need for the below discussion.
 
Unfortunately this channel interleave pattern together with the X tiling again leads to a pathalogical access pattern: If we draw a vertical line in an X tile we advance by 512 bytes for each pixel, which means we'll always hit the same memory channel (since <code>512 % 64 == 0</code>). The same happens for Y tiles when drawing a horizontal line. Looking at the memory address we see that bit 6 essentially selects which memory channel will be used, so we can even out the access pattern by XOR'ing additional, higher bits into bit 6 (which is called swizzling). XOR'ing bit 9 into bit 6 gives us a perfect checkerboard of memory channels for Y tiling. For X tile we can't get better than at most 2 consecutive cachelines in any direction that use the same memory channel, which the hw achieves by XOR'ing bit 10 and 9 into bit 6.
  
<b>Addendum:</b> On most older platforms the firmware selects the whether swizzling is enabled when setting up the memory controller and this can't be changed afterward any more. But on Sandybridge and later the driver controls swizzling, and enables it when it detects a symmetric DIMM configuration in the memory controller. 
  

## Fencing


Now how can we tell the gpu which layout a given buffer object uses? For pretty much most gpu functions we can directly specify the tiling layout in additional state bits (or for the display unit, in special register bits). But some functions don't have any such bits, and on older platforms that even includes the blitter engine and some other special units. For these the hardware provides a limited set of so called fences.
 
The first thing to note is that Intel hardware fences are something rather different than what they normally mean around gpu drivers: Usually a fence denotes a synchronization point that is inserted into the command stream that the gpu processes, so that the cpu knows exactly up to which point the gpu has completed commands (and hence also which buffers the gpu still needs).
 
Fences on Intel hardware though are special ranges in the global GTT that make a tiled area look linear. On modern platforms we have 16 of those and can set them up rather freely, the start and end of a tiled region only need to align to page boundaries. Now most gpu GTT clients have their own tiling parameters and ignore these fences (with the exception of older platforms, where fence usage by the gpu is much more common), but fences are are obeyed by the SA and hence by all CPU access that targets the mmio window into the GTT. Fences are therefore the first reason why cpu access to a buffer object through the GTT mmio window is useful - they detile a buffer automatically, and doing the tiling and swizzling manually is simply a big pain. One peculiarity is though that we can only access X and Y tiled regions, not W tiled regions - so stencil buffers need to be manually detiled and deswizzled with the cpu.
 
Managing the fences is rather straightforward - we keep them in an lru, and if no fence is free, we make one available by ensure that nothing that currently accesses this buffer objects really needs the fence. To ensure that the cpu can't access the object any more through the mmio window we shot down all cpu PTEs pointing at it. For older generations where the gpu also needs fences we keep track of the last command submission that requires it (since not all of them do, and fences are a limited resources) and wait for that to complete.
 
Also, on some older platforms fences have a minimal size and need to be a power of two in alignment and size. To avoid the need to over-allocate a texture (and so waste memory) we allow tiled objects to be smaller than the fenced region they would require. When setting up a fence we simply reserve the entire region that the fence detiles in the GTT memory manager, and so ensure that no other object ends up in the unused, but still detiled area (where it would get corrupted).
 
Now that we've covered how to make memory accessible to the gpu and how to handle tiling, swizzling and setting up fences, the [next installment](http://blog.ffwll.ch/2012/11/i915gem-crashcourse-part-2.html) of this series will look at how to submit work to the gpu and some related issues. 
 
<b>Errata:</b> Mika Kuoppala noticed that my math about the GTT PTE stealing for the PDEs used for PPGTT was off, now it's fixed.
