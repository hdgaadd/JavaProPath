## 1. Kafka概念

### 1.1 Kafka组成模块

> ***面试官：你先说说Kafka由什么模块组成？***

Kafka其实是一款基于**发布与订阅模式**的消息系统，如果按常理来设计，大家是不是把消息发送者的消息直接发送给消息消费者？但Kafka并不是这么设计的，Kafka消息的生产者会对消息进行分类，再发送给中间的消息服务系统，而消息消费者通过订阅某分类的消息去接受特定类型的消息。

其实这么设计的目的也是为了满足大量业务消息的接入，要是单一的消息发送和接收，那开个进程的**管道通信**就可以了。另外如果大家对设计模式的**发布/订阅模式**熟悉的话，对Kafka的设计理念会更容易理解。

总的来说，Kafka由五大模块组成，大家要理解好这些模块的功能作用：消息生产者、消息消费者、`Broker`、主题`Topic`、分区`Partition`。

（1）消息生产者

消息生产者是消息的创造者，每发送一条消息都会发送到特定的主题上去。

（2）消息消费者

消息生产者和消费者都是Kafka的客户端，消息消费者顾名思义作为消息的读取者、消费者。同时Kafka很灵活的一点是，一个消费者可以订阅多个主题，而且一个主题消息也可被不同消息分组的多个消费者处理。这就给我们变化多端的业务设计带来了众多可能性了，方便大家自由发挥。

（3）`Broker`

孤零零部署在Linux的Kafka服务器被称为`Broker`，也就是我上文提到的`中间的消息服务系统`，大家不要小瞧他，单台Broker可以轻松处理**每秒百万级**的消息量。Broker日常工作内容就是接收消息生产者的消息，为每条消息设置偏移量，最后提交到磁盘进行持久化保存。

（4）主题`Topic`

上文我们知道Kafka的消息是有分类的，而分类的标识就是主题`Topic`。大家可以看下具体代码落地会更容易理解，消息生产者`Producer`发送给`clock-topic`主题，消息消费者监听消费`clock-topic`主题下的消息。

```java
// 消息生产者
public class Producer implements ApplicationRunner {
    @Resource
    private RedissonClient redissonClient;
    @Resource
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Override
    public void run(ApplicationArguments args) throws Exception {
        RBlockingQueue<Clock> blockingFairQueue = redissonClient.getBlockingQueue("delay_queue");

        while (true) {
            Clock clock = blockingFairQueue.take();
            kafkaTemplate.send("clock-topic", "key", clock.toString());
            log.info("time out: {} , clock created: {}", new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()), clock.getTime());
        }
    }
}
```

```java
    // 消息消费者
    @KafkaListener(topics = "clock-topic", groupId = "kafka-group")
    public void listener(ConsumerRecord<String, String> record, Acknowledgment ack) {
        log.info("listener get message: " + record.value());
        ack.acknowledge();
    }

```

（5）分区`Partition`

每一个主题下的消息都需要提交到Broker的磁盘里，假如我们搭建了三个Broker节点组成的Kafka集群，一般情况下同一个主题下的消息会被分到三个分区进行存储。说到这，由于顺序发送的消息是存储在不同分区中，我们无法保证消息被按**顺序消费**，只能保证**同一个分区**下的消息被顺序消费.

### 1.2 分区

> ***面试官：那分区有什么作用？***

消费分区的作用主要就是为了提高Kafka处理消息的**吞吐量**，谁叫Kafka设计之初就是作为一款高吞吐量、高可用、可扩展的应用程序。

假如一个topic下有N个分区、N个消费者，每个分区会发送消息给对应的一个消费者，这样N个消费者就可以**负载均衡**地处理消息。

同时消息生产者会发送消息给不同分区，每个分区又是属于不同的Broker，这让Broker集群**平坦压力**，大大提高了Kafka的吞吐量。

大家还需要注意一点，如果一个主题下消费者的数量超过分区的数量，超过数量的消费者是会被**闲置**的，一般N个分区最多搭配N个消费者。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/44ad717052744c3abcd1d3eae4650a27.png#pic_center)

### 1.3 异步回调

> ***面试官：消息生产者的异步回调，知道吧？***

当我们调用send()异步发送消息时，可以指定一个回调函数，该函数会等Broker服务器响应时触发。如下源码所示，我们可以为响应参数`ListenableFuture`添加一个回调函数实现`callback`。

```java
    public ListenableFuture<SendResult<K, V>> send(String topic, K key, @Nullable V data) {
        ProducerRecord<K, V> producerRecord = new ProducerRecord(topic, key, data);
        return this.doSend(producerRecord);
    }

    public interface ListenableFuture<T> extends Future<T> {
       void addCallback(ListenableFutureCallback<? super T> callback);
    }
```

那这个回调函数有什么作用？我们一般用来进行**异常日志的记录**。

Kafka的**异步提交消息**相比同步提交来说不需要在Broker响应前阻塞线程，这也在一定程度提高了消息的处理速度。但异步提交我们是不知道消息的消费情况的，此时就可以通过Kafka提供的回调函数来告知程序**异常情况**，从而方便程序进行日志记录。

## 2. 消费者消息提交

### 2.1 提交消息的方式

> ***面试官：你说说消费者手动提交和自动提交有什么区别？***

手动提交和自动提交是Kafka两种客户端的偏移量提交方式，提交方式的配置选项是`enable.auto.commit`，默认情况下该选项为ture。

偏移量提交是什么？大家可以理解为消费者通知当前最新的**读取位置**给到分区，也就是告诉分区哪些消息已消费了。

如果`enable.auto.commit`为true代表提交方式为自动提交，默认为5秒的提交时间间隔。每过**5秒**，消费者客户端就会自动提交最大偏移量。

如果`enable.auto.commit`为false代表提交方式为手动提交，我们需要让消费者客户端消费**程序执行后**提交当前的最大偏移量。

### 2.2 提交方式的优缺点

> ***面试官：那它们都有什么优、缺点？***

（1）自动提交

自动提交比较方便，我们甚至都不需要配置提交方式，不过可能会导致消息丢失或重复消费。

如果刚好到了5秒的时间间隔自动**提交了**最大偏移量，此时正在执行消息程序的消费者客户端崩溃了，就会导致**消息丢失**。

如果成功消费了消息，下一秒消费者应该自动提交，但如果此时消费者客户端奔溃，就会导致其他分区的消费者**重复消费**。

（1）手动提交

手动提交需要消费者客户端在消费消息后手动提交消息，手动提交的方式又分为同步提交、异步提交。

手动提交是**同步提交**的话，在Broker对请求做出回应之前，客户端会一直阻塞，这样的话限制应用程序的**吞吐量**。

手动提交是**异步提交**的话，不会有吞吐量的问题。不过消费者客户端发送给Broker偏移量之后，**不会管**Broker有没有收到消息。这种情况就要采用上文我提到的消息生产者**异步回调**来进行日志记录，有了日志记录方便后续bug排查，工作效率妥妥的高😏。
