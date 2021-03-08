---
title: git常见命令
date: 2020-10-03 09:46:30
tags: git
---

1. 远程分支

   a.  查看远程分支 

   ``` bash
   git branch -r
   ```
   
   b. 拉取远程分支并创建本地分支
   ``` bash
   git checkout -b 本地分支名x origin/远程分支名x
   ```
      使用该方式会在本地新建分支x，并自动切换到该本地分支x。采用此种方法建立的本地分支会和远程分支建立映射关系。
   
2. 版本回退

   1. 查看版本日志

      ```bash
      git log --pretty=oneline
      ```

   2. 版本回退

      ```bash
      git reset --hard 1094a #回退到指定版本
      git reset --hard HEAD^ #回退到上一版本
      ```