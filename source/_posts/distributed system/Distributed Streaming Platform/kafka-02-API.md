---
title: kafka-03-API
date: 2021-02-13 17:06:11
tags:
---

# 首先在kafka命令行

创建一个名称为 cities 的主题，并创建该主题的订阅者。

```shell
$ bin/kafka-topics.sh --create --bootstrap-server 192.168.59.152:9092 partitions 1 --topic cities #创建cities topic

$ bin/kafka-topics.sh --list --bootstrap-server 192.168.59.153:9092 ##检查是否创建成功
```



# 使用 **kafka** 原生 **API**

创建一个 Maven 的 Java 工程

## 导入依赖

- ```xml
  <!-- kafka 依赖 -->
  <dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka_2.12</artifactId>
    <version>1.1.1</version>
  </dependency>
  ```

## 创建Producer

### [支持批量发送，定时发送，支持回调]()

- 创建Producer
- api方式中的key会被kafka取模，key%kafka的数量 = 发送到的partition；如果不指定key，那么采用轮训的方式

```java
public class OneProducer {
    // 第一个泛型：当前生产者所生产消息的key
    // 第二个泛型：当前生产者所生产的消息本身
    private KafkaProducer<Integer, String> producer;

    public OneProducer() {
      Properties properties = new Properties();
      // 指定kafka集群
      properties.put("bootstrap.servers", "kafkaOS1:9092,kafkaOS2:9092,kafkaOS3:9092");
      // 指定key与value的序列化器
      properties.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
      properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
      
      // 批量发送：指定生产者每10条向broker发送一次，但是send方法还是被调用多次，只不过send将数据缓存了，再一次性发送
      properties.put("batch.size", 10);
      // 批量发送：指定生产者每50ms向broker发送一次，
      properties.put("linger.ms", 50);

      this.producer = new KafkaProducer<Integer, String>(properties);
    }

    public void sendMsg() {
      // 创建消息记录（包含主题、消息本身） 
      // 创建消息记录（包含主题、key、消息本身）  
      // 创建消息记录（包含主题、partition、key、消息本身）  
     for(int i=0; i<50; i++) {
            ProducerRecord<Integer, String> record = new ProducerRecord<>("cities", "city-" + i);
            int k = i;
            producer.send(record, (metadata, ex) -> {
                System.out.println("i = " + k);
                System.out.println("topic = " + metadata.topic());
                System.out.println("partition = " + metadata.partition());
                System.out.println("offset = " + metadata.offset());
            });
        }
    }
}	
```

## 创建Consumer

 (**1**) 自动提交的问题

前面的消费者都是以自动提交 offset 的方式对 broker 中的消息进行消费的，但自动提交 可能会出现消息重复消费的情况。所以在生产环境下，很多时候需要对 offset 进行手动提交， 以解决重复消费的问题。

(**2**) 手动提交分类

手动提交又可以划分为[同步提交、异步提交、同异步联合]()提交。这些提交方式仅仅是 doWork()方法不相同，其构造器是相同的。所以下面首先在前面消费者类的基础上进行构造 器的修改，然后再分别实现三种不同的提交方式。

### 消费者同步手动提交

