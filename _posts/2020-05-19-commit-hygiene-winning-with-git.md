---
id: 2190
title: 'Commit Hygiene &#8211; Winning with GIT'
date: 2020-05-19T21:15:12+10:00
author: eddiewould
layout: post
guid: https://eddiewould.com/?p=2190
permalink: /2020/05/19/commit-hygiene-winning-with-git/
ocean_gallery_link_images:
  - 'off'
ocean_sidebar:
  - "0"
ocean_second_sidebar:
  - "0"
ocean_disable_margins:
  - enable
ocean_display_top_bar:
  - default
ocean_display_header:
  - default
ocean_center_header_left_menu:
  - "0"
ocean_custom_header_template:
  - "0"
ocean_header_custom_menu:
  - "0"
ocean_menu_typo_font_family:
  - "0"
ocean_disable_title:
  - default
ocean_disable_heading:
  - default
ocean_disable_breadcrumbs:
  - default
ocean_display_footer_widgets:
  - default
ocean_display_footer_bottom:
  - default
ocean_custom_footer_template:
  - "0"
ocean_link_format_target:
  - self
ocean_quote_format_link:
  - post
categories:
  - Uncategorized
tags:
  - commit
  - git
  - reauthor
  - rebase
---
GIT101: A novel mental-model of git concepts, the pitfalls of traditional "pulling" and "merging" & how to author a good commit (and why it matters).

## A layman's mental model for everyday GIT concepts

The point here is not to accurately describe how GIT works, but rather to provide enough of a basis for the subsequent section "How to GIT better".

### What is a commit?

Those of you who remember highschool physics may remember being lectured on how light is _simultaneously_ a wave _and_ a particle.

<figure class="wp-block-image size-large"><img src="/wp-content/uploads/2020/05/light-particle-wave-1024x768.jpg" alt="" class="wp-image-2192"/></figure>

Along the same vein, you can think of a commit in GIT as both (_simultaneously_)

1. A state of the filesystem (repository)
2. A set of changes to transition from one state (before) to another (after)

You can ask GIT to checkout a given commit and, like magic, the filesystem will be like it was when the commit was created.

#### Commits have an ancestor (parent) 

Each commit has one (or sometimes more than one) parent commit i.e. the commit that came before it. Except for the initial commit that is.

#### Commits include meta-data

As well as (effectively) representing the state of the files in the repository as-at the commit, the commit includes various meta-data including

* Commit message
* Author
* Author date
* Commit date

It's worth distinguishing the _author date_ and the _commit date_. The former is informational (for humans) and many tools will let you override it. The latter is set automatically when the commit is created. It might be _possible_ to override the commit date, but you probably shouldn't be doing that.

#### Commits are immutable

Once you create a commit, you can’t change it, you can only create a new (similar) commit. 

#### Commits are identified by a SHA

When you create a commit, a cryptographic function (SHA) is applied covering

* The state of the files in the repository at the time the commit was made
* The meta-data in the commit (message, commit date, author date etc)

That means that if any file in the repository is changed, or the commit meta-data is changed, you'll get a different SHA. That's part of the reason you can think of commits as immutable: Even if you apply the _exact same changes_ with the _same message_ and set the author-date, you'll _still get a different SHA_ because the _commit date_ is different.

### What is a branch?

You can think of a branch as a _mutable pointer_ to a _specific commit_. The branch has a *name*, and carries some basic meta-data (such as remote-tracking information).

An analogy: In object-oriented programming, a variable might hold a reference to an object. i.e. `Car myCar = someHonda;` You can then change (mutate) that reference i.e. `myCar = new Mercedes();`

Branches in GIT are very light-weight.

#### What is a tag?

If we continue the analogy a little further, if a branch is a mutable pointer then a _tag_ is a constant reference. In other words, a tag is a named _immutable_ pointer to a specific commit

### A note on commit "reachability"

It's useful to think about whether a given commit in GIT is reachable. 

* If a branch points to a given commit, then that commit is reachable
* Likewise, if a tag points to a given commit then that commit is reachable
* Finally, if the commit _is an ancestor_ of another reachable commit, then the ancestor commit becomes reachable (transitive)

