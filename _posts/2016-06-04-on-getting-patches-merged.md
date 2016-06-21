---
layout: post
title: On Getting Patches Merged
date: '2016-06-04T04:11:00.000-07:00'
author: danvet
tags:
- Maintainer-Stuff
modified_time: '2016-06-12T13:45:05.243-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-4783717525266737585
blogger_orig_url: http://blog.ffwll.ch/2016/06/on-getting-patches-merged.html
---

In some project there's an awesome process to handle newcomer's contributions -
autobuilder picks up your pull and runs full CI on it, coding style checkers
automatically do basic review, and the functional review load is also at least
all assigned with tooling too.

Then there's projects where utter chaos and ad-hoc process reign, like the Linux
kernel or the X.org community, and it's much harder for new folks to get their
foot into the door. Of course there's documentation trying to bridge that gap,
tools like get_maintainers.pl to figure out whom to ping, but that's kinda the
details. In the end you need someone from the inside to care about what you're
doing and guide you through the maze the first few times.

I've been pinged about this a few times recently on IRC, so I figured I'll type
up my recommended best practices.

<!--more-->

The crucial bit is that such unstructured developer communities run entirely on
mutual trust, and patches get reviewed through a market of favours as in "I
review yours and you review my patches". As a newcomer you have neither. The
goal is to build up enough trust and review favours owed to you to get your
patches in.

- Write a few small, simple patches in related areas. When developing your
  patches you probably had to read some code, documentation, look at testcases,
  and very likely you spotted a small thing or two that could be improved.
  Create a patch for each of those and submit them - it's a great way to
  test-drive the process and make first contact. Some projects even have tools
  to hunt for trivial things, e.g. Linux' checkpatch.pl. And as a maintainer I
  try hard to give everyone a chance to land their first few simple patches fast
  and try to review&amp;merge them within days.

- Talk about what you're doing on IRC or whatever your project uses for
  fast&amp;informal communication. Interactive communication is much faster at
  clearing up initial misunderstandings compared to mailing lists and similar
  things like pull request discussions. It's also more scary, but let's be
  honest: If the community you want to contribute too isn't open&amp;friendly,
  you probably don't want to stick around anyway. In short, it's a good
  community test too.

- Go the extra mile: If you create a new interface, helper function or whatever
  try to roll it out everywhere. Even in code you can't run yourself (like
  drivers for other hardware or similar things), or when you're unsure about how
  to convert a given piece of code to the new way of doing things. If you have
  no idea what other piece could benefit from your work, ask around - that's why
  you started out with talking on IRC after all. The benefit is that by touching
  lots of other codes you automatically also gain lots more reviewers who might
  be interesting in your work and help push it forward.

- When submitting your work, or resubmitting revised versions, make sure that
  everyone who showed interest or might be interested stays informed. There's
  various support in tooling for this, but with git and mailing-list patch
  submissions I just add everyone who ever commented on a patch, or who's
  reviewer or maintainer for an area with a Cc: entry to the patch itself. That
  way I never forget to add them when resubmitting.

- Doing all this you will have learned a lot - apply that knowledge by reviewing
  patches from people who commented on your work in related areas. That will
  also fill your review favours kitty and help get your patches landed.

- If the project uses the maintainer model for committing with no  commit rights
  for regular contributors, don't ask the maintainer for  review, they're
  generally overloaded. Instead fan out and try to find  the subject expert,
  like you would do in a project where all regular  contributors have commit
  rights.

- Most important of all: Don't just sit&amp;wait around scared and hope your
  patches will magically land. They won't, not because they're bad but simply
  because people are busy, and your patches are forever lost in the chaos after
  just a week or two. Hence ping after a few days about the status, but don't
  ask when the patches will land, but instead what you can do to move them
  forward. It's a cheap trick, but it helps to elicit useful responses, at least
  in my experience.

- And equally important: Don't fall into the imposter syndrome gap or end up
  blaming yourself when things are a bit bumpy at first - it's just plain hard
  to figure out undocumented rules of projects who run on chaos and personal
  relationships. But do try to improve the documentation once you've managed all
  the pitfalls, to make it easier for the next new person. After all, as the
  most recent new contributor, you're now the residential expert on this topic!

- And finally your patches have landed, and in the process you've learned to
  know a few interesting people. And getting to know new folks is in my opinion
  really what open source and community is all about. Congrats, and all the best
  for the next step in your journey!

And finally for the flip side, there's a great write up from <a
href="http://sarah.thesharps.us/2014/09/01/the-gentle-art-of-patch-review/">Sarah
Sharp about doing review</a>, which applies especially to reviewing newcomer's
patches.
