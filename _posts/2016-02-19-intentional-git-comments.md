---
layout: post
title:  "Intentional Git Commit Messages"
date:   2016-02-19 11:02:35
tags: git scm
---

<img style="float: left; width:180px;" src="/images/github-logo.jpg">

In git, commit messages divide people into two camps - Commit history:

1. is a record of what actually happened
1. is the story of how your project evolved

I live exclusively in camp 2 and I want to tell you why. Three great enablers of business agility are: 

- clarity of intent
- clarity of thought
- ability to reason about your code

Minimal, well-written commit messages can improve all three. Recording every commit for posterity, especially when lines of thought overlap, simply clouds intent, linear thought, and the ability to understand what was going on with your code at any point in time.

Git allows you to trivially spin up branches for experiments and features. This simplifies separating lines of thought. Git also allows you to rewrite commit history much as an editor might rewrite initial drafts of a book to produce a simple, straight-forward biography. 


## References

- [Fundamental guidance by Tim Pope](http://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)
- [Who-T advice](http://who-t.blogspot.com/2009/12/on-commit-messages.html)
- [Git SCM book on commit messages](https://git-scm.com/book/ch5-2.html)
- [Chris Beams on Git commit messages](http://chris.beams.io/posts/git-commit/)
- [Pan Thomakos on commit messages](http://ablogaboutcode.com/2011/03/23/proper-git-commit-messages-and-an-elegant-git-history/)
- [GitUp](http://gitup.co/)


## The problem?

Most commit logs are full of overlapped, stream-of-consciousness commits. There are many paths to how we got to this state, but the real question is: does it server my project and my team today?

I had a production issue in a collection of about 10 artifacts - separate 'repos' to wade through. Features had accumulated over our monthly deployment cycle in a kind of ad hoc way. I could trace the problem to a single feature in the logs and had to figure out what code had changed and where the problem had been introduced. As I read through an accurate record of what commits happened at what time, I found things like 5 commit messages in a single day that changed statements back and forth as the developer would try something, push it to dev, figure out it was wrong and then try something new. Lots of contributing challenges there, but with managers standing over my shoulder, asking "do you have a fix yet", I came to truly appreciate single feature commits.

Does your code base show clarity of thought? Enhance the ability to reason about your code?

When your code has a single commit message from a feature branch that tells what the feature was, how you implemented it, and why you made the choices you made, you can browse git history and easily understand what was going on at any point in time.

Commit messages should be brief, should be summaries of intention and reasoning. GitHub pull requests can link to the code and capture all the thought behind implementation decisions.

Why are well-formed git commit messages and capturing thoughts behind implementation decisions important?

**Bugs are hard to kill** You push to PD and find another nuance of your bug. You need to revisit all the thought that went into the original fix so you don't repeat your previous efforts.

**Debugging** Wading through a month of commit comments to find a bug is tedious, time consuming and robs you of time to apply to new features. You can reduce this problem with frequent CD, but the base problem remains.

**Be kind to those that follow** How do new team members learn about your code? Overlapped, stream-of-consciousness commits are impossible to decipher. Single feature merge commits are very clear.

## What do the pros do?

If you want to up your game, you do what the pros do. Canonical examples are Linus' project [commit logs](https://github.com/torvalds/linux/commits/master), and [Git itself](https://github.com/git/git/commits).

What do you see there? If not a merge of branches or tags, you will see clear feature additions or bug fixes.

Using `git log --pretty=format:"%h %s"`, and skipping branch merges, the linux kernel log looks like

```
a672095 unpack-trees: fix accidentally quadratic behavior
a97262c diff: make -O and --output work in subdirectory
e5f7a5d diff-no-index: do not take a redundant prefix argument
bd02e97 git-add doc: do not say working directory when you mean working tree
e6414b4 completion: complete "diff --word-diff-regex="
```

Compare that to 

```
214b95b Merge remote-tracking branch 'origin/develop' into develop
845e5b1 Merging changes from master
6f3573f Updating tests for billing
b91d5fd Updating billing logic
a226a3e Fixed typo issue and made change. Updated issue with field being resolved by appfog manifest.
58b9345 Merge remote-tracking branch 'origin/develop' into develop
c25d1b8 Updated POM versions.
58170a9 added sleep option to the ansible service command. This will cause a 5 second pause between stop and start of the salt-minion service
931971d always restart since resize is performed before running playbook
95def90 Returning a 204 on AF delete
```

Which of the above would you like to read if you had to maintain a code base?

Ideally, you want to be able to read commit messages, understand how the code base is evolving, without having to read the code. Do you commit messages allow that?


## What the experts say

From [Pan Thomakos](http://ablogaboutcode.com/2011/03/23/proper-git-commit-messages-and-an-elegant-git-history/)

> Joel Spolsky put it best when he said, "The difference between a tolerable programmer and a great programmer is not how many programming languages they know, and it’s not whether they prefer Python or Java. It’s whether they can communicate their ideas... By writing clear comments and technical specs, they let other programmers understand their code, which means other programmers can use and work with their code instead of rewriting it. Absent this, their code is worthless. By writing clear technical documentation for end users, they allow people to figure out what their code is supposed to do, which is the only way those users can see the value in their code."
>
>This same concept applies to git commit messages and git history.

> One of the best skills you can learn as a programmer is to write about your code and to explain what you are doing. Git commit messages and code comments are the best place to start.

> The difference between merging branches and rebasing branches really comes down to how you want to format your history. In my opinion when you have a small atomic fix that is a single commit it's always best to rebase. When you have a large feature history in a separate branch that you've been working on for a while it's always best to merge. Anything between these two is really up for grabs, but think about what you want to convey when you merge your code into another branch. Is the merge commit really as significant as your original commit?


The most fundamental guidance comes from the [Git SCM Book](https://git-scm.com/book/ch5-2.html): Basic guidance: paraphrasing

- try to make each commit a logically separate changeset
- don't combine a week's work into one massive commit, split your work into at least one commit per issue
- be kind to your peer developers, big commits are difficult to review
- smaller change sets are easier to revert if needed
- good commit messages add a lot of clarity to what was intended and simplify reviewing

What should a commit message look like?

```
Short (50 chars or less) summary of changes

More detailed explanatory text, if necessary.  Wrap it to
about 72 characters or so.  In some contexts, the first
line is treated as the subject of an email and the rest of
the text as the body.  The blank line separating the
summary from the body is critical (unless you omit the body
entirely); tools like rebase can get confused if you run
the two together.

Further paragraphs come after blank lines.

  - Bullet points are okay, too

  - Typically a hyphen or asterisk is used for the bullet,
    preceded by a single space, with blank lines in
    between, but conventions vary here
```

What should you include in your commit message? Think about these questions when creating you commit message:

- Why is the change necessary
- How does the change address the issue
- What side effects does the change have
- What will I want to read about this change in six months
- Include a link to the trello card, or zendesk issue that motivated the change

When working in a feature branch, commit often. This allows both rolling back to a check point and collapsing commits that do not need to be kept for posterity. 

Consider creating a best practices page for your project like [Karma Runner](http://karma-runner.github.io/0.13/dev/git-commit-msg.html)


## How to reduce commit noise

Reference: [git-scm book](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)

Merge allows you to bring contents from one branch into another - it does a three way merge between two branches and their most recent common ancestor. It creates a new snapshot and a merge commit.

That merge commit is almost always noise.

Git provides a `rebase` command to update one branch from another and avoid the merge commit. From the git-scm book

> Rebasing replays changes from one line of work onto another in the order they were introduced, whereas merging takes the endpoints and merges them together.

Rebasing also helps ensure that commits apply cleanly on a remote branch.

### When to rebase:

- Rebase local branches to clean up history prior to pushing to remote
- Pull in work from another branch and avoid a merge commit
- Ensure commits apply cleanly to a remote branch

### When not to rebase:

Don't rebase when others have your commit history. If you have a branch that is tracked remotely, others have pulled that branch and based work on it, and you rebase, you will create a mess. A merge and a rebase produce exactly the same end result, but a rebase rewrites your commit history. Everyone will have to reconcile their notion of your commit history and your new notion of your commit history.

- Don't rebase anything you have pushed somewhere

## Examples

### Rebase feature branches onto master to avoid clutter

If you are working on a `feature-1` branch and there is a lot of activity in `master`, you can rebase `feature-1` onto `master` effectively moving the starting point of `feature-1` to the tip of `master`.

```
git checkout feature
git rebase master
```

We have already decided that we value a well told story (git history) over capturing pedantic detail. IOW: we want to clearly tell the story of creating our `feature-1` rather than capture the detail of how `feature-1` commits interleaved with `master` commits.

### Squash noisey commits on a feature branch

If you did a lot of checkpoint commits on your feature branch, you don't need to capture that for posterity. Frequently a single well written commit is all history will require.

Let's say you have made 5 commits on your feature branch, and you would like to sqaush all of them into a single final commit, use `rebase --interactive`. 

`git rebase -i HEAD~5` puts you in edit mode on this temp file:

```
  1 pick 1faaba2 Define feature in README
  2 pick 618bdab implement search
  3 pick e2af136 implement pagination
  4 pick ac3c48d fix bug in pagination offset
  5 pick 1ef774a add release notes
  6 
  7 # Rebase e807c29..1ef774a onto e807c29 (5 command(s))
  8 #
  9 # Commands:
 10 # p, pick = use commit
 11 # r, reword = use commit, but edit the commit message
 12 # e, edit = use commit, but stop for amending
 13 # s, squash = use commit, but meld into previous commit
 14 # f, fixup = like "squash", but discard this commit's log message
 15 # x, exec = run command (the rest of the line) using shell
 16 #
 17 # These lines can be re-ordered; they are executed from top to bottom.
 18 #
 19 # If you remove a line here THAT COMMIT WILL BE LOST.
 20 #
 21 # However, if you remove everything, the rebase will be aborted.
 22 #
 23 # Note that empty commits are commented out
```

Change pick to squash for lines 2-5.

```
  1 pick 1faaba2 Define feature in README
  2 s 618bdab implement search
  3 s e2af136 implement pagination
  4 s ac3c48d fix bug in pagination offset
  5 s 1ef774a add release notes
```

Save the file and you will be taken to the commit message editor holding all of the commit messages. Insert a well written commit message for the entire feature, save the file, and rebase does its magic. When complete, your feature branch will have one commit message that you can merge onto master.

All this work is very hammer and chisel. Want the big red easy button? Try [GitUp](http://gitup.co/) - it's free.

Given the same scenario, in GitUp:

1. Open your repo in GitUp.
1. Right click on your branch head and select `Squash with Parent...`
1. Edit your commit message - provide the final, well written version.
1. Click the squash button.
1. Repeat until you have squashed all the noise.
1. Right click on your branch HEAD
1. Select Edit Local Branch "contact-search" -> merge into... -> master
1. Click on the up arrow in the lower right corner to push to your remote

GitUp also provides a trivial undo to rebound from mistakes.

 
