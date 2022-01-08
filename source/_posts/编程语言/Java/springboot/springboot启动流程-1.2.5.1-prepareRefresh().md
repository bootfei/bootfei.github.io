---
title: springboot启动流程-01-启动流程
date: 2021-04-16 09:30:13
tags: [java,springboot,startup]
---



```java
protected void prepareRefresh() {
        //记录容器启动时间，然后设立对应的标志位
        this.startupDate = System.currentTimeMillis();
        this.closed.set(false);
        this.active.set(true);
        // 打印info日志：开始刷新当前容器了
        if (logger.isInfoEnabled()) {
            logger.info("Refreshing " + this);
        }

        // 这是扩展方法，由子类去实现，可以在验证之前为系统属性设置一些值可以在子类中实现此方法
        // 因为我们这边是AnnotationConfigApplicationContext，可以看到不管父类还是自己，都什么都没做，所以此处先忽略
        initPropertySources();

        //属性文件验证，确保需要的文件都已经放入环境中
        getEnvironment().validateRequiredProperties();

        //初始化容器，用于装载早期的一些事件
        this.earlyApplicationEvents = new LinkedHashSet<>();
    } 
```
