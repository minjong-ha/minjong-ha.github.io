---
layout: posts
title:  "Git Guide - Useful and practical commands for teamwork: Basic"
author: Minjong Ha
published: true
date:   2022-12-01 17:29:02 +0900
---

Git is an essential teamwork tool.
When I managed git repository alone (graduate student), what I had to know is a few commands.
However, working with team requires more systemical managing.

Followings are useful and essential git commands what I use frequently.

## Git commands for individual

```bash
git add 
git commit 
git push
git pull
```

Above commands are basic commands for git.
It is useful even if there is a single developer.

```bash
git branch
git checkout
```

Above commands are required from a small team project.
Each members in team can work in independent branches, and requests merge.

## Git commands for teamwork

```bash
git rebase -i ${commit_uuid}
```

Above command is an unfamiliar at first, but it is very useful for pretty commit history.
Basically, it is git rebase command with interaction.
Rebase supports multiple functions:

```bash
 Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit using the original merge commit's
# .       message (or the oneline, if no original merge commit was
# .       specified). Use -c <commit> to reword the commit message.
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# However, if you remove everything, the rebase will be aborted.
#
```

I will explain about the commit-related features that I use frequently: "p", "r", "s", "e"

`p`(pick) represents no changes.
Commits have p or pick remain still.
It is optional if you want to change the order of the commit, it can be done in `p`.

`r`(reword) rewords the commit log.
If there is a typos in commit or want to change the sentence in commit, `r` is for you.

`s`(squash) merges multiple commits to a single commit.
It is the important to manage the organized commit histories.
Since it is a little hard to understand only with the text, suppose there are some commits like:

`e`(edit) edits commit log.
It looks like very similar to reword, but has wider feature.
It can amend commit using `git commit --amend`.
`git commit --amend` can also modify the date, author not only the log itself.

```bash
$ git log --pretty=oneline
d085827ec8be53c408bbb5b28638040195ef55ac fix typos in changelog
67c842bca087447aa626c54326030c64386ba641 Update changelog
f1d3e06746393fcb9f07d120f04f76801e673486 Add new feature
d4254896637878e71901eb8f4fd061ca1a5c6063 Fix typos in unittest
69c6f96cc79bf6cacfaf189d32806cf6871d8fd4 Fix unittest
096ec8681a62ed81f125b3f620ab4623d4c9fa48 Add unittest
a30fe31c0609a6ecb9c08f9f4317b1edaa029dcc Add new feature
${some_base_commit_uuid}                 ${some_commit_text}
```

Lower commit represents the past and upper commit represents the recent commit.
As you can see, there are rebundant and duplicated commits.
"Add unittest", "Fix unittest", and "Fix typos in unittest" have no reason to be seperated especially there is a little changes between commits.
"Update changelog" and "fix typos in changelog" also have no reason to be exist independently.
So you can execute following commands and see the screen like this:

```bash
# git rebase -i ${some_base_commit_uuid}

pick a30fe31 Add new feature
pick 096ec86 Add unittest
pick 69c6f96 Fix unittest
pick d425489 Fix typos in unittest
pick f1d3e06 Add new feature
pick 67c842b Update changelog
pick d085827 fix typos in changelog
```

## Squash

Be careful that now the order of commits is vice versa (upper commit represents the past commit).
Since we want to merge some commits, it should be:

```bash
pick a30fe31 Add new feature
pick 096ec86 Add unittest
s    69c6f96 Fix unittest
s    d425489 Fix typos in unittest
pick f1d3e06 Add new feature
pick 67c842b Update changelog
s    d085827 fix typos in changelog
```

Following the `pick`s, there will be four commits remain after the rebase.
Commits with `s` will be merged with the commits with `pick`.
"Fix typos in changlog" commit will be merged with "Update Changelog".
And "Fix unittest", "Fix typos in unittest" commits will be merged with "Add unittest".
After you finish your edit, save and exit from vi.

Now there are two things can be happen.
One is the completion of rebase.
There is no conflicts between the commits, and squash succeed.

Another is the conflict.
If there are some conflicts between the commits you want to squash, you should solve it like merge request.
You should solve the conflicts of existing and incoming changes (usually, I select incoming changes only in my cases), and add the solved file with `git add`.
After then, you can continue your rebase with `git rebase --continue`.
If the conflicts remain, you will repeats above processes again.
If there is no conflict, its rebase will be completed.

## Edit

If you use private and public github accounts, you should config your local config in each directories with

```bash
git config --local user.name "name"
git config --local user.email "email"
```

However, since we are human, we forget to change name and email and put wrong author in commit.
In this case, you should change your author info in each commits.
`git rebase -i` also can be used to change these information with `edit`.

```bash
# git rebase -i ${some_commit_uuid}

edit d085827 fix typos in changelog
```

In `edit` mode, you can use only one command: git commit --amend.

```bash
git commit --amend --author="your_right_name <your_email@domain.com>"
git commit --amend --date="Mon Sep 1 44:44:44 2022"
```

After you update your commit information, you can continue with `git rebase --continue`.
