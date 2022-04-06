# RabbitMQ：基础篇

#### 1.什么是MQ

MQ：消息队列(message queue)： 顾名思义，MQ本质是个队列，FIFO先入先出，只不过队列中存放的内容是message，还是一种跨进程的通信机制，用于上下游传递消息。在互联网架构中，MQ是一种非常常见的上下游“逻辑解耦+物理解耦”的消息通信服务。使用了MQ之后，消息发送上游只需要依赖MQ，不用依赖其他服务。

#### 2.MQ的使用场景

- 流量削峰

  流量削峰是秒杀系统抗住高并发请求的关键，**使用消息队列隔离网关和后端服务，以达到流量控制和保护后端服务的目的**，加入消息队列后，整个秒杀流程变为：

  1）网关在收到请求后，将请求放入请求消息队列。

  2）后端服务从请求消息队列中获取客户端请求，完成后续秒杀处理过程，然后返回结果。

  ![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97-%E6%B5%81%E9%87%8F%E5%89%8A%E5%B3%B0)

  秒杀开始后，当短时间内大量的秒杀请求到达网关时，不会直接冲击到后端的秒杀服务，而是先堆积在消息队列中，后端服务按照自己的最大处理能力，从消息队列中消费请求进行处理。对于超时的请求可以直接丢弃，APP将超时无响应的请求处理为秒杀失败即可。运维人员还可以随时增加秒杀服务的实例数量进行水平扩容，而不用对系统的其他部分做任何更改。

  这种设计的优点是：能根据下游的处理能力自动调节流量，达到“削峰填谷”的作用。但这样做同样是有代价的：

  - 增加了系统调用链环节，导致总体的响应时延变长。
  - 上下游系统都要将同步调用改为异步消息，增加了系统的复杂度

- 应用解耦

   以电商应用为例，应用中有订单系统、库存系统、物流系统、支付系统。用户创建订单后，如果耦合调用库存系统、物流系统、支付系统，任何一个子系统出了故障，都会造成下单操作异常。当转变成基于消息队列的方式后，系统间调用的问题会减少很多，比如物流系统因为发生故障，需要几分钟来修复。在这几分钟的时间里，物流系统要处理的内存被缓存在消息队列中，用户的下单操作可以正常完成。当物流系统恢复后，继续处理订单信息即可，这期间用户感受不到物流系统的故障，提升系统的可用性。

- 异步处理

  秒杀系统需要解决的核心问题是，如何利用有限的服务器资源，尽可能多地处理短时间内的海量请求。我们知道，处理一个秒杀请求包含了很多步骤，例如：

  - 风险控制；
  - 库存锁定；
  - 生成订单；
  - 短信通知；
  - 更新统计数据。

  如果没有任何优化，正常的处理流程是：App 将请求发送给网关，依次调用上述 5 个流程，然后将结果返回给 APP。

  对于对于这 5 个步骤来说，能否决定秒杀成功，实际上只有风险控制和库存锁定这 2 个步骤。只要用户的秒杀请求通过风险控制，并在服务端完成库存锁定，就可以给用户返回秒杀结果了，对于后续的生成订单、短信通知和更新统计数据等步骤，并不一定要在秒杀请求中处理完成。

  所以当服务端完成前面 2 个步骤，确定本次请求的秒杀结果后，就可以马上给用户返回响应，然后把请求的数据放入消息队列中，由消息队列异步地进行后续的操作。

  ![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97-%E5%BC%82%E6%AD%A5%E5%A4%84%E7%90%86)

处理一个秒杀请求，从 5 个步骤减少为 2 个步骤，这样不仅响应速度更快，并且在秒杀期间，我们可以把大量的服务器资源用来处理秒杀请求。秒杀结束后再把资源用于处理后面的步骤，充分利用有限的服务器资源处理更多的秒杀请求。

**可以看到，在这个场景中，消息队列被用于实现服务的异步处理。**这样做的好处是：

- 可以更快地返回结果；

- 减少等待，自然实现了步骤之间的并发，提升系统总体的性能。

  

#### 3.RabbitMQ及其核心概念

RabbitMQ是一个开源的遵循AMQP协议实现的基于Erlang语言编写，支持多种客户端（语言）。用于在分布式系统中存储消息，转发消息，具有高可用，高可扩性，易用性等特征。

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/rabbitMQ%E6%A6%82%E5%BF%B5%E5%9B%BE.jpg)

**核心概念**

**Message**：消息，消息是不具名的，它由消息头和消息体组成。消息体是不透明的，而消息头则由一系列的可选属性组成， 这些属性包括routing-key（路由键）、priority（相对于其他消息的优先权）、delivery-mode（指出该消息可 能需要持久性存储）等。

**Publisher**：消息的生产者，也是一个向交换器发布消息的客户端应用程序。

**Exchange**：交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列。

Exchange有4种类型：direct(默认)，fanout,topic,和headers，不同类型的Exchange转发消息的策略有所区别

