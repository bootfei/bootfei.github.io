---
title: 查找CPU飙升的原因
date: 2020-11-05 09:24:42
tags: [java, jstack]
---

它山之石可以攻玉

### ref links:

1. 基础步骤
   https://www.javatt.com/p/65620





## bootFei:

1. ##### 首先，因为CPU飙升了，所以要查看CPU的相关信息，所以需要使用TOP命令
   
<img src="/Users/qifei/Library/Application Support/typora-user-images/image-20201202174921436.png" alt="image-20201202174921436" style="zoom:50%;" />
   
2. ##### 根据1st step中的返回内容，看到进程号116664的CPU很高。所以，需要进一步锁定该进程内部的线程耗费CPU, 所以需要使用top -H -p [pid]命令查看线程， (也可使用shift -h进行切换)

   ![image-20201202175704489](/Users/qifei/Library/Application Support/typora-user-images/image-20201202175704489.png)

   

3. ##### 根据2nd step中的返回内容，可以看到线程号117296(10进制)的java线程耗费CPU，所以需要查看该线程的详细信息，所以需要是一个jstack命令，但是，记住jstack中的nid（Native Thread ID)是系统线程id, 为16进制，需要使用top -Hp pid找到该线程的10进制id，然后使用下边的命令打印出16进制线程id

   ![image-20201202180132749](/Users/qifei/Library/Application Support/typora-user-images/image-20201202180132749.png)

   ```shell
     printf  "%x\n" 10进制tid
   ```

   

4. ##### 根据3rd step中的16进制线程号，使用jstack pid | grep [nid]命令，查询该线程的详细信息

   ![image-20201202180409819](/Users/qifei/Library/Application Support/typora-user-images/image-20201202180409819.png)

   











