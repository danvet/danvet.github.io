---
layout: post
title: "Maintainer Statistics"
tags:
- Maintainer-Stuff
---
As part of my last two talks at LCA on the kernel community, ["Burning Down the
Castle"](/2018/02/lca-sydney.html) and ["Maintainers Don't
Scale"](/2017/01/maintainers-dont-scale.html) I have looked into how the Kernel's
maintainer structure can be measured. One very interesting approach is looking
at the pull request flows, for example done in the LWN article ["How 4.4's
patches got to the mainline"](https://lwn.net/Articles/670209/). What I'm trying
to work out here isn't so much the overall patch flow, but focusing on how
maintainers work, and how that's different in different subsystems.

<!--more-->

## Methodology

In my presentations I claimed that the kernel community is suffering from too
steep hierarchies. And worse, the people in power don't bother to apply the same
rules to themselves as anyone else, especially around purported quality
enforcement tools like code review.

For our purposes a **contributor** is someone who submits a patch to a mailing
lists, but needs a maintainer to apply it for them, to get the patched merged. A
**maintainer** on the other hand can directly apply a patch to a subsystem
tree, and will then send pull requests up the maintainer hierarchy until the
patches land in Linus' tree. This is relatively easy to measure accurately in
git: If the recorded patch author and committer match, it's a maintainer commit,
if they don't match it's a contributor commit.

There's a few annoying special cases to handle:

- Some people use different mail addresses or spellings, and sometimes MTAs,
  patchwork and other tools used in the patch flow chain mangle things further.
  This could be fixed up with the mail mapping database that e.g. LWN uses to
  generate its contributor statistics. Since most maintainers have
  reasonable setups it doesn't seem to matter much, hence I decided to not
  bother.

- There's subsystems not maintained in git, but in quilt. Andrew Morton's tree
  is the only one I'm aware of, and I hacked up my scripts to handle this case.
  After that I realized it doesn't matter, since Andrew merged exceedingly few
  of his own patches himself, most have been fixups that landed through other
  trees.

Also note that this is a property of each commit - the same person can be both
a maintainer and a contributor, depending upon how each of their patches gets
merged.

The ratio of maintainer commits compared to overall commits then gives us a
crude, but fairly useful metric to measure how steep the kernel community
overall is organized.

Measuring review is much harder. For contributor commits review is not recorded
consistently. Many maintainers forgo adding an explicit *Reviewed-by* tag since
they're adding their own *Signed-off-by* tag anyway. And since that's required
for all contributor commits it's impossible to tell whether a patch has seen
any oversight before merging. A reasonable assumption though is that maintainers
actually look at stuff before applying.

A different story are maintainer commits - if there is not tag indicating
review by someone else, then either it didn't happen, or the maintainer felt
it's not important enough work to justify the minimal effort to record it.
Either way, a patch where the git author and committer match, and which sports
no review tags in the commit message, strongly suggests it has indeed seen none.

An objection would be that these patches get reviewed by the next maintainer
when the pull request gets merged. But there's well over a thousand such patches
each kernel release, and most of the pull requests containing them go directly
to Linus in the 2 week long merge window. It is unrealistic to assume that Linus
carefully reviews hundreds of patches himself in just those 2 weeks, while
getting hammered by pull requests all around. The same applies at a subsystem
level.

For counting reviews I looked at anything that indicates some kind of patch
review, even very informal ones. I therefor both included *Reviewed-by* and
*Acked-by* tags, including a plethora of misspelled and combined versions of the
same.

The scripts also keep track of how pull request percolate
up the hierarchy, which allows filtering on a per-subsystem level. Commits in
topic branches are accounted to the subsystem that first lands in Linus' tree.
That's fairly arbitrary, but simplest to implement.

## Last few years of history

history

- declining maintainer commitsin kernel overall, accelarated without graphics ->
  we're dead by 2025
- review going up, but very slowly
- gfx: massive growth, but maintainer ratio roughly stable
- gfx: doing a decent, but not perfect job on review

## v4.16 by Subsystem

zooming into 4.16

- commit rights: very dire

- review:
- xfs has their shit together. "interesting" spread in vfs
- where are the other big subsystems (500+ commits)?
staging: 7%, networking 9%, tip: 10%, arm-soc: 50%
