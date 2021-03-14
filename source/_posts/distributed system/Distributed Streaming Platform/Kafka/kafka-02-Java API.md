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

- 创建Producer
- 如果不指定partiton，api方式中的key会被kafka取模，key%kafka的数量 = 发送到的partition；如果不指定key，那么采用轮训的方式

### 单次发送

#### 代码

```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import java.util.Properties;
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

        this.producer = new KafkaProducer<Integer, String>(properties);
    }

    public void sendMsg() {
        // 创建消息记录（包含主题、消息本身）  (String topic, V value) 不指定partition和key，采用轮训方式
        ProducerRecord<Integer, String> record = new ProducerRecord<>("cities", "tianjin");
        
        // 创建消息记录（包含主题、key、消息本身）  (String topic, K key, V value) 指定key，采用key取模指定partition
        ProducerRecord<Integer, String> record = new ProducerRecord<>("cities", 1, "tianjin");
        
        // 创建消息记录（包含主题、partition、key、消息本身）  (String topic, Integer partition, K key, V value)
        ProducerRecord<Integer, String> record = new ProducerRecord<>("cities", 0, 1, "tianjin");
        producer.send(record);
    }
}	
```

#### 验证

使用kafka tools工具，并且把key和value设置成String，即可查看具体的消息

- 不指定partition和key （[实验结果本应该是轮询，但是没有复现出来，反而都是在一个partition上]()）

  | partition                   | 消息id  |
  | --------------------------- | ------- |
  | partition0（位于broker0上） | record1 |
  | partition1（位于broker1上） | record2 |
  | partition2（位于broker2上） | record3 |

- 不指定partition，但是指定key = 1，则1%3 = 1，即第一个partition

  | partition                   | 消息id                  |
  | --------------------------- | ----------------------- |
  | partition0（位于broker0上） | record1,record2,record3 |
  | partition1（位于broker1上） | null                    |
  | partition2（位于broker2上） | null                    |

- 指定partition=1，那么key失效

  | partition                   | 消息id                  |
  | --------------------------- | ----------------------- |
  | partition0（位于broker0上） | null                    |
  | partition1（位于broker1上） | record1,record2,record3 |
  | partition2（位于broker2上） | null                    |

### 支持回调

当broker收到producer的发送信息时，producer做出反应。好处就是保障信息已经投递给broker集群了。

#### 代码

只修改sendMsg()方法就可以：send(record, CallBack函数)

```java
	public void sendMsg() {
        // 创建消息记录（包含主题、partition、key、消息本身）  (String topic, Integer partition, K key, V value)
        ProducerRecord<Integer, String> record = new ProducerRecord<>("cities", 2, 1, "tianjin");
    	//使用lambda表达式表示callBack函数
        producer.send(record, (metadata, ex) -> {
            System.out.println("topic = " + metadata.topic());
            System.out.println("partition = " + metadata.partition());
            System.out.println("offset = " + metadata.offset());
        });
    }
```

#### 验证

每次发送完数据，broker接收到数据以后，producer都会在console打印出topic\partitin\offset信息

### 批量发送和定时发送

批量发送可不是for循环发送数据，这还是单次发送；批量发送乃是把数据攒起来，一次发送所有攒起来的数据。

- batch.size： 官方解释These buffers are of a size specified by the *batch*.*size* config，其实就是buffer大小

- producer.send()方法还是调用10次，但是它并有实际发送，而是将数据存储起来，第10次才发送

#### 代码

只需要改properties文件就可以。加上batch.size参数

```java
public SomeProducerBatch() {
        Properties properties = new Properties();
        // 指定kafka集群
        properties.put("bootstrap.servers", "kafkaOS1:9092,kafkaOS2:9092,kafkaOS3:9092");
        // 指定key与value的序列化器
        properties.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    
    //新增参数
        // 指定生产者每10条向broker发送一次
        properties.put("batch.size", 1000);
        // 指定生产者每50ms向broker发送一次
        properties.put("linger.ms", 50);

        this.producer = new KafkaProducer<Integer, String>(properties);
    }
```

