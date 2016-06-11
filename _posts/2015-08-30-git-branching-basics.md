---
layout: post
title:  "Git: Branching Basics"
subtitle: ""
date:   2015-08-30 16:56:11
tags: git branch command-line fetch merge pull push rebase
---

Once upon a time, I had to introduce Git to a subversion community. So I read the Pro Git book, mined some cool sites, and took notes.

## Overview

This note describes basic branching concepts and operations from the command line. IDEs with git implementations should simplify these operations, but having an overview allows you to map the concepts onto the IDE.

Branching and merging in git is a daily task. Branch and merge has improved in all VCSs in recent years but it remains an underused feature. Git's implementation is so fast and simple that it is easily incorporated into daily work and thought about code.

## Resources

[Pro Git book – Chapter 3](http://git-scm.com/book/en/v2)

The Pro Git branches chapter is quite good, but if that is too much to read, here are some notes.

## Git Storage

Most VCS systems organize their data as a series of changesets or differences for single files.

Git stores its data as a collection of project file system snapshots. A commit object contains:

- metadata like author, date, and message
- a pointer to the snapshot
- pointers to ancestor (parent) commits.

The snapshot contains

a blob for the contents of each file
a tree defining project directory contents and mapping file names to blobs

A branch is a lightweight pointer to a place in the commit tree; a file with the 40 char SHA-1 checksum of the commit it points to.

## Branches

The default main branch name is master. It is so widely used because “git init” uses that name and no one bothers to change it.

As you start making commits to master, the master branch pointer moves forward along the commit tree.

HEAD is a pointer to the tip of whatever branch you are currently working on. As you create new branches, and start working in them, the HEAD pointer will move to point to your current branch.

Create a new branch

```git branch```

Switch to the new branch

```git checkout```

Switching branches changes files in your working directory; it resets them to match the state of the new branch.

You can switch between branches and commit work with them easily with branch, checkout, and commit.

To create a new feature branch for, say Jira story SS-903, and check it out to the current working directory to start work on the story.

```git checkout –b SS-903```

If you need to switch contexts to work on say, a production issue, commit your changes on the current branch, switch back to the production branch, and create a new branch for the production issue.

{% highlight bash %}
git commit –a –m “incremental progress”
git checkout master
git checkout –b hotfix-953
{% endhighlight %}

Once the work on the hotfix is done, merge it back into the master with

{% highlight bash %}
git checkout master
get merge hotfix
{% endhighlight %}

Now that the fix is merged into master, you can delete the branch

```git branch –d hotfix```

NOTE: If you want a cleaner commit tree; where commits indicate meaningful work, you can “stash” your current branch work, switch to other branches to work, and then “unstash” you changes on the original branch.

Most branches are not long lived in git; as soon as you have merged the work in a feature or story branch, you delete it.

In git, standard practice is to create a branch for all new work. This allows you to trivially shift context from the new work to, say, fix a production issue with a stash or commit and then a checkout.

In contrast, in svn, you would have to create a new branch from the production release tag, which can be a bit tedious because it is no longer the tip of the trunk, checkout that branch to another working directory, create a new Jenkins build job for the production release branch, etc. Git trivializes this workflow so it can be a part of your daily workflow.


## Fetches

Fetch is a “safe” way to update your local repo from a remote: git will not try to merge any files. After fetch, you can see new branches, how far or behind the remote you are, and merge things you think are important.

Use fetch to bring down new project branches, files, without modifying your working directory.

```git fetch origin```

All new branches are shown, but this is just a pointer to the remote branch. To create a local copy of that branch that you can work in:

{% highlight bash %}
git checkout –b origin/
git checkout –track origin/
{% endhighlight %}


## Merges

Merge conflicts occur when two branches have modified the same parts of the same files. Merging is best done with a good three way merge tool like that in IntelliJ.

You can merge on the command line using git status to identify the merge conflicts. and editing them with something like vim. You could also launch a graphical three way merge tool with “git mergetool” although this might need some configuration to launch the tool you want (e.g kdiff3).

You can use long running branches to  represent different work streams. For example, you could use master to represent production ready code. You could have one long running branch to represent the next major release and separate branches that represent each feature in the next release. When a feature branch reaches suitable quality, it would be merged into the release branch. When the release branch contains all features that are ready to deploy, the release branch would be merged into the master and go through the normal CI process.


## Pulls

The git pull command combines a fetch and merge for your current branch. Generally, it is less confusing to fetch and merge to control what is merged when, instead of a bulk merge attempt with pull.


## Pushes

A git push moves code from your local branch to the remote; when you need to share it to collaborate.

The basic push command is “git push ”. If you are working on issue SS-903, contained in a similarly named branch, and want to share that branch with others collaborating on the work.

```git push origin SS-903```


## Rebase

Both merge and rebase integrate changes from one branch into another.

Rebasing replays changes from one line of work onto another in the order they were introduced, whereas merging takes the endpoints and merges them together.

The final commit from a merge or a rebase will point to the same snapshot, only the history will be different.

Why?

- It provides a much cleaner history; eliminating the branch and merge entries and providing a more linear history.
- Ensures your commits apply cleanly to the remote branch in a fast forward or clean apply.

This approach is appropriate when you don’t need to retain all of your local branch comments for posterity; just the final entry.

One Rule: Can’t rebase commits that exist outside of your local repository.

Violating this rule will make everyone else using the rebased branch have to merge in your changes. You can also create a history with duplicate commits – same author, date, and message.

Treat rebasing as a way to clean up and work with commits before you push them, and if you only rebase commits that have never been available publicly, then you’ll be fine.

Excerpt From: Scott Chacon, Ben Straub. “Pro Git.” iBooks.


## Common commands

- **git branch –v** : show all branches and the commit message for the last commit on each.
- **git branch –merged** : list all branches that have been merged into your current branch – the one you have checked out.
- **git branch –no-merged** : show all branches that have work you have not merged into the currently checked out branch.
- **git branch –vv** : Show all branches, including the remote branches they are tracking.\
- **git push origin –delete** : delete a branch from the remote. This can be used on short lived branches, but it is probably less risky to do from GitHub/GitLab. This only deletes the branch pointer; you can recover the branch until git garbage collection runs.
