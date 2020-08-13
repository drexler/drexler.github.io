---
layout: post
title: Git reflog to the rescue!
date: '2020-04-17T09:08:00.000-07:00'
categories: git
author: dReXler
tags:
- general 
---

The standard development workflow utilized by my team requires rebasing of feature branches on the master branch prior to issuing a pull request and the subsequent merge. This allows us to ensure that commits are always *in order* after a merge rather than how they are arranged when a merge strategy is employed. There are pros and cons to each approach. However, having been a heavy Git user for years I tend to favor the rebase approach. So one can imagine when for a hairy 10 minutes i lost my work entirely on a new feature branch. Pull from remote? That was useless since I had done a rebase previously and force-pushed the local branch up to the remote branch thus wiping everything there. Here's how I got into such a state. 

Here's how I got into such a state. The repository was new and had originally been created without any file. So on pushing up my feature branch for the initial review, it became the default branch. To correct this, the idea then was to create a *master* branch from the feature branch and delete all the commits with the exception of a README file it contained. Thereafter, the feature branch will then be rebased on master as always and the PR issued against it.  What happened on the rebase was that Git seeing that all the commits were deleted essentially resolved the rebase automatically by deleting the commits I had in the feature branch. Rebase re-writes history! Being the same commit hash ids, I was left with nothing. Immediately pushing to remote after the rebase compounded the issue.

With `git log -2 --oneline` returning nothing how was I to recover days of work. Git Reflog! I knew about it but had never had the occasion to use it. For anyone reading this, invest your time in and master advanced Git commands and especially that. It's a life(hair?) saver!  Below were my commands: 

```
$ git reflog 
$ git reset HEAD@{number}   //commit before rebase
$ git status
$ git reset --hard 
```

To summarize the above, I needed to get my local branch back to the state it was in before I rebased it. So the first two commands did that. 
Next, to be sure i had the correct files staged - `git status`. Seeing all the deletions there, the obvious thing was to discard those changes thus the `git reset --hard`.  

With work now restored, it was easy to push it back to remote and end on good note. 