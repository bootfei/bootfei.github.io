---
title: java concurrnecy in Practice - 1
date: 2020-10-23 08:47:23
tags:
---







1. Race condition: 由于多线程之间不恰当的执行时序，而出现的不正确结果

   1. 比如: 没有同步的情况下，使用实例遍历count统计servlet的调用次数，count++就会出现race condition;
      这是一个"读取-修改-写入"的操作序列，并且结果依赖于之前的状态

   2. 最常见的race condition: Check Then Act（先检查后执行）

      1. 比如观察某个条件为真（比如文件X不存在），然后根据这个条件作出相应的动作（创建文件X）
      2. 但实际上观察 与 动作之间，观察的结果可能已经无效（另一个线程正在创建文件X）

   3. Check Then Act的一个示例：延迟初始化

      1. ```java
         //这段代码的执行代价很高
         public class LazyInitRace{
           private ExpensiveObject instance = null;
           public ExpensiveObject getInstance(){
             if(instance == null)
               instance = new ExpensiveObject();
             return instance;
           }
         }
         ```

      2. 线程A和线程B同时执行getInstance();

         1. A看到instance为空，则开始初始化
         2. B检查instance是否为空，取决于A是否初始化完成；如果A需要花很长时间初始化，那么B检查到instance是空，也开始初始化

