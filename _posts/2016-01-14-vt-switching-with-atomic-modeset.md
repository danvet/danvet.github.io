---
layout: post
title: VT Switching with Atomic Modeset
date: '2016-01-14T05:55:00.000-08:00'
author: danvet
tags:
- In-Depth Tech
modified_time: '2016-01-14T05:55:36.328-08:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-8106885102788327740
blogger_orig_url: http://blog.ffwll.ch/2016/01/vt-switching-with-atomic-modeset.html
---

First the title is a slight lie, this really is about compositor switching and not necessarily about using Linux VTs for that. But I hope that the title draws in the right folks and tempts them to read this. Since with atomic there's a bit a problem if you want to switch between different compositors - maybe you have X running and hack on wayland-mutter, or kwin and mutter or just a DE and a login manager&nbsp; - and expect it to not end up in a modern arts project like [this](https://twitter.com/__damien__/status/676741658940186625).

<!--more-->

Now the trouble with atomic modesetting and switching between different compositors is that atomic display updates are incremental for two reasons:

<ul><li>First we need to be able to support the legacy KMS interface for existing userspace, and that inteface was only updating parts of the overall display state.</li><li>Second we want atomic to be extensible. Which means compositors must be able to only update the properties they understand and leave everything else at a presumably reasonable default values.</li></ul>But if you mix this with a bunch of different compositors which all understand different subsets of all the atomic extensions a driver supports, suddenly the assumption that unhandled values have reasonable settings becomes invalid, and partial updates become a problem. Recently there's been a discussions on mailing lists and IRC about how to solve this, which ultimately ended in the conclusion that us kernel folks don't really know what would work best for distros, desktop environments and their compositors. Just going ahead with some new kernel ABI could easily result in a mistake that we have to support for the next 10 years. Therefore this blog post here covers the ideas with come up with to tackle this, just to make it clear that kernel folks are aware of this gap. As soon as userspace people with real clue about this topic run into problems they're more than welcome on dri-devel@lists.freedesktop.org and then we can figure out what to implement.



With that out of the way, let's look at possible solutions.



<b>FBDEV resets to defaults</b>



This is the cheap cop-out that is essentially implemented right now - when switching to a kernel console running on top of the FBDEV emulation KMS drivers can provide, the driver resets atomic properties of new extensions (like rotation, color management, Z-position/alpha/blending, whatever) to hopefully sane defaults. That's good enough for developers hacking around on different compositors, as long as you remember to VT-switch to a kernel console after things went south.



But FBDEV is seriously uncool, and a lot of people are working towards removing the kernel's VT subsytem from modern distros, too. This doesn't really work everywhere, but it's kinda the minimal and what we'll definitely implement. This has also the downside that maybe you only want to restore some properties, while keeping others (since they might be crucial to your setup, for example rotating the screen).



<b>System compositor restores boot-up state</b>



If doing something in the kernel isn't flexible enough then the usual approach is to do it in userspace. Most systems have some kind of master compositor that's run in-between user sessions, like a login manager. That system compositor could restore modeset state to something sensible every time it runs again, and user session compositors could then take over that sensible setup as their starting point.



This has the benefit that more clever and specific stuff like only restoring some properties is easy to implement. This shouldn't need a hole lot of code since a compositor needs to be able to restore it's state anyway to allow switching between them.



But, you're saying, how can a system compositor know about all the possible and future atomic KMS extensions? Won't this scheme break the much heralded extensibility? Well the neat trick is that userspace doesn't need to be able to understand properties to save and restore them - the actual property value transport between kernel and userspace is fully generic. There are a few special cases, like the need to disable outputs that have been unplugged meanwhile, and also some properties with special meaning, like framebuffers. But generic userspace can read out all the metadata, and even if future property types extend e.g. the value range that should still work.



The downsides of this approach is that it depends upon the global system compositor to do this right, so if that crashes and leaves your display setup in shambles you can't easily recover any more. This also requires that everyone who cares about switching between different compositors to have such a system compositors.



<b>Interlude: Only launching compositors matters</b>



The above observation that atomic clients can always faithfully restore a state (even if they don't understand the semantics of all properties) means that switching compositors itself will always work. The real trouble is making sure that a compositor can start up with a sane enough configuration to be able to successfully get pixels onto the screen. 



<b>New atomic IOCTL kernel flag</b>



What might be needed is a way to make this safe state persistent in the kernel. The next option tries that, by adding a new flag to the atomic IOCTL which asks the kernel to start out with an atomic KMS state reset to some default value.



But again the trouble here is, like with the FBDEV approach, that it's monolithic, and doesn't easily allow userspace to control what's being restored. And again the question is, should things get restored to the boot-up state (and which boot-up state - something equivalent to what FBDEV emulation would pick, what the firmware would have picked or a mix), or maybe reset values (set everything to unrotated) is better?



On top of that most often compositors don't want to reset state at all, to be able to smoothly take over the display configuration from the preceeding KMS client. Users have lost pretty much all appreciation of unsightly flickering that commonly happened in the pre-KMS world when switching compositors.



Another problem with keeping the boot-up state around is that the kernel then needs to keep a copy of all such state (and some objects like gamma tables are sizeable) around. Just in case there's userspace around to ask for it.



<b>Use SysRq to reset atomic state</b>



A problem with adding a flag to the atomic IOCTL to reset state is that all compositors need to implement support for it, and somehow make a decision for when to employ it. But on most systems compositors shouldn't completely mess up the atomic state. Most likely that's after a crash, and then there's no userspace around anyway to fix things up. Now generally this is, or well should, only be a problem for developers, and a possible solution might be to wire up a SysRq hotkey where the kernel force-resets atomic state to defaults. This would be similar to the FBDEV based solution, except without FBDEV and not tied to VT switching.



An alternative would be to implement this in the boot-splash service, by sampling boot-up state and providing some command to force-reset to that. But that's pretty much the system compositor approach, but using the boot splash, and a tool to restore its state from a stored location, as the system compositor.



<b>Expose reset or boot-up state</b>



An easy fix to give control back to userspace over what will get restored is to instead expose the boot-up values or the reset values to userspace, through an extension to the GET_PROPERTY IOCTL. But again storing boot-up state in the kernel would be wasteful on systems that will never need it (like Android or CrOS), and exposing reset values somewhat pointless if e.g. you really want your screen rotated, always.



<b>Per-compositor atomic state</b>



A slight spiel on all this is to make atomic state per-compositor in the kernel. This sounds a bit like it might help, but on the other hand implementing full state restore isn't more effort when compositors need to restore their state after a VT switch anyway. This leaves the tricky question of what the inherited state should be when a new compositor starts up: Normally you want the current state, so that the compositor can take over smoothly. Except when that's a really bad idea because the current state is badly mangled from a different compositor that just crashed.



Overall lots of different approaches and ideas, but no clear winner. That's why kernel folks need distro, compositor and desktop people to run into this issue first, to make sure the solution that lands actually solves the right problem. And in a way that suits userspace.



Thanks to Daniel Stone, Pekka Paalanen and Ville<span style="font-weight: normal;"><span class="gD" name="Ville Syrjälä"> Syrjälä for input on this.</span></span>