Any commits that don't fall into the categories above are _unreachable_. Similar to memory-managed languages, unreachable commits eventually get garbage collected (deleted) by GIT.

### What is a remote?

The first thing to realise is that GIT is a _distributed_ version control system. That means there are _lots of copies of the repository_. Indeed, every time a developer checks out the repository you’ve created _clone_.

As a side note, many organisations will choose to treat the copy of the repository that resides on GitHub/Bitbucket as “authoritative” (source of truth). But it's really no more special than any other copy (clone) of the repository.

A remote in GIT represents _a named copy of the repository that isn't the current working  copy_. It encapsulates the details that GIT needs to communicate with that copy, for example, the URL and credentials. Many users of GIT will only work with a single remote (conventially named 'origin') 

### Relationship between local and remote branches

Let’s assume you’re not doing anything fancy - you’ve created a branch in your working copy and pushed the branch to the remote (in this case, GitHub). The branch in your working copy is set up as a _remote tracking branch_ (tracking the branch with the same name in the remote).

Earlier, I described a GIT branch as a "pointer to a specific commit". In the aforementioned scenario, it's worth realising there are *three* such pointers to think about:

1. The branch in your working copy
2. The branch in the remote
3. The _locally cached copy of the remote branch_. In other words, *what the remote branch was pointing at last time we checked*

#3 is not talked about often, but is cruical for understanding GIT primitive operations:

