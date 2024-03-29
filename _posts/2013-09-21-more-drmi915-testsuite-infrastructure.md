---
layout: post
title: More drm/i915 Testsuite Infrastructure
date: '2013-09-21T09:27:00.002-07:00'
author: sima
tags:
- In-Depth Tech
modified_time: '2013-09-25T03:16:53.544-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-6942625817573587722
blogger_orig_url: http://blog.ffwll.ch/2013/09/more-drmi915-testsuite-infrastructure.html
---


After the recent [overview over our kernel test
infrastructure](/2013/08/recent-drmi915-testsuite-improvements.html)
I've had to write a bunch multithreaded testcases. And since I'm rather lazily
I've opted to create a few more helpers that hide all the little details when
forking and joining processes. While at I think it's also a good time to explain
a bit the infrastructure we have to help running the kernel testsuite on
simulated hardware and a few other generally useful things in our testsuite
helper library. 
<!--more-->

## Multithreaded Tests 


One important thing when specifically testing the kernel is to exercise race conditions and truely fancy corner-cases. After all the normal cases of submitting a bit of work for the gpu and waiting for the results is really well-tested through testsuites for the userspace driver parts like piglit. Which means that the kernel tests are brimful of multithreaded testcases. 

Now writing parallel testcases with <code>fork</code> and friends isn't really hard. But correctly propagating exit codes from all children and cleaning up things if a test goes south is annoying boilerplate. Especially when taking into account interactions with other testsuite helper infrastructure. Hence we now have another magic code-block macro called <code>igt_fork(i, num_children)</code>. Conceptually it works like a parallel for loop from frameworks like OpenMP: <code>parallel_for (i = 0; i < num_children; i++)</code>. The difference is that control flow in the main thread will continue, and joining together all the children happens by calling <code>igt_waitchildren()</code>. This will block until all children have exited, and it will also forward the exit code if any of them have failed the testcase by calling <code>igt_exit()</code>. 

Two special considerations apply: Skipping a testcase from within a child process doesn't make much sense - such checks should happen in the main test process before forking parallel threads. Hence the testframework will catch such abuse and abort the test if a child process ends up calling <code>igt_skip()</code>. Secondly childs are spawned using <code>fork()</code> and not as pthreads. In my experience of writing testases thus far this is the simpler and more intuitive programming model. The only tricky part is that children need to reinitialize buffer objects (and associated tracking) if they don't want them to be shared - gem buffer objects do not have copy-on-write semantics after <code>fork()</code>. The pitfal here are the reused buffer references stored in the libdrm buffer manager. If children don't reinitialize those different processes could end up reusing the same objects (inherited from their parent) for batches, resulting in some decent hilarity. 


## Cleanup Handling


Another piece of annoying boilerplate code is handling cleanup tasks after the test is done. We have a lot of little things that needs to be reset again, like re-enabling prefaulting when a testcase disabled it or re-enabling the kernel console after a test put it into graphics mode for a modesettting test. libc has <code>atexit()</code> handlers, but that's not good enough since our testcases tend to die a lot through signals. To fix this <code>igt_install_exit_handler</code> will install special signaler handlers (which re-raises the signal after processing all the exit handlers) besides a normal <code>atexit()</code> handler. 

The nice thing is that this plays well together with all the other testsuite helpers, especially for multi-threaded tests. For children the exit handlers get automatically reset, but it's still possible that they have their own exit handling code, e.g. if each child process has it's own helper process to clean up. One strange thing encountered though is that if a child process receives a signal right after forking it seemingly can get lost when our special re-raising handler is registered. We duct-tape over that by disabling the exit handling temporarily across the fork. See the [relevant patch](http://cgit.freedesktop.org/xorg/app/intel-gpu-tools/commit/?id=a031a1bf93b828585e7147f06145fc5030814547), explanations what exactly might be going on highly welcome. 


## Helper Processes for Auxiliary Tasks


Often a testcase needs a parallel thread which isn't part of the actual test, but concurrently does something nasty to it. For example constantly interrupting the parent with a signal to exercise our ioctl restarting code. Or using a special debugfs interface to drop gem object caches and mappings - this is especially in conjunction with disabling prefaulting and allocating userspace pointers as gtt mmmapped objects to exercise the lock-dropping <code>copy_*_user</code>slowpaths.  

Since these process are expected to run forever and until the parent explicitly kills them (once the testcase has completed) there's a second set of fork helpers. The <code>igt_fork_helper</code> magic codeblock forks such a process, and <code>igt_stop_helper</code> cleans it up. 

Both the helper process code and the test children support code use <code>waitpid()</code> to reap dead child processes. This way exit codes won't get stolen and test children and helper process can be forked and reaped in any order. Also both types of process helpers register exit handlers to kill and clean all child processes if the test process dies prematurely. Otherwise, especially for long-running test children and obviously for helper processes, leftover process will keep on running and distort subsequent testcases. 


## Running in Simulation Enviroments


Another result of the kernel tests concentrating on exercising race conditions and similar corner cases is that the testcases tend to take a fair amount of time to run. After all running a stresstest for longer should increase the chances that it will actually hit the race condition. Developers can manage the overtly-long test feedback cycles by restricting the testcase list sufficiently while developing and then running the full set on our QA's patch testing infrastructure. And occasionally we need to re-tune a testcase when it simply takes way too long to complete. 

But we also run the kernel testsuite on simulated hardware, and there all these long-running tests really take forever. Furthermore they often exercise purely software issues with no interactions down to the actual hardware platforms. 

To still keep a reasonable cost-benefit ratio of running the kernel tests in simulated enviroments we skip a lot of the long running testcases. <code>igt_skip_on_simulation()</code> is used for that, which internally calls <code>igt_skip()</code> if the enviroment-variable <code>INTEL_SIMULATION</code> is set. This can be explicitly tested with <code>igt_run_in_simulation()</code>. But there's not much need for that since for race-condition testcase that we want to run but just reduce loop counts a bit we have the <code>SLOW_QUICK</code> macro. This selects one of the supplied values depending upon whether <code>INTEL_SIMULATION</code> is set or not. 

Currently we skip a lot of the kernel tests overall that way, but then the big value of pre-silicon enviroments is to adapt and test the userspace drivers and to be able to do basic integration testing of the entire graphics stack. Hunting for corner-cases and races in the kernel driver is still best done on real hardware, and since we have a unified driver across all platforms most of the code can be tested on older platforms easily. 


## Format String Support


Finally one short paragraph on a very recent addition: All the special test assertion macros like <code>igt_assert</code> or <code>igt_require</code> have grown cousins that accept format strings. Those have a <code>_f</code> suffix. With that we can replace almost all test checks that still use explicit conditions and their own output formatting, which overall tides up the code rather neatly. I've also added <code>igt_skip_on[_f]</code> macros which is just an <code>igt_require[_f]</code> with inverted condition. 

Overall I think that our kernel testsuite is now in fairly good shape and writing testcase is much less a chore than it was just a few months ago. So after the flurry of patches and improvements in the past few weeks I expect things to calm down a lot again and go back to the usual trot of adding testcases here and there. 
   
