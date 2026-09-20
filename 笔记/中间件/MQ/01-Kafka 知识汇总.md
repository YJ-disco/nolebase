# Kafka 知识汇总

## 简介

Kafka 被定义为一个分布式流式处理平台，它以高吞吐、可持久化、可水平扩展、支持流数据处理等多种特性而被广泛应用，在现代系统中主要承担三大角色：

- 消息系统：Kafka 和传统 MQ 一样，都具有系统解耦、冗余存储、流量削峰、异步通信、可扩展性、可恢复性等功能。与此同时，Kafka 还提供了消息顺序性保障及回溯消费等功能。
- 存储系统：Kafka把消息持久化到磁盘,相比于其他基于内存存储的系统而言,有效地降低了数据丢失的风险。也正是得益于Kafka的消息持久化功能和多副本机制,我们可以把Kafka作为长期的数据存储系统来使用,只需要把对应的数据保留策略设置为"永久"或启用主题的日志压缩功能即可。
- 流式处理平台:Kafka不仅为每个流行的流式处理框架提供了可靠的的数据来源,还提供了一个完整的流式处理类库,比如窗口、连接、变换和聚合等各类操作。

## 基本概念

Kafka 体系架构基本上由四部分组成，Producer、Broker、Consumer 以及一个 ZooKeeper 集群。

- Producer：作为消息的生产者，负责创建消息并将消息投递到 Kafka。
- Broker：作为 Kafka 的服务节点，负责接收并处理消息及请求。
- Consumer：作为 Kafka 的消费者，负责从 Broker 订阅并消费消息。
- Zookeeper：负责 Kafka 集群的元数据管理、控制器的选举等操作
自 Kafka 2.8 版本之后，Kafka 使用 raft 算法自己实现了对于元数据的管理，并逐渐淘汰对于 Zookeeper的依赖。

