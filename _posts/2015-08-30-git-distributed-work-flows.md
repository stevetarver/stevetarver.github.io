---
layout: post
title:  "Git: Distributed Work Flows"
subtitle: ""
date:   2015-08-30 19:26:41
categories: centralized repo dictator and lieutenants git integration manager
---

Once upon a time, I had to introduce Git to a subversion community. So I read the Pro Git book, mined some cool sites, and took notes.


## Overview

This note describes ways to organize repositories for the three fundamental distributed workflows.

Distributed work flows are primarily concerned with repository organization and permissions.


### Resources

- [Git Pro Book – chapter 5](http://git-scm.com/book/en/v2)


## Centralized Repository

Subversion uses a centralized repository that every developer checks out from, and checks into.

Many enterprise git projects will use this same approach. It is ideal for small cohesive groups that pair program.

### Permissions

All developers have read permission, project developers have read/write access on a single repository.

### Code Review

Continuous code review is the best form of code review.

Groups can use pull/merge requests when merging a feature into a more stable branch (e.g. next release). Merge requests allow line by line discussion of the new feature. A merge request can be opened early in feature development allowing developers to collaborate and discuss feature implementation – all saved in one convenient place.

External contributions are reviewed in response to a pull/merge request.


## Integration Manager

This strategy uses git’s ability to have multiple remote repositories for a single project.  Each developer has a public repository that forks the blessed repository and their work is pulled by an Integration Manager to apply to the blessed repository.

The Integration Manager is a quality and management checkpoint for all code entering the blessed branch.

This strategy is appropriate for disparate developers or groups of developers that need a checkpoint to ensure a base level of quality and to manage feature addition and releases.

Another benefit to this approach is that each merge request can have a line by line discussion of the merge in GitLab which can be saved for posterity. This helps answer questions like “I know there was a reason we did this, but why, oh why?” Note that this can be implemented between branches in the Centralized Repository strategy as well.

### Permissions

Each developer has read/write access to their own repository which is a fork of blessed, and read access to the blessed repository.

The integration manager has read access to each developer’s public repo, and read/write access to the blessed repo.

Individual developers complete work locally, push to their public repos, and send a pull request to the integration manager.

The integration manager evaluates the change, and if acceptable, pushes to the blessed repo.

### Code Review

This strategy adds a code review checkpoint at the Integration Manager level to ensure a common base quality usually among disparate developers or groups.

Individual developers or groups will implement their preferred method of code review.


## Dictator and Lieutenants

This strategy scales the integration manager strategy, providing several first tier  integration managers, called lieutenants, who all have a single integration manager known as the dictator.

This pattern is generally used by huge projects with many collaborators, e.g. the linux kernel.

Each lieutenant is an expert in part of the project. All liertenants have one integration manager; the benevolent dictator. The dictator has final say on what goes into the blessed branch and when.

### Permissions

Each developer has read/write access to their own repository which is a fork of blessed, and read access to the blessed repository.

Each lieutenant has read/write access to their own repository, read access to the blessed repository, and read access to each developer’s public repo.

The dictator has read/write access to their own repository, read/write access to the blessed repository, and read access to each lieutenant’s repository.

When a developer completes work, they send pull requests to their lieutenant. The lietenants send pull requests to the dictator who decides what and when to push to blessed.

### Code Review

This strategy adds domain experts as quality checkpoints, and a top level manager that controls feature addition into the blessed repository.

Individual developers or groups will implement their preferred method of code review.

Lieutenants will enforce project level quality guidelines and domain and code quality.

The dictator performs summary quality checks, trusting, but verifying, lieutenants are applying appropriate quality control. The dictator’s main job is controlling feature addition to the blessed branch.