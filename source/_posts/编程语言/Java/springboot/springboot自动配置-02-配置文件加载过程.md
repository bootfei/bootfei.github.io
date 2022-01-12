---
title: springboot-02-自动配置
date: 2020-12-03 12:12:59
tags:
---

### 配置文件application.yml的加载

application.yml 文件对于 Spring Boot 来说是核心配置文件，至关重要，那么，该文件是 如何加载到内存的呢?需要从启动类的 run()方法开始跟踪。

- xxxApplication: 
  - SpringApplication.run(DemoApplication.class, args);
- SpringApplication:  
  - (new SpringApplication(primarySources)).run(args);
  - this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);
  - listeners.environmentPrepared(bootstrapContext, (ConfigurableEnvironment)environment);
- SpringApplicationRunListeners
  - listener.environmentPrepared(environment);
- EventPublishingRunListener
  - initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
- SimpleApplicationEventMulticaster
  - this.invokeListener(listener, event);
  - this.doInvokeListener(listener, event);
- ConfigFileApplicationListener

