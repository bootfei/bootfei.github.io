---
title: 'spring-cloud-8.消息系统整合框架 Spring Cloud Stream'
date: 2020-12-12 15:40:07
tags:
---

> spring boot本来就整合kafka，为什么spring cloud为什么还要整合kafka呢？
>
> 因为spring boot使用kafka很麻烦；

# 简介

一个轻量级的事件驱动微服务框架，用于快速构建可连接到外部系统的应用程序。 在 Spring Boot 应用程序之间使用 Kafka 或 RabbitMQ 发送和接收消息的简单声明式模型。

Spring Cloud Stream 是一个用来为微服务应用构建消息驱动能力的框架。通过使用 Spring Cloud Stream，可以有效简化开发人员对消息中间件的使用复杂度，让系统开发人员 可以有更多的精力关注于核心业务逻辑的处理。但是目前 Spring Cloud Stream 只支持 RabbitMQ 和 Kafka 的自动化配置。

# 原理模型

应用程序的核心部分(Application Core)通过 inputs 与 outputs 管道，与中间件连接， 而管道是通过绑定器 Binder 与中间件相绑定的。

# 具体实现

## 创建消息生产者

> 总步骤
>
> - 导入spring-cloud-stream-binder-kafka依赖
> - 创建消息生产者类。在生产者类上添加@EnableBinding()注解，并声明管道 
> - 定义处理器，在处理器中调用消息生产者，使其发送消息
> - 在配置文件中注册kafka集群，并指定管道所绑定的主题及类型

- 添加依赖

  - ```xml
    <dependency>
      <groupId>org.springframework.cloud</groupId> 
      <artifactId>spring-cloud-stream-binder-kafka</artifactId>
    </dependency>
    ```

- 创建消息生产者类

  - 添加@EnableBinding()注解，并声明管道 

  - 管道(outputs)可以自定义, 仿照Source写就可以

    - ```java
      public interface CustomSource {
          String CHANNEL_NAME = "xxx";
      
          @Output(CustomSource.CHANNEL_NAME)
          MessageChannel output();
      }
      ```

    - [有了自定义outputs, 那么一个消息就可以通过多个outputs可以发给多个主题]()

  - ```java
    @Component
    // 将MQ与生产者类通过消息管道相绑定
    @EnableBinding({Source.class,CustomSource.class})
    public class SomeProducer {
        // 必须使用byName方式的自动注入,因为还有Source.ERROR，所以使用byName
        @Autowired
        @Qualifier(Source.OUTPUT)
        private MessageChannel channel;
    
        @Autowired
        @Qualifier(Source.OUTPUT)
        private MessageChannel customedChannel;
      
        public String sendMessage(String msg) {
            // 通过消息管道outputs发送消息，即消息写入到消息管道outputs，再通过消息管道写入到MQ
            channel.send(MessageBuilder.withPayload(msg).build());
          	customedChannel.send(MessageBuilder.withPayload(msg).build());
            return msg;
        }
    }
    ```

- 修改Controller, 调用消息生产者，使其发送消息

  - ```java
    @RestController
    public class SomeController {
        // 将生产者注入
        @Autowired
        private SomeProducer producer;
    
        @PostMapping("/msg/send")
        public String sendHandler(@RequestParam("message") String msg) {
            // 生产者发送消息
            return producer.sendMessage(msg);
        }
    }
    ```

- 修改配置文件

  - ```yaml
    spring:
      application:
        name: abcmsc-consumer-depart
    
      cloud:
        stream:
          kafka:
            binder:
              # 指定要连接的kafka集群
              brokers: kafkaOS1:9092,kafkaOS2:9092,kafkaOS3:9092
              # 指定是否自动创建主题
              auto-create-topics: true
          bindings: #绑定了2个管道output：output和xxx
            output:  # 指定要绑定的输出管道，及要输出到单管道中的消息主题及类型
              destination: names #kafka topic
              content-type: text/plain
             
    				xxx: 
              destination: cities
              content-type: text/plain
    ```

    

## 创建消息消费者

Spring Cloud Stream 提供了三种创建消费者的方式，这三种方式的都是在消费者类的“消 费”方法上添加注解。只要有新的消息写入到了管道，该“消费”方法就会执行。只不过三种注解，其底层的实现方式不同。即当新消息到来后，触发“消费”方法去执行的实现方式 不同。

- @PostConstruct:以发布/订阅方式实现
- @ServiceActivator:以新消息激活服务的方式实现 
- @StreamListener:以监听方式实现

> 总步骤
>
> - 导入spring-cloud-stream-binder-kafka依赖
> - 创建消费者类。在消费者类上添加@EnableBinding()注解，并声明管道。 在消费者类中定义消费方法，在方法上添加相应的注解。
> - 在配置文件中注册kafka集群，并指定管道所绑定的主题

- 添加依赖（同上）

- 创建消费者类

  - 只以@PostConstruct为例

  - ```java
    @Component
    @EnableBinding({Sink.class})
    public class SomeConsumer {
        @Autowired
        @Qualifier(Sink.INPUT)
        private SubscribableChannel channel;
      
        @PostConstruct
        public void printMessage() {
            channel.subscribe(msg -> {
                // MessageHeaders headers = msg.getHeaders();
                System.out.println(new String((byte[])msg.getPayload()));
            });
        }
    }
    ```

- 修改配置文件

  - ```yaml
    spring:
      cloud:
        stream:
          kafka:
            binder:
              # 指定要连接的kafka集群
              brokers: kafkaOS1:9092,kafkaOS2:9092,kafkaOS3:9092
              # 指定是否自动创建主题
              auto-create-topics: true
          bindings:
            # 指定要绑定的输入管道input，及要消费的管道中的消息主题
            input:
              destination: names
    ```

    

# 附录

## spring cloud stream原理图

![Image result for spring cloud steam](data:image/png;base64,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)

