---
layout: post
title: How do you do docs?
date: '2016-12-22T06:00'
author: danvet
tags: 
- Maintainer-Stuff
---
The fancy new [Sphinx](http://www.sphinx-doc.org/)-based documentation has
landed a while ago in upstream. Jani Nikula has written a [nice overview on
LWN](https://lwn.net/Articles/692704/) ([part
2](https://lwn.net/Articles/692705/)). And it is getting used a lot. But judging
by how often I type it in replies on the mailing list what's missing is a
super-short howto. To build the documentation, run:

	$ make DOCBOOKS="" htmldocs

The output can then be found in Documentation/output/. When typing documentation
please always check that your new text does get rendered. The output also
contains documentation about kernel-doc and the toolchain itself. Since the
build is incremental it is recommended that you first run it before touching
anything.  That way you'll only see warnings in areas you've touched, not all of
them - the build is unfortunately somewhat noisy.

*Update*: of course also check out the nice [documentation on
kernel-doc](https://dri.freedesktop.org/docs/drm/doc-guide/index.html) itself.
