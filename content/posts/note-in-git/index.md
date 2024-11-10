---
date: '2022-05-28T14:32:58+07:00'
draft: true
title: 'Note in Git'
---
## What is a branch?
A branch is just a pointer that point to a specific commit. Imagine your repository as a graph of commits, commits are identified by hash strings and be connected together. When you create a new branch, git creates a new pointer and points to the selected commit. Even when you delete a branch, the commit is still there.

## What is the HEAD pointer?
`HEAD` point to the current commit you are standing. By default `HEAD` point to the newest commit of the current branch. When you checkout to a specific commit, `HEAD` will change to point to that commit too.

## Git revert vs reset?

![Revert](revert.png 'Revert')
Git-revert will create a new commit to revert a previous commit, while git-reset will move your HEAD pointer and branch to the selected commit.

`git revert HEAD`: delete current commit

`git revert HEAD^`: delete parent of the current commit

![Reset](reset.png 'Reset')
When your branch is shared with others then you want to do a git-revert, otherwise git-reset would be fined.

`git reset HEAD^`: checkout to `HEAD^` commit – parent of the current commit, reassign branch refs to `HEAD^`.

## Git reset –hard, –mixed, and –soft
[![stackoverflow](reset-modes.png 'stackoverflow')](https://stackoverflow.com/questions/3528245/whats-the-difference-between-git-reset-mixed-soft-and-hard)

- `soft`: uncommit your changes, your changes are now left staged (index).
- `mixed` (default): uncommit + unstaged changes, changes are left in the working tree.
- `hard`: uncommit + unstaged + delete changes, nothing left.

## Redo git reset hard
If you accidentally reset your branch, you still can revert that change by using git `reflog`.
![](reflog.png 'after git reset –hard HEAD^')

`3be1a3c`: this is the current `HEAD`

`df61ddd`: this is the commit that I lost

`git reset --hard df61ddd` this command will bring back the commit that I lost.
## Git rebase vs merge?
![](merge.png 'Merge')
Git merge preserved your history, it will create a new commit to combine to branch, therefore you might notice that a merge commit will have two parents.

`(master*) git merge feature`
![](rebase.png 'Rebase')
Git rebase otherwise will edit the history of the target branch, it will make new commits from the source branch, one by one, and append to the target branch(new base), which makes your history look like a straight line. Rebase needs to create new commits because you need to resolve conflict, which makes the new commit doesn’t look like the original commit.

`(feature*) git rebase master`: git will make new commits based on the **feature** branch and append to **master**

`(master*) git rebase feature`: this will fast-forward **master** point to the same commit (newest) with the **feature**.

## Git rebase-interactive and cherry-pick
git rebase-interactive let you decide what to do with every commits, git will create new commits to replace after that.

You can edit commit, edit commit message, squash commits, drop a commit,…

`(develop*) git rebase -i HEAD~5`: this is used when you want to edit a previous commit on the current branch.

`(develop*) git rebase -i main`: this is to rebase develop into main, but it will let you modify commit when rebasing.

git cherry-pick helps you make a copy of a commit from another branch and append it to your current branch.