**Queue**：消息队列，用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。一个消息可投入一个或多个队列。消息一直 在队列里面，等待消费者连接到这个队列将其取走。

**Binding**：绑定，用于消息队列和交换器之间的关联。一个绑定就是基于路由键将交换器和消息队列连接起来的路由规则，所以可以将交 换器理解成一个由绑定构成的路由表。

Exchange和Queue的绑定可以是多对多的关系。

**Connection**：网络连接，比如一个TCP连接。

**Channel**：信道，多路复用连接中的一条独立的双向数据流通道。信道是建立在真实的TCP连接内的虚拟连接，AMQP命令都是通过信道 发出去的，不管是发布消息、订阅队列还是接收消息，这些动作都是通过信道完成。因为对于操作系统来说建立和销毁TCP都 是非常昂贵的开销，所以引入了信道的概念，以复用一条TCP连接。如果每一次访问 RabbitMQ 都建立一个Connection，在消息量大的时候建立 TCP Connection的开销将是巨大的，效率也较低。Channel是在connection内部建立的逻辑连接，如果应用程序支持多线程，通常每个thread创建单独的channel进行通讯，AMQP method包含了channel id 帮助客户端和message broker 识别 channel，所以channel之间是完全隔离的。Channel作为轻量级的Connection极大减少了操作系统建立TCP connection的开销。 

**Consumer**：消息的消费者，表示一个从消息队列中取得消息的客户端应用程序。

**Virtual Host**：虚拟主机，表示一批交换器、消息队列和相关对象。出于多租户和安全因素设计的，把 AMQP 的基本组件划分到一个虚拟的分组中，类似于网络中的namespace概念。当多个不同的用户使用同一个RabbitMQ server提供的服务时，可以划分出多个vhost，每个用户在自己的 vhost 创建 exchange／queue 等 。

**Broker**：表示消息队列服务器实体。接收和分发消息的应用，RabbitMQ Server就是Message Broker 

#### 4.Exchange类型

- Direct交换器

  是一种点对点，实现发布/订阅标准的交换器。Producer发送消息到RabbitMQ中，MQ中的Direct交换器接受到消息后，会根据Routing Key来决定这个消息要发送到哪一个队列中。Consumer则负责注册一个队列监听器，来监听队列的状态，当队列状态发生变化时，消费消息。注册队列监听需要提供交换器信息，队列信息和路由键信息。

  这种交换器通常用于点对点消息传输的业务模型中。如电子邮箱。

- Fanout交换器

  不处理路由键。你只需要简单的将队列绑定到交换机上。一个发送到交换机的消息都会被转发到与该交换机绑定的所有队列上。很像子网广播，每台子网内的主机都获得了一份复制的消息。Fanout交换机转发消息是最快的。它是将接收到的所有消息**广播**到它知道的所有队列中

- Topic交换器

  主题交换器，也称为规则匹配交换器。是通过自定义的模糊匹配规则来决定消息存储在哪些队列中。当Producer发送消息到RabbitMQ中时，MQ中的交换器会根据路由键来决定消息应该发送到哪些队列中

- Header交换器

  不处理路由键。而是根据发送的消息内容中的headers属性进行匹配。在绑定Queue与Exchange时指定一组键值对；当消息发送到RabbitMQ时会取到该消息的headers与Exchange绑定时指定的键值对进行匹配；如果完全匹配则消息会路由到该队列，否则不会路由到该队列。headers属性是一个键值对，可以是Hashtable，键值对的值可以是任何类型。而fanout，direct，topic 的路由键都需要要字符串形式的。

#### 5.Springboot整合rabbitMQ

springboot整合rabbitMQ非常简单，只需要引入如下依赖：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

这样就可以引入自动配置类RabbitAutoConfiguration，在项目启动时候自动rabbitMQ相关配置。此时假如我们需要修改一些rabbitMQ的相关默认配置，就需要研究这个自动配置类代码，比如说我们想要修改mq消息的序列化规则，这时候我们根据RabbitAutoConfiguration源码可知，如果我们没有自定义MessageConvert注入 到容器中，那么默认使用时jdk序列化类。那么我们要修改的化只需要自定义MessageConvert并注入到容器中：

```java
 /**
     * 设置MQ发送消息的序列化规则
     *
     * @return
     */
    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
```

引入依赖之后，我们就可以使用RabbitTemplate， AmqpAdmin等组件创建exchange，queue，binding，发送消息等等。此时不需要在启动类上添加注解@EnableRabbit，但是如果我们需要监听队列消息，使用@RabbitListener注解，这个时候必须开启@EnableRabbit注解，这个可在@RabbitListener源码注释中可见。

使用@RabbitListener监听接收队列消息，示例如下：

