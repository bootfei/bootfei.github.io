---
title: GitHub上搭建hexo博客
date: 2020-10-02 20:34:45
tags: [hexo]
categories: [前端]
---

它山之石可以攻玉

### ref links:

 在GitHub上搭建hexo博客:

1. https://juejin.im/post/6844903887732736007

保存hexo博客源码: 

1. https://blog.csdn.net/gatieme/article/details/82317704

添加子标签并且对标签分类:

1. https://stackoverflow.com/questions/46194621/grouping-categories-in-hexo
2. https://github.com/hexojs/hexo/pull/2734





### 需要注意的点

1. ssh-keygen -t rsa -C "邮件地址" 的命令并不适用于mac，具体适配的命令参考https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/connecting-to-github-with-ssh

2. `hexo d`的话一般会报如下错误：

   ```shell
   Deployer not found: github 或者 Deployer not found: git
   ```
   
   原因是还需要安装一个插件：

   ```shell
   npm install hexo-deployer-git --save
   ```

3. 使用git分支做备份的时候，需要注意
   a. `hexo init` 只能用于empty folder，所以如果folder有git了，需要把.git文件拷贝出来，把folder清空, 然后执行hexo init, 再.git复制回来

   b. 最好使用git clone下载备份代码