![](http://qiniu.yj-dis.top/image/20250615230516.png)

在 Kafka 中还有两个非常重要的概念—— 主题（topic）与分区（Partition）。在 Kafka 中，消息以主题为单位进行归类，生产者负责将消息发送到特定的 Topic ，然后消费者去订阅对应的 Topic 并进行消费。

Topic 是一个逻辑上的概念，它可以细分为多个 Partition，其中 Topic：Partition 的关系为 1：N ，很多时候我们也会把分区称作为 Topic-Partition 。同一主题下不同分区包含的消息是不同的，分区在存储层面上可以看作一个不断追加写的日志文件，消息在追加到日志文件的时候会被分配一个特定的偏移量（offset），作为本消息在当前分区的唯一标识，需要注意的是，offset 并不跨区，因此想要描述一个消息在 Kafka 中位置的话可以用 Topic-Partition-Offset 来进行标识。Kafka 通过使用 Offset 来保证消息在分区内的顺序性，**Kafka 保证的是分区有序而不是主题有序**。

![](http://qiniu.yj-dis.top/image/20250615232532.png)

每条消息在发送到 Broker 前，会根据设置的 Producer 分区规则选择存储到哪个具体的分区，在分区规则合理的情况下，所有消息可以均匀地分配到不同的分区中。

为什么要设置这么一个分区的概念呢，实际上分区带来的是 Kafka 的可扩展性，假如说我们没有 Partition 这个概念，每个 Topic 都对应一个消息文件，随着消息量的增大，这个文件所在机器的 I/O 性能将会成为这个 Topic 的性能瓶颈，假如说我们有了 Partition 的概念，将 Topic 的消息存储到多个机器上的 Paitition，我们的并发能力将会得到极大的提升。

在创建主题的时候，可以通过指定的参数来配置分区的个数，也可以在主题创建完成后再去修改分区的数量，实现水平扩展，实际环境中应通过评估在创建 Topic 的时候指定好 Partition 数目，避免因为修改 Partiton 数目而带来的 rebalance。

Kafka 为分区引入了多副本 （Replica）机制,通过增加副本数量可以是升容灾能力。同一分区的不同副本中保存的是相同的消息，副本之间是"一主多从"的关系，其中 leader 副本负责处理读写请求，follower 副本只负责与 leader 副本的消息同步。副本处于不同的 Broker 中，当 leader 副本出现故障时，从 follower 副本中重新选举新的 leader 副本对外提供服务。Kafka 通过多副本机制实现了故障的自动转移，当 Kafka 集群中某个 Broker 失效时仍然能保证服务可用。

如下图所示，Kafka集群中有 4 个 Broker，某个主题中有 3 个分区，且副本因子 （即副本个数）也为 3，如此每个分区便有 1 个 leader 副本和 2 个 follower 副本。生产者和消费者只与 leader 副本进行交互，而 follower 副本只负责消息的同步，很多时候 follower 副本中的消息相对 leader 副本而言会有一定的**滞后**。

![](http://qiniu.yj-dis.top/image/20250615234412.png)

## ISR 机制

分区中的所有副本称为 AR（Assigned Replicas）。所有与 leader 副本保持一定程度同步的副本（包括 leader）在内组成 ISR （In-Sync Replicas），ISR 集合是 AR 集合中的一个子集。消息会先发送到 leader 副本，然后 follower 副本才能从 leader 副本中拉取消息进行同步，同步期间内 follower副本相对于 leader 副本而言会有一定程度上的滞后。前面所说的"一定程度的同"是指可以忍受的滞后范围，这个范围可以通过参数进行配置。与 leader 副本同步滞后过多的副本（不包含 leader副本）组成 OSR （Out-of Sync Replicas），由此课件，AR = ISR + OSR。正常情况下，所有的 follower 副本动应该与 leader 副本保持一定程度上的同步，即 AR = ISR。OSR 集合为空。

leader 副本负责维护和跟踪 ISR 集合中的所有 follower 副本的滞后状态，当 follower 副本落后太多或者失效时，leader 副本会把它从 ISR 集合中剔除。如果 OSR 副本中有 follower 副本同步进度追上了 leader 副本，那么 leader 副本会把它从 OSR 集合中转移到 ISR集合中。**默认情况下，当 leader 发生故障时，只有 ISR 集合中的副本才有资格被选举成为新的 leader**（这个可以通过修改参数来改变）。

ISR 与 HW（High Watermark ）和 LEO （Log End Offset）之间有着紧密的联系，

HW 标识了一个特定的 offset ，它表示了在多副本中消息被同步到了哪个位置，消费者只能拉取这个 offset 之前的消息。
LEO 标识当前日志文件中下一条待写入消息的 offset

![](http://qiniu.yj-dis.top/image/20250616001051.png)

如图所示，这是一个日志文件，这个文件中有 9 条消息，第一条消息的 offset （LogStartOffset）为0，最后一条消息的 offset 为 8，offset 为 9 的消息用虚线框表示，代表下一条待写入的消息。日志文件的 HW 为 6，表示消费者只能拉取到 offset 在 0 到 5 之间的消息，而 offset 为 6 的消息对消费者而言是不可见的。

**分区 ISR 集合中的每个副本都会维护自身的 LEO，而 ISR 集合中最小的 LEO 即为分区的 HW。**

如图所示：

![](http://qiniu.yj-dis.top/image/20250616002629.png)

假设某个 partition ISR 集合存在 3 个副本，即一个 leader 副本与两个 follower 副本，此时 LEO 与 HW 都为 3.

producer 将消息3、4投递至分区 leader 副本，在消息写入 leader 后，follower 副本会发送拉取请求来拉取消息 3 和消息4。

> Follower 副本不会实时监控 leader 状态，而是周期性的向leader 发送 Fetch 请求来同步 leader 中的数据

![](http://qiniu.yj-dis.top/image/20250616091810.png)

![](http://qiniu.yj-dis.top/image/20250616091845.png)

在同步过程中，follower的同步效率有所不同，在某一时刻，follower1 完全追上 ledaer，而 follower2 只同步到了消息 3，如此，leader LEO 为 5，follower1 LEO 为5，follower2 LEO 为4，那么当前 ISR 集合的 HW=4，此时消费者只能消费到 offset 0到3之间的消息。

Kafka的复制机制并非完全同步，也非完全异步。
同步：同一条消息需要被所有的 follower 复制，才能被视为已提交，这种复制方式保证了很高的一致性，但也极大的影响了性能。
异步：follower 副本异步的从 leader 中复制数据，数值只要写入 leader 中就被认为已经成功提交，在这种情况下，如果数据刚刚写入 leader 还没来得及同步，leader 突然宕机，会有数据丢失的风险。

## 生产者

由于生产者客户端使用的语言不一致，这里只讨论生产者客户端的内部实现。

消息在真正发往 Kafka 之前，需要经历拦截器（interceptor）、序列化器（Serializer）和分区器（Partitioner）等一些列的作用，最后才执行发送操作。

![](http://qiniu.yj-dis.top/image/20250627153944.png)

整个生产者客户端主要分为两个线程，主线程与发送线程。在主线程中由 KafkaProducer 创建消息，然后通过可能的拦截器、序列化器和分区器的作用之后缓存到消息累加（RecordAccumulator，也称为消息收集器）中。Sender 线程负责从 RecordAccumulator 中获取消息并将其发送到 Kafka 中。

主线程步骤：

1. **创建 `ProducerRecord`**
    
2. **经过拦截器（Interceptor）**（可选）
    
3. **经过序列化器（Serializer）**
    
4. **经过分区器（Partitioner）**  
    决定消息被发送到哪个分区，例如：`topic-A` 的 `partition-2`
    
5. **写入缓存**  
    消息被追加到 `RecordAccumulator` 中对应分区的缓存队列中，等待 Sender 线程发送
    
RecordAccumulator 的内部为每个分区都维护了一个双端队列，队列中的内容就是 ProducerBatch，即 `Deque＜ProducerBatch＞`。消息写入缓存时，追加到双端队列的尾部；

RecordAccumulator 中的结构可以理解为：
```java
Map<TopicPartition, Deque<ProducerBatch>>
//每个 TopicPartition 就是一个 (topic, partition id) 组合，表示 Kafka 中的一个具体物理存储单元。
```

Sender 线程步骤：



> - Kafka 作为一个消息队列，涉及到磁盘 I/O 主要有两个操作：
> - Provider 向 Kakfa 发送消息，Kakfa 负责将消息以日志的方式持久化落盘；
> 
> Consumer 向 Kakfa 进行拉取消息，Kafka 负责从磁盘中读取一批日志消息，然后再通过网卡发送。
> 
> Kakfa 服务端接收 Provider 的消息并持久化的场景下使用 mmap 机制，能够基于顺序磁盘 I/O 提供高效的持久化能力，使用的 Java 类为 java.nio.MappedByteBuffer。
> 
> Kakfa 服务端向 Consumer 发送消息的场景下使用 sendfile 机制，这种机制主要两个好处：
> 
> - sendfile 避免了内核空间到用户空间的 CPU 全程负责的数据移动；
> - sendfile 基于 Page Cache 实现，因此如果有多个 Consumer 在同时消费一个主题的消息，那么由于消息一直在 page cache 中进行了缓存，因此只需一次磁盘 I/O，就可以服务于多个 Consumer。
> 
> 使用 mmap 来对接收到的数据进行持久化，使用 sendfile 从持久化介质中读取数据然后对外发送是一对常用的组合。但是注意，你无法利用 sendfile 来持久化数据，利用 mmap 来实现 CPU 全程不参与数据搬运的数据拷贝。