```
@Component
@Slf4j
@RabbitListener(queues = {"order.release.order.queue"})
public class OrderCloseListener {

    @Autowired
    private OrderService orderService;

    @RabbitHandler
    public void listener(OrderEntity orderEntity, Message message, Channel channel) throws IOException {
        log.info("收到过期的订单信息，准备关闭订单" + orderEntity.getOrderSn());
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            orderService.closeOrder(orderEntity);
            channel.basicAck(deliveryTag,false);
        } catch (Exception e){
            channel.basicReject(deliveryTag,true);
        }

    }
}
```

这里的监听方法参数会被自动映射对应的消息信息，方法参数大概有三种类型：

1）Message message：原生消息信息， 头+体

2）Channel channel：当前传输数据的信道，可靠信息投递的确认机制需要用到信道

3）T(泛型)：消息内容的实体，这样就回自动映射为消息内容，不需要转换。

同一个队列的消息可以被很多消费者监听接收，只要消息被消费，那么队列就会删除消息，**消息只能被一个消费者消费。**

**@RabbitHandler注解监听接收同一队列不同消息内容或者RabbitListener监听多个队列的不同消息**

#### 6.RabbitMQ消息确认机制和可靠抵达

消息在发送和消费的过程都有可能出现问题，比如说在消费消息时候服务挂了，此时消息没有被正常消费，但是被队列删除了；

在消息发送给MQ服务器broker时网络抖动发送失败，再或者消息从交换机到队列中时发生错误等等。

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/rabbitMQ%E6%B6%88%E6%81%AF%E5%8F%AF%E9%9D%A0%E6%8A%95%E9%80%92%E6%9C%BA%E5%88%B6)

1）confirmCallback确认模式：此时消息发送到服务器的回调方法，示例如下：

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/7/6 22:55
 */
@Configuration
@Slf4j
public class RabbitmqConfig {

    @Resource
    private RabbitTemplate rabbitTemplate;

    @PostConstruct // 在RabbitmqConfig对象创建完之后执行
    public void initRabbitMq() {
        /**
         * 设置发送消息到mq服务器即broker，回调判断发送成功与否，这个时候需要开启配置：
         * 老版本的配置是：spring.rabbitmq.publisher-confirms=true   新版本中已经过时
         * 新版本的配置是：spring.rabbitmq.publisher-confirm-type=correlated
         */
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            /**
             *
             * @param correlationData:当前消息数据的唯一id，需要我们在发送消息时候指定
             * @param b 是否发送成功
             * @param s 失败的原因
             */
            @Override
            public void confirm(CorrelationData correlationData, boolean b, String s) {
                log.info("correlationData={}, b={}, s={}", correlationData, b, s);
            }
        });
    }
```

2）returnCallback模式：服务器把消息经交换机路由到队列，示例如下：

```java
 /**
         * 设置MQ服务器把消息从交换机路由到队列是否成功回调，这个时候需要开启配置：
         * spring.rabbitmq.publisher-returns=true
         * spring.rabbitmq.template.mandatory=true
         * 该方法只有在消息没有正常抵达队列才会触发回调该方法
         */
        rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
            /**
             *
             * @param message 失败的消息的内容
             * @param replyCode 回复的状态码
             * @param replyText 回复的文本内容
             * @param exchange 消息发送的交换机
             * @param routeKey 消息发送的路由键
             */
            @Override
            public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routeKey) {

            }
        });

```

3）消息正常消费确认机制：**ack确认机制**

消息被消费之后默认是自动确认的，即消息被监听接收到，那么消费者客户端会自动确认，然后服务端会删除消息。这种机制是有问题，比如说我们接收到消息之后，后面的逻辑没有正常处理完，或者我们接收了很多消息，但是只有一个消息被正常处理了，然后服务宕机了，此时消息全被删除了，不合理。解决办法就是ack确认机制改为手动，而不是自动确认。

ack手动确认机制依赖于chanel：

```java
@Component
@RabbitListener(queues = {"order.release.order.queue"})
public class OrderCloseListener {

    @Autowired
    private OrderService orderService;

    @RabbitHandler
    public void listener(OrderEntity orderEntity, Message message, Channel channel) throws IOException {
        System.out.println("收到过期的订单信息，准备关闭订单" + orderEntity.getOrderSn());
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            orderService.closeOrder(orderEntity);
            //手动确认消息被消费，然后被删除
            channel.basicAck(deliveryTag,false);
        } catch (Exception e){
            //发生异常，此时消息没有被正常消费，不能确认，baseRject的第一个参数设置为true即重新入队，如果设置为false就丢弃消息
            channel.basicReject(deliveryTag,true);
             //basicNack与basicReject一样，只是多了一个参数控制是否可以批量拒绝
            //channel.basicNack(deliveryTag, true, true);
        }

    }
}
```

注意：如果Consumer没有处理消息确认，将导致严重后果。如：所有的Consumer都没有正常反馈确认信息，并退出监听状态，消息则会永久保存，并处于锁定状态，直到消息被正常消费为止。消息的发送者Producer如果持续发送消息到RabbitMQ，那么消息将会堆积，持续占用RabbitMQ所在服务器的内存，导致“内存泄漏”问题。