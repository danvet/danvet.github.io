---
layout: post
title: git for Bug Reporters
date: '2012-05-19T09:51:00.000-07:00'
author: danvet
tags:
- Maintainer-Stuff
modified_time: '2013-07-21T06:56:09.025-07:00'
blogger_id: tag:blogger.com,1999:blog-8047628228132312466.post-3581280925304476115
blogger_orig_url: http://blog.ffwll.ch/2012/05/git-for-bug-reporters.html
---

Pretty often I point bug reporters at random git branches and sometimes they'll happily compile kernels from sources, but have no idea about git. Unfortunately there doesn't seem to be a quick how-to tailored to the needs of bug reporters, so let's fix that.



$ git clone --depth 1 --no-checkout --no-single-branch &lt;git-repo-path&gt;



This clones the git repository, but avoids downloading the entire history. It also avoids to check out the default branch because usually you need a specific branch for testing.



All the following commands only work if the working directory is within the newly cloned git repository.



$ git checkout origin/&lt;branch-name&gt;



will check out the git branch &lt;branch-name&gt; (from the remote git repository) into the local git repository so that you can use it. git will complain about 'detached HEAD' but that's only really important when committing new changes, so just ignore that. If you already have an older clone of the git same remote repository, you first need to update the local git database (in the .git directory in your local repository) with



$ git fetch origin



before checking out the branch again. Even if you need to test the same, but updated branch, you need to do <i>both</i> steps - git doesn't update anything automatically by default.



If you have patches or other local changes applied, git might complain that there's a conflict. You can simply remove all local changes with



$ git reset --hard



and then re-do the checkout command. Much more rarely some build artifacts could get in the way (or you just need a quick way to start a clean build),



$ git clean -dfx



will remove any files not in the currently checked-out git commit.