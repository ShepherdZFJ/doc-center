# kafka：基础篇

### 1.kafka是什么

**Kafka传统定义**:Kafka是一个分布式的基于发布/订阅模式的消息队列(Message Queue)，主要应用于大数据实时处理领域。**发布/订阅**:消息的发布者不会将消息直接发送给特定的订阅者，而是将发布的消息 分为不同的类别，订阅者只接收感兴趣的消息。

**Kafka**最新定义:Kafka是一个开源的分布式事件流平台(Event Streaming Platform)，被数千家公司用于高性能数据管道、流分析、数据集成和关键任务应用。

消息队列的应用场景无外乎是：**削峰填谷、应用解耦、异步处理**等等，具体使用案例我们在之前讲[rabbitmq基础篇](http://blog.shepherd126.top/archives/2022-04-06-18-13-02)已经详述过，这里不在做讲述，这里说一下消息队列的两种模型：

- **点对点模型**：也叫消息队列模型。如果拿上面那个“民间版”的定义来说，那么系统 A 发送的消息只能被系统 B 接收，其他任何系统都不能读取 A 发送的消息。日常生活的例子比如电话客服就属于这种模型：同一个客户呼入电话只能被一位客服人员处理，第二个客服人员不能为该客户服务。
- **发布 / 订阅模型**：与上面不同的是，它有一个主题（Topic）的概念，你可以理解成逻辑语义相近的消息容器。该模型也有发送方和接收方，只不过提法不同。发送方也称为发布者（Publisher），接收方称为订阅者（Subscriber）。和点对点模型不同的是，这个模型可能存在多个发布者向相同的主题发送消息，而订阅者也可能存在多个，它们都能接收到相同主题的消息。生活中的报纸订阅就是一种典型的发布 / 订阅模型。

### 2.kafka基础架构和核心概念

![](https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fimg2018.cnblogs.com%2Fblog%2F1726840%2F201907%2F1726840-20190717152526528-1830238135.png&refer=http%3A%2F%2Fimg2018.cnblogs.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=auto?sec=1652420642&t=2b7d1fd183538cbfb4a5ee8091784eaa)

在 Kafka 中，发布订阅的对象是**主题（Topic**），你可以为每个业务、每个应用甚至是每类数据都创建专属的主题。

**生产者（Producer）**：消息生产者，就是向 kafka broker 发消息的客户端，生产者程序通常持续不断地向一个或多个主题发送消息。

**消费者（Consumer）**：消息消费者，向 kafka broker 取消息的客户端，消费者就是订阅这些主题消息的客户端应用程序。

和生产者类似，消费者也能够同时订阅多个主题的消息。我们把生产者和消费者统称为客户端（Clients）。你可以同时运行多个生产者和消费者实例，这些实例会不断地向 Kafka 集群中的多个主题生产和消费消息。

**消费者组Consumer Group (CG)**：由多个 consumer 组成。消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费，消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。

**Broker** ：一台 kafka 服务器就是一个 broker。一个集群由多个 broker 组成。一个 broker 可以容纳多个 topic。

**主题（topic）** ：可以理解为一个队列，生产者和消费者面向的都是一个 topic;

**分区（Partition）**：为了实现扩展性，一个非常大的 topic 可以分布到多个 broker(即服务器)上， 一个 topic 可以分为多个 partition，每个 partition 是一个有序的队列; 

**副本（Replica）**：副本，为保证集群中的某个节点发生故障时，该节点上的 partition 数据不丢失，且 kafka 仍然能够继续工作，kafka 提供了副本机制，一个 topic 的每个分区都有若干个副本， 一个 **leader** 和若干个 **follower**。

 **leader**：每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对 象都是 leader。

**follower**：每个分区多个副本中的“从”，实时从 leader 中同步数据，保持和 leader 数据 的同步。leader 发生故障时，某个 follower 会成为新的 follower

**副本的工作机制也很简单：生产者总是向领导者副本写消息；而消费者总是从领导者副本读消息。至于追随者副本，它只做一件事：向领导者副本发送请求，请求领导者把最新生产的消息发给它，这样它能保持与领导者的同步。**

Kafka 使用消息日志（Log）来保存数据，一个日志就是磁盘上一个**只能追加写（Append-only）消息**的物理文件。因为只能追加写入，故**避免了缓慢的随机 I/O 操作，改为性能较好的顺序 I/O 写操作，这也是实现 Kafka 高吞吐量特性的一个重要手段**。不过如果你不停地向一个日志写入消息，最终也会耗尽所有的磁盘空间，因此 Kafka 必然要定期地删除消息以回收磁盘。怎么删除呢？简单来说就是通过日志段（Log Segment）机制。在 Kafka 底层，一个日志又近一步细分成多个日志段，消息被追加写到当前最新的日志段中，当写满了一个日志段后，Kafka 会自动切分出一个新的日志段，并将老的日志段封存起来。Kafka 在后台还有定时任务会定期地检查老的日志段是否能够被删除，从而实现回收磁盘空间的目的

### 3.kafka的命令操作

在我们部署好kafka环境之后，我可以查看kafka的bin目录的文件，提供许多操作执行脚本，包括服务端、生产者、消费者、主题等等。

```shell
[root@shepherd-master bin]# ls
connect-distributed.sh        kafka-consumer-offset-checker.sh     kafka-replica-verification.sh       kafka-verifiable-producer.sh
connect-standalone.sh         kafka-consumer-perf-test.sh          kafka-run-class.sh                  windows
kafka-acls.sh                 kafka-delete-records.sh              kafka-server-start.sh               zookeeper-security-migration.sh
kafka-broker-api-versions.sh  kafka-mirror-maker.sh                kafka-server-stop.sh                zookeeper-server-start.sh
kafka-configs.sh              kafka-preferred-replica-election.sh  kafka-simple-consumer-shell.sh      zookeeper-server-stop.sh
kafka-console-consumer.sh     kafka-producer-perf-test.sh          kafka-streams-application-reset.sh  zookeeper-shell.sh
kafka-console-producer.sh     kafka-reassign-partitions.sh         kafka-topics.sh
kafka-consumer-groups.sh      kafka-replay-log-producer.sh         kafka-verifiable-consumer.sh

```

**主题命令操作**

```shell
[root@shepherd-master bin]# kafka-topics.sh
```

对应参数如下：

| 参数                                             | 描述                                                         |
| ------------------------------------------------ | :----------------------------------------------------------- |
| --bootstrap-server <String: server toconnect to> | 连接的 Kafka Broker 主机名称和端口号。 操作的 topic 名称。这个参数新版kafka中添加的，从v2.8版本开始，Kafka可以不再依赖ZooKeeper，以前老版是使用zookeeper地址进行连接 |
| --topic <String: topic>                          | 操作的 topic 名称                                            |
| --create                                         | 创建主题                                                     |
| --delete                                         | 删除主题                                                     |
| --alter                                          | 修改主题                                                     |
| --list                                           | 查看所有主题                                                 |
| --describe                                       | 查看主题详细描述                                             |
| --partitions <Integer: # of partitions>          | 设置主题分区数                                               |
| -replication-factor<Integer: replication factor> | 设置主题分区副本数                                           |
| --config <String: name=value>                    | 更新主题默认的系统配置，比如该主题的最大保存时长，一条消息最大大小等 |

示例：

```shell
# 查看主题列表
bin/kafka-topics.sh --zookeeper 10.10.0.18:2181 --list

# 创建一个名为zfj-topic的主题，3个分区，1个副本
bin/kafka-topics.sh --zookeeper 10.10.0.18:2181 --create --replication-factor 1 --partitions 3 --topic zfj-topic

# 生产者，往主题发送消息
bin/kafka-console-producer.sh --broker-list 10.10.0.18:9092 --topic zfj-topic

# 消费者 消费主题列表的消息
bin/kafka-console-consumer.sh --bootstrap-server 10.10.0.18:9092 --topic zfj-topic

# 查看该主题的详细信息
bin/kafka-topics.sh --zookeeper 10.10.0.18:2181 --describe --topic zfj-topic
```

上面包含了对生产者、消费者的命令操作，所以这里不在做单独陈述

### 4.项目整合kafka

springboot项目整合kafka相当简单，引入依赖：

```xml
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>
```

**生产者**：发送消息

```java
package com.shepherd.kafka.producer;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;

/**
 * @author fjzheng
 * @version 1.0
 * @date 2022/1/24 14:48
 */

public class CustomProducer {
    public static void main(String[] args) {
        Properties props = new Properties();
        // kafka 集群，broker-list
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "10.10.0.18:9092");
        props.put("acks", "all");
        // 重试次数
        props.put("retries", 1);
        // 批次大小
        props.put("batch.size", 16384);
        // 等待时间
        props.put("linger.ms", 1);
        // RecordAccumulator 缓冲区大小
        props.put("buffer.memory", 33554432);
        // 序列化
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, "com.shepherd.kafka.producer.MyPartitioner");
        // 创建消息生产者
        Producer<String, String> producer = new KafkaProducer<>(props);
        // 发送消息
        /**
         * {@link ProducerRecord的构建函数
         *     public ProducerRecord(String topic, Integer partition, K key, V value) {
         *         this(topic, partition, (Long)null, key, value, (Iterable)null);
         *     }
         *     public ProducerRecord(String topic, K key, V value) {
         *         this(topic, (Integer)null, (Long)null, key, value, (Iterable)null);
         *     }
         *     public ProducerRecord(String topic, V value) {
         *         this(topic, (Integer)null, (Long)null, (Object)null, value, (Iterable)null);
         *     }
         *
         * 消息分区策略
         * 1、支持指定分区；
         * 2、传入key，然后hash，再对分区数取余
         * 3、随机选择分区，下次选择的分区和上次的为不同分区。
         * }
         */
        for (int i = 0; i < 10; i++) {
            // 指定分区
         //   producer.send(new ProducerRecord<>("topic1", 1, Integer.toString(i), "val" + Integer.toString(i)));
            // 通过传入key计算分区
            producer.send(new ProducerRecord<String, String>("topic1",
                    Integer.toString(i), "val"+Integer.toString(i)));
        }
            producer.close();
    }

}
```

通过上面代码我们可知消息发送往主题的哪个分区，有不同策略。同时我也必须知道kafka的消息只有在同一分区才能保证消费严格有序的，所以有时候我们必须吧某一批消息发往同一个分区，这时候我们可以自定义分区策略，只需要实现`Partitioner`接口即可

```java
package com.shepherd.kafka.producer;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;

import java.util.Map;

/**
 * @author fjzheng
 * @version 1.0
 * @date 2022/4/11 11:58
 */

/**
 * 1、实现{@link Partitioner}接口
 * 2、重写partition()方法
 */
public class MyPartitioner implements Partitioner {
    /**
     *
     * @param s 主题topic
     * @param o key
     * @param bytes key序列化后字节数组
     * @param o1 value
     * @param bytes1 value序列化后的字节数组
     * @param cluster 集群元数据信息可以查看分区
     * @return
     */
    @Override
    public int partition(String s, Object o, byte[] bytes, Object o1, byte[] bytes1, Cluster cluster) {
        // 获取消息的key，这里假设只处理key为整数的
        Integer key = Integer.valueOf(o.toString());
        // 获取topic的分区总数
        Integer count = cluster.partitionCountForTopic(s);
        return key % count;
    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> map) {

    }
}

```

然后再创建生产者的参数配置添加下面参数就可以啦

```java
props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, "com.shepherd.kafka.producer.MyPartitioner");
```

**消费者**：消费消息

```java
package com.shepherd.kafka.consumer;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.Deserializer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Properties;

/**
 * @author fjzheng
 * @version 1.0
 * @date 2022/4/11 15:05
 */
public class CustomConsumer {
    public static void main(String[] args) {
        // 创建消费者的配置对象
        Properties properties = new Properties();
        // 给消费者配置对象添加参数
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "10.10.0.18:9092");
        // 配置序列化 必须
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        // 设置消费者组
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, " test");
        // 创建消费者对象

        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<String, String>(properties);
        // 添加消费者订阅的主题topic，可以添加多个
        ArrayList<String> topics = new ArrayList<>();
        topics.add("first");
        topics.add("topic1");
        kafkaConsumer.subscribe(topics);
        // 拉取数据
        while (true) {
            // 每隔1s消费一批数据
            ConsumerRecords<String, String> consumerRecords = kafkaConsumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> consumerRecord : consumerRecords) {
                // 打印数据
                System.out.println(consumerRecord);
            }
        }
    }
}

```

当然以上操作都是自己手写实现kafka生产者、消费者客户端，我们大可不别这样，因为spring给我们封装好了，我们只需要简单配置，再结合注解便可优雅使用kafka

配置如下：

```properties
server.port=1008
###########【Kafka集群】###########
spring.kafka.bootstrap-servers=10.10.0.18:9092
###########【初始化生产者配置】###########
# 重试次数
spring.kafka.producer.retries=0
# 应答级别:多少个分区副本备份完成时向生产者发送ack确认(可选0、1、all/-1)
spring.kafka.producer.acks=1
# 批量大小
spring.kafka.producer.batch-size=16384
# 提交延时
spring.kafka.producer.properties.linger.ms=0
# 当生产端积累的消息达到batch-size或接收到消息linger.ms后,生产者就会将消息提交给kafka
# linger.ms为0表示每接收到一条消息就提交给kafka,这时候batch-size其实就没用了

# 生产端缓冲区大小
spring.kafka.producer.buffer-memory = 33554432
# Kafka提供的序列化和反序列化类
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
# 自定义分区器
# spring.kafka.producer.properties.partitioner.class=com.felix.kafka.producer.CustomizePartitioner

###########【初始化消费者配置】###########
# 默认的消费组ID
spring.kafka.consumer.properties.group.id=defaultConsumerGroup
# 是否自动提交offset
spring.kafka.consumer.enable-auto-commit=true
# 提交offset延时(接收到消息后多久提交offset)
spring.kafka.consumer.auto.commit.interval.ms=1000
# 当kafka中没有初始offset或offset超出范围时将自动重置offset
# earliest:重置为分区中最小的offset;
# latest:重置为分区中最新的offset(消费分区中新产生的数据);
# none:只要有一个分区不存在已提交的offset,就抛出异常;
spring.kafka.consumer.auto-offset-reset=latest
# 消费会话超时时间(超过这个时间consumer没有发送心跳,就会触发rebalance操作)
spring.kafka.consumer.properties.session.timeout.ms=120000
# 消费请求超时时间
spring.kafka.consumer.properties.request.timeout.ms=180000
# Kafka提供的序列化和反序列化类
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
# 消费端监听的topic不存在时，项目启动会报错(关掉)
spring.kafka.listener.missing-topics-fatal=false
# 设置批量消费
# spring.kafka.listener.type=batch
# 批量消费每次最多消费多少条消息
# spring.kafka.consumer.max-poll-records=50

# 设置消息的自定义分区策略
spring.kafka.producer.properties.partitioner.class=com.shepherd.kafka.partition.CustomizePartitioner
```

生产者：

```java
package com.shepherd.kafka.producer;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.util.concurrent.ListenableFutureCallback;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author fjzheng
 * @version 1.0
 * @date 2022/1/24 18:10
 */
@RestController
@RequestMapping("/api/kafka/produce")
public class ProducerController {
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    /**
     *  Kafka Producer 是异步发送消息的，也就是说如果你调用的是 producer.send(msg) 这个 API，那么它通常会立即返回，
     *  所以成功与否不确定，不带回调的发送消息是不能保证消息成功发送的，最终可能导致消息丢失。
     * @param message
     */
    @GetMapping("/{message}")
    public void sendMessageNoCallback(@PathVariable("message") String message) {
        kafkaTemplate.send("topic1", message);
    }

    /**
     *
     * @param message
     */
    @GetMapping("/callback1/{message}")
    public void sendMessage2(@PathVariable("message") String message) {
        kafkaTemplate.send("topic1", message).addCallback(success -> {
            // 消息发送到的topic
            String topic = success.getRecordMetadata().topic();
            // 消息发送到的分区
            int partition = success.getRecordMetadata().partition();
            // 消息在分区内的offset
            long offset = success.getRecordMetadata().offset();
            System.out.println("发送消息成功:" + topic + "-" + partition + "-" + offset);
        }, failure -> {
            System.out.println("发送消息失败:" + failure.getMessage());
        });
    }

    @GetMapping("/callback2/{message}")
    public void sendMessage3(@PathVariable("message") String message) {
        kafkaTemplate.send("topic1", message).addCallback(new ListenableFutureCallback<SendResult<String, Object>>() {
            @Override
            public void onFailure(Throwable ex) {
                System.out.println("发送消息失败："+ex.getMessage());
            }

            @Override
            public void onSuccess(SendResult<String, Object> result) {
                System.out.println("发送消息成功：" + result.getRecordMetadata().topic() + "-"
                        + result.getRecordMetadata().partition() + "-" + result.getRecordMetadata().offset());
            }
        });
    }
}
```

消费者：

```java
@Component
public class KafkaConsumerConfig {
    // 消费监听
    @KafkaListener(topics = {"topic1"})
    public void onMessage1(ConsumerRecord<?, ?> record){
        // 消费的哪个topic、partition的消息,打印出消息内容
        System.out.println("简单消费："+record.topic()+"-"+record.partition()+"-"+record.value());
    }
}
```