```java
public class SyncAsyncManualConsumer extends ShutdownableThread {
    private KafkaConsumer<Integer, String> consumer;

    public SyncAsyncManualConsumer() {
        // 两个参数：
        // 1)指定当前消费者名称
        // 2)指定消费过程是否会被中断
        super("KafkaConsumerTest", false);

        Properties properties = new Properties();
        String brokers = "kafkaOS1:9092,kafkaOS2:9092,kafkaOS3:9092";
        // 指定kafka集群
        properties.put("bootstrap.servers", brokers);
        // 指定消费者组ID
        properties.put("group.id", "cityGroup1");

        // 开启手动提交
        properties.put("enable.auto.commit", "false");
        // 指定自动提交的超时时限，默认5s
        // properties.put("auto.commit.interval.ms", "1000");
        // 指定一次提交10个offset
        properties.put("max.poll.records", 10);

        // 指定消费者被broker认定为挂掉的时限。若broker在此时间内未收到当前消费者发送的心跳，则broker
        // 认为消费者已经挂掉。默认为10s
        properties.put("session.timeout.ms", "30000");
        // 指定两次心跳的时间间隔，默认为3s，一般不要超过session.timeout.ms的 1/3
        properties.put("heartbeat.interval.ms", "10000");
        // 当kafka中没有指定offset初值时，或指定的offset不存在时，从这里读取offset的值。其取值的意义为：
        // earliest:指定offset为第一条offset
        // latest: 指定offset为最后一条offset
        properties.put("auto.offset.reset", "earliest");
        // 指定key与value的反序列化器
        properties.put("key.deserializer",
                "org.apache.kafka.common.serialization.IntegerDeserializer");
        properties.put("value.deserializer",
                "org.apache.kafka.common.serialization.StringDeserializer");

        this.consumer = new KafkaConsumer<Integer, String>(properties);
    }

    @Override
    public void doWork() {
        // 订阅消费主题
        consumer.subscribe(Collections.singletonList("cities"));
        // 从broker摘取消费。参数表示，若buffer中没有消费，消费者等待消费的时间。
        // 0，表示没有消息什么也不返回
        // >0，表示当时间到后仍没有消息，则返回空
        ConsumerRecords<Integer, String> records = consumer.poll(1000);
        for(ConsumerRecord record : records) {
            System.out.println("topic = " + record.topic());
            System.out.println("partition = " + record.partition());
            System.out.println("key = " + record.key());
            System.out.println("value = " + record.value());
            consumer.commitAsync((offsets, ex) -> {
                if(ex != null) {
                    System.out.print("提交失败，offsets = " + offsets);
                    System.out.println(", exception = " + ex);

                    // 同步提交
                    consumer.commitSync();
                }
            });
        }
    }
}
```

手动同步提交：  `properties.put("enable.auto.commit", "false");`  + `consumer.commitSync()`;



### 消费者异步手动提交

手动同步提交方式需要等待 broker 的成功响应，效率太低，影响消费者的吞吐量。异步提交方式是，消费者向 broker 提交 offset 后不用等待成功响应，所以其增加了消费者的吞吐量。

```java
consumer.commitAsync((offsets, ex) -> {
                if(ex != null) {
                    System.out.print("提交失败，offsets = " + offsets);
                    System.out.println(", exception = " + ex);
                }
            });
```

### 消费者同异步手动提交

同异步提交，即同步提交与异步提交组合使用。一般情况下，在异步手动提交时，若偶尔出现提交失败，其也不会影响消费者的消费。因为后续提交最终会将这次提交失败的 offset 给提交了。

但异步手动提交会产生重复消费，为了防止重复消费，可以将同步提交与异常提交联合 使用。

```java
consumer.commitAsync((offsets, ex) -> {
                if(ex != null) {
                    System.out.print("提交失败，offsets = " + offsets);
                    System.out.println(", exception = " + ex);

                    // 同步提交
                    consumer.commitSync();
                }
            });
```

# 使用**Spring Boot Kafka**



## 导入依赖

```
<dependency>
<groupId>org.springframework.kafka</groupId>
<artifactId>spring-kafka</artifactId>
</dependency>
```

## 配置文件

```yaml
# 自定义属性
kafka:
  topic: cities

# 配置Kafka
spring:
  kafka:
    bootstrap-servers: kafkaOS1:9092,kafkaOS2:9092,kafkaOS3:9092
    # producer:   # 配置生产者
      # key-serializer: org.apache.kafka.common.serialization.StringSerializer
      # value-serializer: org.apache.kafka.common.serialization.StringSerializer

    consumer:   # 配置消费者
      group-id: group0  # 消费者组
      # key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      # value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
```

 key-deserializer和value-deserializer不需要配置，因为自动装配类中已经设置了该默认值

<!--spring-cloud-autoconfig包下，找到spring.factories,再从中找到kafka的自动装配类，最后有EnableProperties注解，可以看到该默认配置的值-->

## Producer

Spring 是通过 KafkaTemplate 来完成对 Kafka 的操作的

```java
@RestController
public class SomeProducer {
    @Autowired
    private KafkaTemplate<String, String> template;
    // 从配置文件读取自定义属性
    @Value("${kafka.topic}")
    private String topic;

    // 由于是提交数据，所以使用Post方式
    @PostMapping("/msg/send")
    public String sendMsg(@RequestParam("message") String message) {
        template.send(topic, message);
        return "send success";
    }
}
```
## Consumer

Spring Kafka 是通过 KafkaListener 监听方式来完成消息订阅与接收的。当监听到有指定主题的消息时，就会触发@KafkaListener 注解所标注的方法的执行。

```java
@Component
public class SomeConsumer {

    @KafkaListener(topics = "${kafka.topic}")
    public void onMsg(String message) {
        System.out.println("Kafka消费者接受到消息 " + message);
    }

}
```

