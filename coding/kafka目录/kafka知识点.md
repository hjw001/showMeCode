# kafka

    定义：Kafka是一个分布式消息中间件，支持分区，多副本，多订阅者，基于zookeeper协调的消息系统
    特点：
    ● 高吞吐、低延迟：kafka每秒可以处理几十万条消息，它的延迟最低只有几毫秒，每个topic可以分多个partition, 由多个consumer group 对partition进行consume操作。
    ● 可扩展性：kafka集群支持热扩展
    ● 持久性、可靠性：消息被持久化到本地磁盘，并且支持数据备份防止数据丢失
    ● 容错性：允许集群中节点失败（若副本数量为n,则允许n-1个节点失败）
    ● 高并发：支持数千个客户端同时读写

    参考：https://www.yuque.com/books/share/73142574-681b-424b-996e-f34f658d4cf8/tag74g


![图 2](../../../images/33d74a01b393ab3f7b923f538e9d240dae8220f72c30e95db06d7f73ff962e70.png)  


## kafka 读写流程


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

![图 4](../../../images/93e33558c193a67d59ab25da1db4e0f173caa73463c31c411fafc90a51617af6.png)  


## Cousumer
### Consumer与topic关系

    每个group中可以有多个consumer，每个consumer属于一个consumer group，通常情况下，一个group中会包含多个consumer，这样不仅可以提高topic中消息的并发消费能力，而且还能提高"故障容错"性，如果group中的某个consumer失效那么其消费的partitions将会有其他consumer自动接管。

    对于Topic中的一条特定的消息，只会被订阅此Topic的每个group中的其中一个consumer消费，此消息不会发送给一个group的多个consumer,不过一个consumer可以同时消费多个partitions中的消息。

### consumer负载均衡

![图 3](../../../images/4cba8f226e9e406146048fc3d373fc2ef77170c5da695a3365a4029fab07ef3d.png)  

    当一个group中,有consumer加入或者离开时,会触发partitions均衡.均衡的最终目的,是提升topic的并发消费能力，步骤如下：
    假如topic1,具有如下partitions: P0,P1,P2,P3 ;假如group中,有如下consumer: C1,C2
    首先根据partition索引号对partitions排序: P0,P1,P2,P3 ; 根据consumer.id排序: C0,C1
    计算倍数: M = [P0,P1,P2,P3].size / [C0,C1].size,本例值M=2(向上取整)
    然后依次分配partitions: C0 = [P0,P1],C1=[P2,P3],即Ci = [P(i * M),P((i + 1) * M -1)]

## Kafka存储机制

    在Kafka文件存储中，一个topic中不同partition 对应着自己的文件名，每个partition的文件被分成多个大小相同的segment，每个partition仅仅需要支持顺序读写即可，segment的文件生命周期由服务器参数决定

    segment file组成：index file 与 data file 成对出现，后缀".index"和“.log”分别表示为segment索引文件、数据文件
    索引包含两个部分（均为4个字节的数字），分别为相对offset和position。



## Kafka为什么查询速度快

    4.3.1 分段

    Kafka解决查询效率的手段之一是将数据文件分片，数据文件以该段中最小的offset命名。这样在查找指定offset的Message的时候，用二分查找就可以定位到该Message在哪个段(segment)中。
    4.3.2稀疏索引

    为了进一步提高查找的效率，Kafka为每个分段后的数据文件建立了索引文件，文件名与数据文件的名字是一样的，只是文件扩展名为.index。

    索引包含两个部分（均为4个字节的数字），分别为相对offset和position。

    4.3.3 顺序读写
    4.3.5 批量发送
    生产者发送多个消息到同一个分区的时候，为了减少网络带来的系能开销，kafka会对消息进行批量发送
    batch.size
    通过这个参数来设置批量提交的数据大小，默认是16k,当积压的消息达到这个值的时候就会统一发送（发往同一分区的消息）

    4.3.6 数据压缩。
    Producer 端压缩、Broker 端保持、Consumer 端解压缩。

## Kafka 数据可靠性

    7.1.1 Topic分区副本角度

    Kafka 可以保证单个分区里的事件是有序的，分区可以在线（可用），也可以离线（不可用）。在众多的分区副本里面有一个副本是 Leader，其余的副本是 follower，所有的读写操作都是经过 Leader 进行的，同时 follower 会定期地去 leader 上的复制数据。当 Leader 挂了的时候，其中一个 follower 会重新成为新的 Leader。通过分区副本，引入了数据冗余，同时也提供了 Kafka 的数据可靠性。
    Kafka 的分区多副本架构是 Kafka 可靠性保证的核心，把消息写入多个副本可以使 Kafka 在发生崩溃时仍能保证消息的持久性。

    7.1.2 ACK机制

    如果我们要往 Kafka 对应的topic发送消息，我们需要通过 Producer 完成。Kafka 在 Producer 里面提供了消息确认机制。也就是说我们可以通过配置来决定消息发送到对应分区的几个副本才算消息发送成功。可以在定义 Producer 时通过 acks 参数指定（在 0.8.2.X 版本之前是通过 request.required.acks 参数设置的）。这个参数支持以下三种值：

    ● acks = 0：意味着如果生产者能够通过网络把消息发送出去，那么就认为消息已成功写入 Kafka 。在这种情况下还是有可能发生错误，比如发送的对象无能被序列化或者网卡发生故障，但如果是分区离线或整个集群长时间不可用，那就不会收到任何错误。在 acks=0 模式下的运行速度是非常快的，你可以得到惊人的吞吐量和带宽利用率，不过如果选择了这种模式， 一定会丢失一些消息。
    ● acks = 1：意味若 Leader 在收到消息并把它写入到分区数据文件（不一定同步到磁盘上）时会返回确认或错误响应。在这个模式下，如果发生正常的 Leader 选举，生产者会在选举时收到一个 LeaderNotAvailableException 异常，如果生产者能恰当地处理这个错误，它会重试发送悄息，最终消息会安全到达新的 Leader 那里。不过在这个模式下仍然有可能丢失数据，比如消息已经成功写入 Leader，但在消息被复制到 follower 副本之前 Leader发生崩溃。
    ● acks = all（这个和 request.required.acks = -1 含义一样）：意味着 Leader 在返回确认或错误响应之前，会等待所有同步副本都收到悄息。如果和 min.insync.replicas 参数结合起来，就可以决定在返回确认前至少有多少个副本能够收到悄息，生产者会一直重试直到消息被成功提交。不过这也是最慢的做法，因为生产者在继续发送其他消息之前需要等待所有副本都收到当前的消息。
    根据实际的应用场景，我们设置不同的 acks，以此保证数据的可靠性。
    另外，Producer 发送消息还可以选择同步（默认，通过 producer.type=sync 配置） 或者异步（producer.type=async）模式。如果设置成异步，虽然会极大的提高消息发送的性能，但是这样会增加丢失数据的风险。如果需要确保消息的可靠性，必须将 producer.type 设置为 sync。
    
    7.3 消息丢失与重复

    7.3.1 数据丢失
    提交了偏移量，但是消费的时候发生了异常

    7.3.2 数据重复
    数据消费正常，但是提交偏移量失败