* When you perform a `git push`, you’re updating the branch on the remote (#2) to point to a different commit.
* When you perform a `git fetch`, you’re updating the locally cached copy of the remote  (#3)<

In both cases above, if the commits that the pointer refers to aren’t present, they are uploaded/downloaded as applicable.

### What is a merge commit?

On the "happy path", commits are linear - each commit has exactly one ancestor (i.e. the commit that came directly before it).

Assume you have two branches that started from the same point (commit). Work is done independently on those branches meaning commits are added. The two branches have _diverged_. 

<figure class="wp-block-image size-large"><img src="/wp-content/uploads/2020/05/git_diverged.png" alt="" class="wp-image-2198"/></figure>

A merge commit is a special commit that splices together two (or more!) sequences of commits *that have diverged*. After the merge commit, the sequence of commits becomes linear again.

Merge commits are created when you ask GIT to merge branch "B" into branch "A", but the commits on branch "B" do not "follow-on" from the last commit on branch "A". If the commits on branch "B" _did_ simply continue on where branch "A" left-off, then GIT would perform a _fast-forward_ merge.

<figure class="wp-block-image size-large"><img src="/wp-content/uploads/2020/05/git_ff_merge.png" alt="" class="wp-image-2199"/></figure>

*Note:* There is an option to ask GIT to create a merge-commit even when a fast-forward merge is possible. 

The diagram above shows a "fast-forward" merge from "origin/master" into "master". Because commit `4b7c` is in the lineage of `a95b`, all GIT has to do is simply advance the pointer of the "master" branch.

### What does a "git pull" actually do?

`git pull` is really a *composite* operation. Roughly speaking, it  breaks down to

1. Perform a `fetch`. That is to say, go to the remote (e.g. GitHub), find the associated _remote-tracking_ branch, and see what commit it is pointing at. If we don't have that commit, download it. Update our locally cached copy of "where the remote branch is at"
2. Perform a `merge` from the remote branch into the local branch. How GIT handles that merge will depend on the situation.

## How to GIT better

### GIT pull considered harmful

Given you now know what `git pull` does under the hood, you might want to consider avoiding it. 

One commonly-cited argument against `git pull` is that it has a habit of creating ugly merge commits.

Assume two developers are working on the same branch and are both up-to-date with respect to the remote:

* Developer A makes a commit and pushes his changes to the remote
* Developer B makes a commit
* Developer B pulls from the remote

When git executes `pull` on behalf of developer B it creates a merge commit. Why does it do this? Quite simply, because it needs to merge the remote branch into the local branch, but the local branch has _diverged_ from the remote branch.

_This_ merge commit doesn't really contain any useful information. All it tells you is that two developers were working on the same branch at the same time & that one made a commit before incorporating the commit the other had made.

Granted, these merge commits do seem pointless and are indeed ugly. However there is a *more important* reason to avoid `git pull` and that is i*t encourages ignorance/complacency when incorporating upstream changes*.

Pull-requests are a great tool for ensuring code quality. That said, on all but the smallest teams, for any given pull request *only a handful of developers will examine the PR before it is merged*. Also note that the PR being merged is asynchronous with respect to other developers incorporating the changes - it could be days before a developer is ready to pause what they're doing and incorporate changes from `master`.

The upshot of that is: When the time comes for you to incorporate upstream changes, *it should be conscious activity*. It’s important to understand *all* of the changes that have been made on the trunk since your branch diverged from it. Even if your changes still compile, *they may no longer be correct* with respect to how the codebase has moved on.

<figure class="wp-block-image size-large"><img src="/wp-content/uploads/2020/05/git-pull-meme.png" alt="" class="wp-image-2202"/></figure>

### How to avoid GIT pull

As an alternative to using `git pull`, you could apply the following workflow:

1. Start by performing a `git fetch` to work with the latest from the remote
2. Then, inspect where your (local) branch is at c.f. the remote branch.

* Your local branch is behind the remote? 👉 Perform a fast-forward merge
* Your local branch is ahead of the remote? 👉 No action needed 🎉
* Your local branch has _diverged_ from the remote? 👉 Perform an _interactive_ rebase

_Yes_*,* there is an option to tell GIT to perform a rebase when pulling. I suggest that you *do not use it*. 

### What is an interactive rebase?

Recall from the previous section that commits

* Represent a set of changes to apply (like a patch)
* Are immutable

_Conceptually_, a rebase takes a sequence of commits and “lifts and shifts” them so that they *start from a different point* (commit). In reality, git _creates new commits_ that apply the same patch as the old commits did.

By default, rebases are unattended - git goes and creates the new commits without any intervention with you. When you perform an _interactive_ rebase, git shows you each of your new commits as they being made and gives you an opportunity to _edit_ each commit.

Actually, interactive rebases can do more than "lift-and-shift" commits - when you perform an interactive rebase, you’re given the opportunity to

* Edit each commit (perhaps you missed making the change in one spot)
* Skip (omit) the commit (perhaps the commit contained temporary-only changes that you don't want to keep)
* Reorder the commits (perhaps you realised that the commit adding the new database table should come before the commit containing code that _uses_ the new table)
* Combine the commits (perhaps it took you a few goes to get something working)
* Splice in a new commit

Whilst lifting-and-shifting is the _common_ use-case, you can actually rebase using the same commit as the starting point. This is extremely powerful - it allows you to “reauthor” a sequence of commits taking advantage of the powerful operations listed above.

_As an aside_, the interactive rebase in <a href="https://tortoisegit.org/">TortoiseGit</a> is particularly easy to use.

### Incorporating changes from the trunk (master)

If it becomes necessary to incorporate changes from the trunk (`master`) into your feature branch, the classic approach is to ask git to merge the trunk branch into the feature branch.

Instead of doing that, I advocate _rebasing the feature branch onto the trunk. _That is to say, "lifting and shifting" the feature branch commits so that they start from where the trunk "left off".

#### Conflicts

Regardless of whether you're merging or rebasing, conflicts are a fact of life. The difference is in how they manifest:

* With a merge, you’ll see all the conflicts at once with no context. This "sea of red" can be intimidating to deal with. "What change was I making to this file?" "What change was _he_ making to this file?" "How should I resolve it?" Dealing with all of the conflicts at once is stressful. Stress leads to mistakes, bugs being (re)introduced etc.
* Because an interactive reapplies your commits _one at a time_, you only see the conflicts associated with a _single given commit_ of yours. That's often *a much smaller set of conflicts*, and you also have the *context* (commit message) from your commit to help resolve the conflicts

Even with the reduced set of conflicts that an interactive rebase offers, sometimes the set of conflicts is *still* too intimidating. A workable strategy that can be employed in this situation is to "accept defeat". That is, to use the upstream version of the conflicted files and revert all other your other changes, leaving you with a clean slate / empty commit. Then _while still editing the commit, _recreate it. Sometimes the commit message is enough, but otherwise look at the diffs from the _original_ commit (i.e. before rebasing) and piece it together. It sounds complicated / like a lot of work, but in some situations is subsantially easier than resolving the conflicts.

Once you've fast-fowarded a feature branch onto the trunk, the subsequent merge becomes trivial (i.e. a fast-forward/simple advancing of the pointer). As a result, you'll end up with a nice linear history that is easy to follow.

As with any form or merging, I highly recommend *reviewing the changes upstream before starting the rebase*. You want to know what has changed in the trunk since your branch diverged from it.

When you go through your commits one by one, keep in mind how the trunk has evolved since you made those commits. The bare minimum is to check that the code still compiles & tests still pass at each commit. However you should also be considering

* Is the commit still _needed_? Perhaps changes on the trunk mean this commit is redundant
* Is this commit still _complete_? Does it still cover everything "it says on the tin" or are more changes needed somewhere else now?
* Is there a _better way_ of making the changes now (e.g. a new interface was introduced that you can take advantage of)

Consciously revisiting your commits one-by-one after reviewing upstream changes *is very valuable*. In my experience, this does not happen with the traditional merge approach.

#### With great power...

...comes great responsibility. Here are some rules I suggest you abide to when rebasing:

* Don’t mix merging and rebasing on your feature branch. *Stick to one or the other*
* Avoid automatic rebases, only use the interative variety.
* Don’t rebase branches other developers might have based their work on. In particular, *don’t rebase the trunk* (i.e. `master`)
* If you’re pushing your branch to the remote after rebasing it, you’ll need to force push. Make sure you use the safe variant i.e. `--force-with-lease`. You’re still overwriting the value of the branch, but you’re telling git “only overwrite it if it’s still what I think it is”.

## How to author a great commit

### Changes

A good commit should be

* Atomic
  * That is to say, it makes one *well-defined* improvement to the system. That improvement needn’t be a user-visible change (for example, refactoring), but you *should be able to describe it in one or at most two sentences*.
  * Code must be deployable as-at that commit. That means it should obviously compile but should also not cause regressions in existing features.
  * Tests should pass. Ideally _all_ tests, but it's acceptable to limit the scope to *all previously existing tests*. That is to say, it's _acceptable_ for a commit to deliberately introduce new failing tests.
* Minimal
  * The commit includes only what is *absolutely necessary* _to effect the specified change_. If you feel like it would be a good idea to apply wholesale reformatting "while you're in there", *do that in a separate commit*.

### Message

The subject line should start with the the ticket number (if applicable) as the first thing in square brackets e.g. `[JIRA-9999]`

When writing commit messages, most developers tend to focus solely on the “what”. In some sense, the “what” is redundant - you can tell the “what” by looking at the diffs. However, the "what" should still form the basis for your commit message (subject line) to save people having to look at the diffs. That said, when writing the "what", try to focus on the big picture or _intent_ - i.e. *what you’d need to know to recreate the commit from scratch if you couldn't see the diffs*.

To write a *really* valuable commit message, put more effort into the _why_ - the _intention_ behind the commit. This will implicitly tell you things like


* What will break if I revert this commit
* What motivated this change

You might also consider (briefly) discussing alternatives considered, *and why they were rejected*.

Finally, sometimes it's useful to include information as to how the changes were generated, e.g.

> These files were created by executing `SwaggerGen --SomeMysteriousParameter`

## Why bother authoring great commits?

* It's a courtesy to your reviewer.
  * If you make the review easier, you'll get better feedback and therefore reduce the likelyhood of bugs creeping in
* Easier to merge & release progressively
  * Because your commits are atomic and in a sensible order, you can make small, incremental deployments to production
* Assist yourself (or team mates) with creating "similar" changes in the future
  * If you're really disciplined, you can go back and look at the last commit where you made a similar change (e.g. adding a new "command"). By looking at that commit, you'll know the changes you need to make - add these three files, register the handler here, add the database migration etc - *this can save valuable time*.
* Enterprise software *tends to stick around for a long time*. Commit messages (if authored well) can be a gold-mine from a software archaeology perspective

Hope this was of value to you. Happy GITing everyone!