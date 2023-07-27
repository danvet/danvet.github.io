---
layout: post
title: Better Markup for the Kernel GPU DocBook
date: '2016-01-10T15:00:00.000-08:00'
author: sima
tags:
- Maintainer-Stuff
modified_time: '2016-01-10T15:00:00.050-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-5274572225825971862
blogger_orig_url: http://blog.ffwll.ch/2016/01/better-markup-for-kernel-gpu-docbook.html
---

This summer Intel sponsored some work to improve the kerneldoc toolchain, with
the aim to use all that to extend the DRM and i915 driver documentation we have.
Most of it landed, but the last bit to integrate some type&nbsp; of text markup
processing was stalled until it could be discussed at the kernel summit, see the
[LWN summary](https://lwn.net/Articles/662930/" rel="nofollow). Unfortunately it
died in a bikeshed fest due to an alliance of people who think docs are useless
and you should just read the code, and others who didn't even know how to
convert the kerneldoc into something pretty.

But we still need this, since without lists, highlighting, basic tables and
inserting code snippets it's really hard to write decent documentation. Luckily
Dave Airlie is ok with using it for DRM kerneldoc as long as Intel maintains the
support. It's purely opt-in and the only downside of not using
<code>asciidoc</code> is that the resulting docs won't be as pretty. All the
changes to the text itself to use this markup are going into upstream as normal.
The only bit that's not in upstream is the tooling, which is available in a
topic branch at

	git://anongit.freedesktop.org/drm-intel topic/kerneldoc

If you want to build pretty docs just install <code>asciidoc</code> and base
your drm documentation patches on top of drm-intel-nightly from the same
repository - that tree also includes all of Dave's tree. Alternatively pull in
the above topic branch into your own personal tree.  Note that
<code>asciidoc</code> is detected automatically, so you really only need it and
the tooling branch to check the rendering of your changes.

For added convenience Intel also maintains an autobuilder that pushes latest
drm-intel-nightly DRM documentation builds to
[http://dri.freedesktop.org/docs/drm/](http://dri.freedesktop.org/docs/drm/"
rel="nofollow).

Aside: If all you want to build is just build the GPU DocBook instead of all of
them, you can do that with

	$ make DOCBOOKS="gpu.xml" htmldocs

With that have fun reading the new&amp;improved documentation, and if you spot
anything please submit a patch to dri-devel@lists.freedesktop.org. 
