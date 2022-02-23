# Kafka 总结

### kafka 消息队列，具备高吞吐、低延迟、高伸缩性、持久性、可靠性

    高吞吐、低延迟：kakfa 最大的特点就是收发消息非常快，kafka 每秒可以处理几十万条消息，它的最低延迟只有几毫秒。
    高伸缩性： 每个主题(topic) 包含多个分区(partition)，主题中的分区可以分布在不同的主机(broker)中。
    持久性、可靠性： Kafka 能够允许数据的持久化存储，消息被持久化到磁盘，并支持数据备份防止数据丢失，Kafka 底层的数据存储是基于 Zookeeper 存储的，Zookeeper 我们知道它的数据能够持久存储。
    容错性： 允许集群中的节点失败，某个节点宕机，Kafka 集群能够正常工作
    高并发： 支持数千个客户端同时读写



### Kafka 体系以及角色及其作用

- Producer:生产者，发送消息的一方，将其投递到Kafka中
- Consumer:消费者,连接到Kafka 上并接受不了消息
- Broker:服务代理节点， broker 可以看作 一台Kafka 服务器，一个或多个 broker 组成了 Kafka 集群
- Kafka 两个重要概念：主题（topic） 与 分区（Partition）
    
    
    Kafka以主题为单位进行归类，生产者负责将消息发送到特定的主题，而消费者负责订阅主题并进行消费，一个主题下可以细分多个分区，同一个主题下的不同分区消息是不同的，
    分区在存储层面可以看作一个可追加的日志文件，消息在被追加到分区日志文件时候都会分配一个特定的偏移量（offset）。offset是消息在分区中的唯一标识，Kafka通过
    offset 保证消息在分区内的顺序性，由于offset 并不跨越分区，因此Kafka 只保证单分区有序，而非主题有序

![图 6](../../../images/aa5d472998b6073adba0ae125ca9093dc519f961bbcc40d210b7f91c45ac2cec.png)  

replica：Kafka 定义了 两类副本，leader replica、follower replica， follower 拉去 leader 的消息



### Kafka 速度块的原因

    Kafka 实现了零拷贝原理来快速移动数据，避免了内核之间的切换。Kafka 可以将数据记录分批发送，从生产者到文件系统（Kafka 主题日志）到消费者，可以端到端的查看这些批次的数据。
    批处理能够进行更有效的数据压缩并减少 I/O 延迟，Kafka 采取顺序写入磁盘的方式，避免了随机磁盘寻址的浪费


#### kafka 读写流程


    kafka的写入流程：
    
    1）producer先从zookeeper的 "/brokers/.../state"节点找到该partition的leader
    2）producer将消息发送给该leader
    3）leader将消息写入本地log
    4）followers从leader pull消息，写入本地log后向leader发送ACK
    5）leader收到所有ISR中的replication的ACK后，增加HW（high watermark，最后commit 的offset）并向producer发送ACK
    
    无论消息是否被消费，kafka都会保留所有消息。有两种策略可以删除旧数据：
    1）基于时间：log.retention.hours=168
    2）基于大小：log.retention.bytes=1073741824
    需要注意的是，因为Kafka读取特定消息的时间复杂度为O(1)，即与文件大小无关，所以这里删除过期文件与提高 Kafka 性能无关。
    
    kafka的 读取方式：以组为单位  同一个组消费过的数据将不会再被消费
    消费方式
    consumer采用pull（拉）模式从broker中读取数据。
    push（推）模式很难适应消费速率不同的消费者，因为消息发送速率是由broker决定的。它的目标是尽可能以最快速度传递消息，但是这样很容易造成consumer来不及处理消息，典型的表现就是拒绝服务以及网络拥塞。而pull模式则可以根据consumer的消费能力以适当的速率消费消息。
    对于Kafka而言，pull模式更合适，它可简化broker的设计，consumer可自主控制消费消息的速率，同时consumer可以自己控制消费方式——即可批量消费也可逐条消费，同时还能选择不同的提交方式从而实现不同的传输语义。

![图 7](../../../images/046a599bfe222992502bbeff9b046d6ac024b9cf7e9c220d1a41f30c05dbf450.png)  





#### kafka 分区数设置
    kakfa 每个分区下都会维护 index,log，product 吞吐量为 20m/s，customer 为 60m/s，期望吞吐量为 100m/s 时，设置分区数为5



#### kafka 数据重复与丢失处理
    Ack=0，相当于异步发送，消息发送完毕即 offset 增加，继续生产。 
    Ack=1，leader 收到 leader replica 对一个消息的接受 ack 才增加 offset，然后继续生产。（当数据写入leader 时，kafka刚好挂了，会导致 replica 同步失败）
    Ack=-1，leader 收到所有 replica 对一个消息的接受 ack 才增加 offset，然后继续生产。 （保证数据不丢失）
    
    exactly once: 幂等性 + 设置 ack = -1 + 事务
    幂等性：是指producer无论向broker发送了多少条重复的消息，broker只会持久化一条。在producer中开启enable.idompotence=true,此时默认就开启了acks=-1
    幂等性实现原理：
    producer连接到broker的时候会分配一个PID （producer_id）,producer发送消息到统一个分区partition的时候，消息会附带一个sequence number序列号，broker会以作为主键key进行缓存，当具有相当的
    主键key的消息进行提交的时候，Broker只会持久化一条。


#### kafka 消费数据挤压处理
    消费者消费速度过慢导致挤压，增加分区数
    下游数据处理不及时：提高生产者 批次拉取的数量


#### kafka 相关参数及优化
    log.retention.hours=72，数据保留三天

#### kafka 乱序问题
    1、kafka 单分区有序，设置分区为1 （吞吐量小），consumer 也要保证单线程消费
    2、通过自定义HashPartition 的形式，将相同的key 推入同一个分区，需要实现 Partitioner 接口
    3、消息重试导致乱序 ： 对于一个有着先后顺序的消息A、B，正常情况下应该是A先发送完成后再发送B，但是在异常情况下，在A发送失败的情况下，B发送成功，而A由于重试机制在B发送完成之后重试发送成功了。这时对于本身顺序为AB的消息顺序变成了BA。
        针对这种问题，严格的顺序消费还需要max.in.flight.requests.per.connection参数的支持。
        该参数指定了生产者在收到服务器响应之前可以发送多少个消息。它的值越高，就会占用越多的内存，同时也会提升吞吐量。把它设为1就可以保证消息是按照发送的顺序写入服务器的。
        此外，对于某些业务场景，设置max.in.flight.requests.per.connection=1会严重降低吞吐量，如果放弃使用这种同步重试机制，则可以考虑在消费端增加失败标记的记录，然后用定时任务轮询去重试这些失败的消息并做好监控报警
#### 