#### 验证

发送3条数据，没有达到batch.size=1000字节，发现数据并没有发送

## 创建Consumer

 (**1**) 自动提交的问题

前面的消费者都是以自动提交 offset 的方式对 broker 中的消息进行消费的，但自动提交 可能会出现消息重复消费的情况。所以在生产环境下，很多时候需要对 offset 进行手动提交， 以解决重复消费的问题。

(**2**) 手动提交分类

手动提交又可以划分为[同步提交、异步提交、同异步联合]()提交。这些提交方式仅仅是 doWork()方法不相同，其构造器是相同的。所以下面首先在前面消费者类的基础上进行构造 器的修改，然后再分别实现三种不同的提交方式。

### 创建消费者组

![img](https://pic3.zhimg.com/80/v2-27ed316eb692a347dbcabacf09779d96_720w.jpg)

#### 消费者群体

- 消费者可以使用相同的 group.id 加入群组
- 一个组的最大并行度是组中的消费者数量←不是分区。
- Kafka将主题的分区分配给组中的使用者，以便每个分区仅由组中的一个使用者使用。
- Kafka保证消息只能被组中的一个消费者读取。
- 消费者可以按照消息存储在日志中的顺序查看消息。

#### 重新平衡消费者

添加更多进程/线程将导致Kafka重新平衡。 如果任何消费者或代理无法向ZooKeeper发送心跳，则可以通过Kafka集群重新配置。 在此重新平衡期间，Kafka将分配可用分区到可用线程，可能将分区移动到另一个进程。

#### 代码

```java
import kafka.utils.ShutdownableThread;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Collections;
import java.util.Properties;

public class SomeConsumer extends ShutdownableThread {
    private KafkaConsumer<Integer, String> consumer;

    public SomeConsumer() {
        // 两个参数：
        // 1)指定当前消费者名称
        // 2)指定消费过程是否会被中断; fasle表示不会被中断
        super("KafkaConsumerTest", false);

        Properties properties = new Properties();
        String brokers = "kafkaOS1:9092,kafkaOS2:9092,kafkaOS3:9092";
        // 指定kafka集群
        properties.put("bootstrap.servers", brokers);
        // 指定消费者组ID
        properties.put("group.id", "cityGroup1");
        // 开启自动提交，默认为true
        properties.put("enable.auto.commit", "true");
        // 指定自动提交的超时时限，默认5s
        properties.put("auto.commit.interval.ms", "1000");
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
        // 订阅消费主题topic
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
        }
    }
}

```



#### 验证

- 创建1个消费者，topic=cities, group.id = cityGroup1

```java
public class ConsumerTest {
    public static void main(String[] args) {
        SomeConsumer consumer = new SomeConsumer();
        consumer.start();
    }
}
```

| 线程  | 启动状况   | 消费状态                                                     |
| ----- | ---------- | ------------------------------------------------------------ |
| 线程1 | 第一次启动 | 从头开始消费，历史数据也能获取                               |
| 线程1 | 第二次启动 | 从第一次启动获取的最后一条历史数据为起始，开始获取历史数据；<br />如果历史数据已经消费完，那么就开始等待新数据 |

<font color="red">原理其实很简单，因为kafka集群会记录group.id的offset，所以如果换一个新的group.id，那么又从头开始消费</font>

- 创建2个消费者（不同线程），属于同一个消费者组消费者同步手动提交



### 手动同步提交

```JAVA
properties.put("enable.auto.commit", "false");  
consumer.commitSync();
```



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



## 验证

```java
import java.io.IOException;

public class OneProducerTest {

    public static void main(String[] args) throws IOException {
        OneProducer producer = new OneProducer();
        producer.sendMsg();
        System.in.read();
    }
}
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

