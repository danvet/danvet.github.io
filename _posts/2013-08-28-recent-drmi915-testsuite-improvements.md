---
layout: post
title: Recent drm/i915 Testsuite Improvements
date: '2013-08-28T02:56:00.001-07:00'
author: danvet
tags:
- In-Depth Tech
modified_time: '2013-08-29T09:54:33.765-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-7561193979125790593
blogger_orig_url: http://blog.ffwll.ch/2013/08/recent-drmi915-testsuite-improvements.html
---


So recently I've again wreaked decent havoc in our intel-gpu-tools kernel
testsuite. And then shockingly noticed that I've never done a big pompous blog
post to announce what we've started about one-a-half years ago. So besides just
describing the new infrastructure for writing testcases (apparently every decent
hacker must reinvent a testframework at least once ...) I'll also recap a bit
what's been going on in the past.

<!--more-->

## What Has Happened Thus Far ...


drm/i915 kernel testing was in a very sorry state two years ago - we've had
pretty much zero regression testing of corner cases. Of course we've implicitly
tested the kernel by running userspace drivers, e.g. testing mesa with piglit.
But that only exercise the best case code and not at all the fun stuff hidden
behind memory and other resource exhausting handling, racing multiple threads
(or the cpu against the gpu) or trying to provoke coherency issues. 

But there was a really basic set of manually run testcases in intel-gpu-tools.
So we've started with them and wrapped them up in a very hackish testrunner
using autotools and went ahead integrating this into our nightly regression
testing. Like I've said in my <a
href="http://blog.ffwll.ch/2013/02/fosdem-slides-2013.html">fosdem presentation
this year</a> I was shocked how much fallout and regressions even the shittiest
testsuite catches ... 

Since then an awful lot of stuff has happened: 

- We've switching to piglit as our testrunner framework. The big reason for that
  was to automatically enumerate subtests and even more important to be able to
  easily filter them. Often we have tests that check combinations of different
  things. One example is `gem_concurrent_blt` which races gpu access
  against cpu access in different ways, using different cpu access methods and
  different caching policies, leading to a nice combinatorial explosion of
  subtests. And some tests also take an extreme amount of time to run, like the
  coherency checkers, especially the ones that exercise the swap paths ... 

  The reason for using piglit was that no other established testrunner seems to sanely support skipping testcases out of the box, and we have tons of platform specific tests. And obviously tests for new features shouldn't fall over on older kernels. The other reason for picking piglit was that we could cobble together an integration with the existing tests, without really changing them, in a few lines of python. All other testsuites I've looked at were way more invasive to convert over to . 

- We've added tons more testcases. Starting with just twenty-odd test tools
  we've ended up with currently over 300 testcases. And very often when we've
  started to test an existing feature bugs just kept on showing up. So nowadays
  new features don't go in without decent test coverage. 

- We've added a lot of debugfs interfaces to the kernel to allow us to exercise
  tricky corner conditions. One of the first examples is the ring stop interface
  to simulate gpu hangs without actually hanging the gpu - the latter has the
  ugly tendency to take down the entire system, hard. With simulated gpu hangs
  we can exercise the hangcheck code, error capturing and dumping and the hw and
  soft reset logic without too much risk. This way we can exercise gpu hang vs.
  pageflip races and other nasty stuff that up to then has all been nicely
  broken. Other examples are interfaces to provoke gpu completion number
  wrap-arounds, disabling the prefetching to exercise slowpaths or interfaces to
  our shrinker code to more accurately exercise some of the memory trashing
  scenarios without actually thrashing the machine too badly. 

- We've also added tons of infrastructure for writing testcases in
  `lib/drmtest.c`. Detailing a bit what's all in there is the aim of
  this blog post. The most recent improvement here is that tests with subcases
  can now properly enumerate all subtests even when the drm kernel driver isn't
  available. 

## Running i-g-t Kernel Tests 

At the moment there are no provisions for installing the tests as binaries and
shipping them in distros - no one yet demanded this. So first you need to grab
intel-gpu-tools and piglit sources: 

	$ git clone git://anongit.freedesktop.org/piglit
	$ git clone git://anongit.freedesktop.org/xorg/app/intel-gpu-tools

Then build them: 

	$ cd intel-gpu-tools
	$ ./autogen.sh &amp;&amp; make
	$ cd ../piglit
	$ cmake . &amp;&amp; make

We also need to tell piglit where the kernel testsuite is with a symlink: 

	$ cd bin
	$ ln ../../intel-gpu-tools igt -s

Finally we can run testcases. Note that currently we assume that testcases are run as root, and there's no other drm client running. The rte testcase (short for runtime enviroment) will fail if that's not the case. Yeah, one of the projects is to improve the piglit testrunner to notice this earlier. Anyway, best get used to always run piglit for the kernel as root: 

	$ sudo ./piglit-run.py tests/igt result

Just enumerating the testcases doesn't need root though. But it will run the binaries of tests with subtests since igt subtests are enumerated at runtime - that helps greatly with constructing combinatorial testcases. So to list all tests just run 

	$ ./piglit-run.py -d tests/igt result

For further options and tools to display the test results, list the test commands and other useful stuff please consult the piglit documentation. 


## Helper Functions


The biggest part of the helper library is a pile of small wrappers around drm
ioctls. libdrm is often at a way to high abstraction level, especially if you
try to be nasty and provoke the kernel into doing stupid things. There's also
the usual duplication of headers that we replicate (in slightly different
versions) in all our userspace driver repositories ... 

