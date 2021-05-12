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
   ​    使用该方式会在本地新建分支x，并自动切换到该本地分支x。采用此种方法建立的本地分支会和远程分支建立映射关系。
   
2. 远程仓库

   a. 查看远程分支

   ```
   git remote -v
   ```

   b. 从HTTPs切换到SSH，或者反过来

   ```shell
   #见官网https://docs.github.com/en/github/using-git/changing-a-remotes-url
   git remote set-url origin git@github.com:USERNAME/REPOSITORY.git
   ```

   

### 版本回退

1. 查看版本日志

   ```bash
   git log --pretty=oneline
   ```

2. 版本回退

   ```bash
   
   ```



撤销工作区改动：

```
git checkout -- 文件名
```

清空staged 、版本回退：

```
git reset HEAD 文件名 #清空staged区域

git reset --hard 1094a #回退到指定版本
git reset --hard HEAD^ #回退到上一版本
```





### **合并相关**

#### merge

merge是最常用的合并命令，它可以将某个分支或者某个节点的代码合并至当前分支。具体命令如下：

```
git merge 分支名/节点哈希值
```

如果需要合并的分支完全领先于当前分支，如图3所示：

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNluW6taqv3LemRM5sq6GFejXjGic5hcsmiaJBwbpWyjRcrsqqIQ0nn42czhloUicqldA5B6iawSLtsz7g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

由于分支ft-1完全领先分支ft-2即ft-1完全包含ft-2，所以ft-2执行了“git merge ft-1”后会触发fast forward(快速合并)，此时两个分支指向同一节点，这是最理想的状态。

但是实际开发中我们往往碰到的是下面这种情况：如图4（左）。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNluW6taqv3LemRM5sq6GFejLuN05oovuOzTRatAVOf1bSkNpFqnyIRx3ibiaIUYNHbeBx6r4aicmbfSg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

这种情况就不能直接合了，当ft-2执行了“git merge ft-1”后Git会将节点C3、C4合并随后生成一个新节点C5，最后将ft-2指向C5 如图4（右）。

注意点：如果C3、C4同时修改了同一个文件中的同一句代码，这个时候合并会出错，因为Git不知道该以哪个节点为标准，所以这个时候需要我们自己手动合并代码。



#### rebase

rebase也是一种合并指令，命令行如下：

```
git rebase 分支名/节点哈希值
```

与merge不同的是rebase合并看起来不会产生新的节点（实际上是会产生的，只是做了一次复制），而是将需要合并的节点直接累加，如图5。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNluW6taqv3LemRM5sq6GFejYqzToG8tUbvFKtquRVZA0xguWERGL1yeV1kuB7NVEEGwEwnBThlE1g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

当左边示意图的ft-1.0执行了git rebase master后会将C4节点复制一份到C3后面，也就是C4'，C4与C4'相对应，但是哈希值却不一样。

rebase相比于merge提交历史更加线性、干净，使并行的开发流程看起来像串行，更符合我们的直觉。既然rebase这么好用是不是可以抛弃merge了？其实也不是了，下面我罗列一些merge和rebase的优缺点：

merge优缺点：

- 优点：每个节点都是严格按照时间排列。当合并发生冲突时，只需要解决两个分支所指向的节点的冲突即可
- 缺点：合并两个分支时大概率会生成新的节点并分叉，久而久之提交历史会变成一团乱麻

rebase优缺点：

- 优点：会使提交历史看起来更加线性、干净
- 缺点：虽然提交看起来像是线性的，但并不是真正的按时间排序，比如图3-3中，不管C4早于或者晚于C3提交它最终都会放在C3后面。并且当合并发生冲突时，理论上来讲有几个节点rebase到目标分支就可能处理几次冲突



对于网络上一些只用rebase的观点，作者表示不太认同，如果不同分支的合并使用rebase可能需要重复解决冲突，这样就得不偿失了。但如果是本地推到远程并对应的是同一条分支可以优先考虑rebase。所以我的观点是 根据不同场景合理搭配使用merge和rebase，如果觉得都行那优先使用rebase。



#### cherry-pick

cherry-pick的合并不同于merge和rebase，它可以选择某几个节点进行合并，如图6。

命令行：

```
git cherry-pick 节点哈希值
```

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNluW6taqv3LemRM5sq6GFejt1bVibO8aVypicn3Y3hDqdVSTVFvBA1TOU5fYZnYJnib6PIMQP5GOmuUA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

图6



假设当前分支是master，执行了git cherry-pick C3(哈希值)，C4(哈希值)命令后会直接将C3、C4节点抓过来放在后面，对应C3'和C4'。



### **回退相关**

#### 分离HEAD

在默认情况下HEAD是指向分支的，但也可以将HEAD从分支上取下来直接指向某个节点，此过程就是分离HEAD，具体命令如下：

```
git checkout 节点哈希值
//也可以直接脱离分支指向当前节点
git checkout --detach
```

由于哈希值是一串很长很长的乱码，在实际操作中使用哈希值分离HEAD很麻烦，所以Git也提供了HEAD基于某一特殊位置（分支/HEAD）直接指向前一个或前N个节点的命令，也即相对引用，如下：

```
//HEAD分离并指向前一个节点
git checkout 分支名/HEAD^
//HEAD分离并指向前N个节点
git checkout 分支名～N
```

将HEAD分离出来指向节点有什么用呢？举个例子：如果开发过程发现之前的提交有问题，此时可以将HEAD指向对应的节点，修改完毕后再提交，此时你肯定不希望再生成一个新的节点，而你只需在提交时加上--amend即可，具体命令如下：

```
git commit --amend
```

#### 回退

回退场景在平时开发中还是比较常见的，比如提交后，发现写的有问题，于是你想将代码回到前一个提交，这种场景可以通过reset解决，具体命令如下：

```
//回退N个提交
git reset HEAD~N
```

reset和相对引用很像，区别是reset会使分支和HEAD一并回退。



### **远程相关**

当我们接触一个新项目时，第一件事情肯定是要把它的代码拿下来，在Git中可以通过clone从远程仓库复制一份代码到本地，具体命令如下：

```
git clone 仓库地址
```

前面的章节我也有提到过，clone不仅仅是复制代码，它还会把远程仓库的引用（分支/HEAD）一并取下保存在本地，如图7所示：



![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNluW6taqv3LemRM5sq6GFejv3TdGoicPz4yiaaqj7WeiaGfL2rCE0rOx9AMzjuHjsALlick8MbA356qPw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

图7



其中origin/master和origin/ft-1为远程仓库的分支，而远程的这些引用状态是不会实时更新到本地的，比如远程仓库origin/master分支增加了一次提交，此时本地是感知不到的，所以本地的origin/master分支依旧指向C4节点。我们可以通过fetch命令来手动更新远程仓库状态。



小提示：并不是存在服务器上的才能称作是远程仓库，你也可以clone本地仓库作为远程，当然实际开发中我们不可能把本地仓库当作公有仓库，说这个只是单纯的帮助你更清晰的理解分布式。



fetch：



说的通俗一点，fetch命令就是一次下载操作，它会将远程新增加的节点以及引用(分支/HEAD)的状态下载到本地，具体命令如下：

```
git fetch 远程仓库地址/分支名
```

pull：



pull命令可以从远程仓库的某个引用拉取代码，具体命令如下：

```
git pull 远程分支名
```

其实pull的本质就是fetch+merge，首先更新远程仓库所有状态到本地，随后再进行合并。合并完成后本地分支会指向最新节点。



另外pull命令也可以通过rebase进行合并，具体命令如下：

```
git pull --rebase 远程分支名
```