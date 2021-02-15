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

- 导入依赖

  - ```xml
    <!-- kafka 依赖 -->
    <dependency>
      <groupId>org.apache.kafka</groupId>
      <artifactId>kafka_2.12</artifactId>
      <version>1.1.1</version>
    </dependency>
    ```

- 创建发布者 **OneProducer**

  - 创建发布者类 **OneProducer**：[支持批量发送，定时发送，支持回调]()
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

- 创建消费者类

# 使用**Spring Boot Kafka**