But the far more interesting functionality is in the other code in
`lib/drmtest.c`. One of the most often used helpers is the rude
interruptor, which forks a second process to constantly interrupt the first with
a signal. That's useful for three reasons: X makes heavy use of signals, so we
need to be able to exercise these paths, too. Then the kernel code itself relies
on the ioctl restarting to bail out threads when the gpu is hung - this way we
can avoid ugly issues when threads hog locks waiting for the gpu, while the
reset code needs those logs to complete the requests (or restart them if we ever
implement them) after the hardware resets. And finally it's just a great way to
exercise error paths, since most of the tricky error handling code for e.g. out
of memory conditions also needs to deal with interruptions due to a pending
signal. 

Another big chunk of helper logic is the just recently improved support code for
subtests. The next section will go into greater detail on those. Then there's
also helper code for system suspend/resume, gpu aperture thrashing, prefault
disabling. And finally we have a few helpers for modesetting tests, to draw nice
framebuffers with some gradients and corner markers and to get the VT handling
right - it's rather tricky to stop the fbcon from interferring by blanking the
screen in the middle of a test ... 
 
## Subtest Infrastructure and Magic Codeblocks


As mentioned already igt kernel tests enumerate subtests at runtime. Generally
the subtest infrastructure is pretty simple. At the beginning of the test
`igt_subtest_init(argc, argv)` needs to be called so that the subtest
logic can correctly disable subtest - either all off them when only listing
subtest names with `--list-subtests` or all but one when running a
specific subtest with `--run-subtes &lt;subtest-name&gt;`. There is
also a special variant of this init function called
`igt_subtest_init_parse_opts` useful when the test program wants to
augment its own option parsing with this functionality. 

Then it's just a matter of enclosing the code for each subtest in an
`igt_subtest` code block, e.g.

	igt_subtest("basic")
		run_basic_test();

This works like any C codeblock, i.e. if the block contains more than one
statement it needs to be enclosed in curly braces. The block so marked up is
only run when the named subtest should be run - when only listing subtests the
code itself is skipped. The magic `igt_subtest` block will also print
the result of the subtest to stdout. For combinatorial subtests where the
subtest name is constructed at runtime there's also `igt_subtest_f`
which takes a `printf`-like format string and additional paramters
instead of a fixed string. 

Now this is all nice and dandy but leaves the question how to do common setup
code outside of the actual subtests and make sure it doesn't interfere when just
listing the subtests or when running on a unsupported system where all subtests
should be skipped. And then there's the question what to do when a subtest
should be skipped or failed. 

For the first issue there's another magic codeblock, `igt_fixture`.
Anything within such a codeblock will be skipped when only listing subtests, but
unconditionally run when any subtests should be run. So code to setup the global
test fixture should be enclosed in such blocks. 

For testcases without subtests the skipping and failing is easy to do - failed
tests simply exit with a nonzero exit code, and the special exit code 77 is
reserved to denote a skipped test. But with subtests we want to go through the
entire remaining program so that the results for all subsequent subtests is
still printed. Here comes `igt_skip()` and
`igt_exit(&lt;exitcode&gt;)` to the rescue. With some
`longjmp` magic they will bail out of the enclosing magic code block
(either a subtest or a fixture). If those magic macros bail out of a subtest
only that subtest is skipped/failed and all subsequent code is run - subtest of
course only if they should be run, but fixtures (e.g. used for cleanup) are
unconditionally run. 

But if those magics macros are called within a fixture all subsequent fixture
and subtest blocks are skipped since presumably a required piece of state
couldn't be set up. Also for all subtests that still follow their status will be
reported as either skipped or failed. This way when new features are added to a
new combinatorial testcase which depends upon kernel support (like a new buffer
object coherency mode) this can be added at the end of the test sequence: If the
setup in a fixture block then fails all the subtests which need this new kernel
support will be automagically skipped, but all the previous tests still run
correctly. 

For convenience there's also the `igt_assert` and
`igt_require` macros which work like a normal assert. But instead of
calling `abort()` internally they call `igt_fail()` and
`igt_skip()`, respectively. And of course the print the file and line
where the failure happened and the condition itself for easy identification.
Unfortunately macros get expanded before being passed on, so ioctl numbers are
really hard to read ... 

Finally there's also an `igt_success()`, but most tests don't need to
call this explicitly since the magic subtest code block will call this function
at the very end of the code block, presuming that the subtest succeeded when it
completed. And finally there's `igt_exit()` which will exit the test
program with the correct exitcode for the piglit testrunner to interpret the
result. 

One thing to remember is that both fixtures and subtests internally use long
jumps and that those can wreak havoc with stack variables. But when the
testcases are written properly (specifically subtests must be independent of
each another and only depend upon all the preceeding fixtures to have completed
successfully) then this shouldn't ever cause trouble. It did cause a bit of
trouble while converting testcase on an earlier version of&nbsp; the fixture
block - that used a long jump even when the fixture successfully complete and so
could wreak the just set up stack variables. 

So all in all for a testcase with subtests like `gem_mmap_gtt` this
results in a fairly streamlined main function, despite all the magic:

	int
	main(int argc, char **argv)
	{
		int fd;

		igt_subtest_init(argc, argv);

		igt_fixture
			fd = drm_open_any();

		igt_subtest("access")
			test_access(fd);

		/* More subtests ... */

		igt_fixture
			close(fd);

		igt_exit();
	}

The test functions themselves can also shed a lot of the control flow logic by
directly using the `igt_require/igt_asssert` macros. Compared to the
previous state before the introduction of the magic subtest and fixture code
blocks all this control flow was explicitly done in each testcase and
correspondingly buggy. And dropping all that code also nicely clarified what's
actually being tested. 

**Update:** Fix the setup commands a bit as spotted by Mika Kuoppala. 
