---
title: git分支管理
date: 2020-10-03 09:46:30
tags: git
---



#### 本地和远程建立联系

```shell
#本地git和远程仓库建立联系
git remote add origin git@github.com:bootfei/bootfei.github.io.git
#git 新本地-分支master
git branch -M master
#git 新本地-分支master，推送到远程-仓库，并建立远程-分支master
git push -u origin master
```



#### 远程仓库

a. 查看远程分支

```
git remote -v
```

b. 从HTTPs切换到SSH，或者反过来

```shell
#见官网https://docs.github.com/en/github/using-git/changing-a-remotes-url
git remote set-url origin git@github.com:USERNAME/REPOSITORY.git
```

