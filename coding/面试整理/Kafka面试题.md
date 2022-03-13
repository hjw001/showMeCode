
### 8.3 kafka中生产数据的时候，如何保证写入的容错性？

    设置发送数据是否需要服务端的反馈,有三个值0,1,-1
    0: producer不会等待broker发送ack
    1: 当leader接收到消息之后发送ack
    -1: 当所有的follower都同步消息成功后发送ack
    request.required.acks=0

    Ack=0，相当于异步发送，消息发送完毕即 offset 增加，继续生产。 
    Ack=1，leader 收到 leader replica 对一个消息的接受 ack 才增加 offset，然后继续生产。（当数据写入leader 时，kafka刚好挂了，会导致 replica 同步失败）
    Ack=-1，leader 收到所有 replica 对一个消息的接受 ack 才增加 offset，然后继续生产。 （保证数据不丢失）
   
### 8.4如何保证kafka消费者消费数据是全局有序的

    1、kafka 单分区有序，设置分区为1 （吞吐量小），consumer 也要保证单线程消费
    2、通过自定义HashPartition 的形式，将相同的key 推入同一个分区，需要实现 Partitioner 接口
    3、消息重试导致乱序 ： 对于一个有着先后顺序的消息A、B，正常情况下应该是A先发送完成后再发送B，但是在异常情况下，在A发送失败的情况下，B发送成功，而A由于重试机制在B发送完成之后重试发送成功了。这时对于本身顺序为AB的消息顺序变成了BA。
        针对这种问题，严格的顺序消费还需要max.in.flight.requests.per.connection参数的支持。
        该参数指定了生产者在收到服务器响应之前可以发送多少个消息。它的值越高，就会占用越多的内存，同时也会提升吞吐量。把它设为1就可以保证消息是按照发送的顺序写入服务器的。
        此外，对于某些业务场景，设置max.in.flight.requests.per.connection=1会严重降低吞吐量，如果放弃使用这种同步重试机制，则可以考虑在消费端增加失败标记的记录，然后用定时任务轮询去重试这些失败的消息并做好监控报警
    4、max.in.flight.requests.per.connection需要设置小于等于5。
    原因说明：因为在kafka1.x以后，启用幂等后，kafka服务端会缓存producer发来的最近5个request的元数据，故无论如何，都可以保证最近5个request的数据都是有序的。


### 8.5 消息丢失和消息重复

    针对消息丢失:同步模式下，确认机制设置为-1，即让消息写入 Leader和 Fol lower之后再确认消息发送成功:

    exactly once: 幂等性 + 设置 ack = -1 + 事务

    幂等性：是指producer无论向broker发送了多少条重复的消息，broker只会持久化一条。在producer中开启enable.idompotence=true,此时默认就开启了acks=-1
    幂等性实现原理：
    producer连接到broker的时候会分配一个PID （producer_id）,producer发送消息到统一个分区partition的时候，消息会附带一个sequence number序列号，broker会以作为主键key进行缓存，当具有相当的
    主键key的消息进行提交的时候，Broker只会持久化一条。

    因此幂等性：指保证了单分区单回话内不重复


    不丢消息配置：
    # producer
    acks = all
    block.on.buffer.full = true
    retries = MAX_INT
    max.inflight.requests.per.connection = 1

    # consumer
    auto.offset.commit = false

    # broker
    replication.factor >= 3
    min.insync.replicas = 2
    unclean.leader.election = false


![图 16](../../../images/782deb1078c911d5e1b76fa37d55d70f18c000933a79865b573f09260bed28d8.png)  


### ISR

    Kafka采用的就是一种完全同步的方案，而完全同步方案当一个follower发生了重启将会导致所有的写入都阻塞。
    而ISR是基于完全同步的一种优化机制，Leader 持续地对 Follower 做同步性检查，如果 Follower 并不能保持同步状态，那么该 Follower 会被移出 ISR，不会再阻塞写入。当同步间隔时间超过 多少 秒时，isr会将replica 提出 ISR

### HIGH WATERMARK

    定义：小于 HW 下标的消息视为 “Committed”。消费者只能看到大于 HW 的消息。
    保障了用户读到的数据都是持久化到所有 ISR 的

![图 17](../../../images/379901e33ffc353394b66a1d0df97b957b1f3aa84cc14a95538741653b88b7a5.png)  


### Kafka 读写快

    读数据采用稀疏索引，可以快速定位要消费的数据

    Kafka 可以将数据记录分批发送，从生产者到文件系统（Kafka 主题日志）到消费者，可以端到端的查看这些批次的数据。
    批处理能够进行更有效的数据压缩并减少 I/O 延迟，Kafka 采取顺序写入磁盘的方式，避免了随机磁盘寻址的浪费。官网有数据表明，同样的磁盘，顺序写能到600M/s，而随机写只有100K/s

    零拷贝：Kafka的数据加工处理操作交由Kafka生产者和Kafka消费者处理。Kafka Broker应用层不关心存储的数据，所以就不用走应用层，传输效率高。

