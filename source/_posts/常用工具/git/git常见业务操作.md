---
title: git常见业务操作
date: 2021-03-23 09:46:30
tags: git
---

# 从master分支中创建自己的远程分支

## First, you create your branch locally:

```
git checkout -b <branch-name> # Create a new branch and check it out
```

The remote branch is automatically created when you push it to the remote server. 

## So when you feel ready for it, you can do:

```
git push <remote-name> <branch-name> 
```

Where `<remote-name>` is typically `origin`, the name which git gives to the remote you cloned from. Your colleagues would then just pull that branch, and it's automatically created locally.

Note however that formally, the format is:

```
git push <remote-name> <local-branch-name>:<remote-branch-name>
```

But when you omit one, it assumes both branch names are the same. Having said this, as a word of **caution**, do not make the critical mistake of specifying only `:<remote-branch-name>` (with the colon), or the remote branch will be deleted!

## So that a subsequent `git pull` will know what to do, you might instead want to use:

```
git push --set-upstream <remote-name> <local-branch-name> 
```

As described below, the `--set-upstream` option sets up an upstream branch:

> For every branch that is up to date or successfully pushed, add upstream (tracking) reference, used by argument-less git-pull(1) and other commands.



## 验证

```shell
git branch -vv
* developing 3411786 [origin/developing] qifei
  master     3411786 [origin/master] qifei
```





# 或者

## Switch to a Branch That Came From a Remote Repo

1. To get a list of all branches from the remote, run this command:
   - **git pull**
2. Run this command to switch to the branch:
   - **git checkout --track origin/my-branch-name**

## Push to a Branch

- If your local branch does not exist on the remote, run either of these commands:
  - **git push -u origin my-branch-name**
  - **git push -u origin HEAD**

**NOTE:** HEAD is a reference to the top of the current branch, so it's an easy way to push to a branch of the same name on the remote. This saves you from having to type out the exact name of the branch!

- If your local branch

   

  already exists 

  on the remote, run this command:

  - **git push**

## Merge a Branch

1. You'll want to make sure your working tree is clean and see what branch you're on. Run this command:
   - **git status**
2. First, you must check out the branch that you want to merge another branch into (changes will be merged into this branch). If you're not already on the desired branch, run this command:
   - **git checkout master**
   - **NOTE:** Replace **master** with another branch name as needed.
3. Now you can merge another branch into the current branch. Run this command:
   - **git merge my-branch-name**
   - **NOTE:** When you merge, there may be a conflict. Refer to **Handling Merge Conflicts** (the next exercise) to learn what to do.