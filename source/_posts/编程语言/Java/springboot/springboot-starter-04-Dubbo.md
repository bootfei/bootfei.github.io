---
title: springboot-starter-03-Dubbo
date: 2021-05-12 12:53:52
tags:
---



  ### 步骤

  - 消费者与提供者工程均需要导入四个依赖 
    -  Dubbo 与 Spring Boot 整合依赖
    -  zkClient依赖
    -  slf4j-log4j12依赖

  - 自定义 commons 工程依赖 
  - 提供者工程
    - 将 Service 接口实现类的@Service 注解更换为阿里的注解，并添加@Component 注解
    - 在启动类上添加@EnableDubboConfiguration 与@EnableTransactionManager 注解
    - 修改配置文件:指定应用名称与注册中心地址 
  - 消费者工程
    - 将处理器中 Service 的声明上的@Autowired 注解更换为阿里的@Reference 注解 
    - 在启动类上添加@EnableDubboConfiguration 注解
    - 修改配置文件:指定应用名称与注册中心地址



### 定义 **commons** 工程

- 依赖：无，因为是纯java项目
- 定义实体类
- 定义业务接口

### 定义提供者

- 依赖：

  - 添加 **dubbo** 与 **spring boot** 整合依赖，需要从alibaba的github中找到依赖
  - 添加 **zkClient** 依赖
  - dubboCommons依赖
  - 还有其他的mysql, druid,mybatis依赖

- 定义业务接口

  - 将 Service 接口实现类的@Service 注解更换为阿里的注解，并添加@Component 注解

- 修改启动类

  - 添加@EnableDubboConfiguration 与@EnableTransactionManager 注解

- 修改配置文件

  - ```yaml
    spring:
      # 功能等价于 spring-dubbo 配置文件中的<dubbo:application/> # 该名称是由服务治理平台使用
      application:
      	name: 11-provider-springboot # 指定zk注册中心
      dubbo:
      	registry: zookeeper://zkOS:2181
      # zk 集群作注册中心
      # registry: zookeeper://zkOS1:2181?backup=zkOS2:2181,zkOS3:2181
    ```

### 定义消费者(略)

- 依赖：

  - 添加 **dubbo** 与 **spring boot** 整合依赖，需要从alibaba的github中找到依赖
  - 添加 **zkClient** 依赖
  - dubboCommons依赖

- 修改配置文件

  - ```yaml
    spring:
      # 功能等价于 spring-dubbo 配置文件中的<dubbo:application/> # 该名称是由服务治理平台使用
      application:
      	name: 11-consumer-springboot # 指定zk注册中心
      dubbo:
      	registry: zookeeper://zkOS:2181
      # zk 集群作注册中心
      # registry: zookeeper://zkOS1:2181?backup=zkOS2:2181,zkOS3:2181
    ```

    



## 