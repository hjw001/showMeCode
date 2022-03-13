### 2.maxwell bootstrap的同时，mysql在变化，怎么保证写到hbase的数据是正确的？
    参考cdc，加mysql 全局读锁，该锁将阻止其他数据库的写入。然后，它读取当前binlog位置以及数据库和表的schema。之后，将释放 全局读取锁。
    全局读取锁 在读取binlog位置和schema期间保持。这可能需要几秒钟，具体取决于表的数量。全局读取锁定会阻止写入，因此它仍然可能影响在线业务。如果要跳过读取锁，并且可以容忍至少一次语义，则可以添加'debezium.snapshot.locking.mode' = 'none'选项以跳过锁。

### 1.interval join不上的数据，怎么处理？怎么做数据修复？

    如果是延迟导致，设置延迟时间
    sql：可以通过设置 table.exec.emit.late-fire.enabled 和 table.exec.emit.late-fire.delay 来触发晚于 watermark 到达的数据。 其中允许等待晚与 watermark 的数据的时间由 table.exec.state.ttl 控制，等价于 Datastream 中的 allowedLateness， 故 window 的最大等待时间为 watermark 的 outOfOrder + allowedLateness。

    stream api 采用 cogroup 进行 full join 操作

### 重启策略

    每隔三秒重启一次，重启三次
    streamEnv.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, Time.seconds(10)))

### 分流

    分流是指旁路输出，通过process 将流分流
    ctx.output(new OutputTag<OrderDetail>("deliveryTag") {}, value); //分流
    output.getSideOutput(new OutputTag<OrderDetail>("deliveryTag"){}); //获取旁路输出流

### 数据倾斜与反压处理

    1、source消费不均匀导致反压
    增加并行度
    2、数据倾斜导致反压
    采用两阶段聚合的方式，通过添加随机前缀，打散 key 的分布，使得数据不会集中在几个 Subtask
    3、并发导致反压
    排查是否是 regular join导致并发，若无问题，无问题则添加资源 提高并行度

### Flink 内存管理

#### TaskManager 内存管理

    仅适用于flink 1.10以后版本
    内存划分：Flink 应用使用的内存（Flink 总内存）以及由运行 Flink 的 JVM 使用的内存。 其中，*Flink 总内存（Total Flink Memory）*包括 JVM 堆内存（Heap Memory）、*托管内存（Managed Memory）*以及其他直接内存（Direct Memory）或本地内存（Native Memory）。

    托管内存：
    托管内存是由 Flink 负责分配和管理的本地（堆外）内存。 以下场景需要使用托管内存：
    流处理作业中用于 RocksDB State Backend。
    流处理和批处理作业中用于排序、哈希表及缓存中间结果。
    流处理和批处理作业中用于在 Python 进程中执行用户自定义函数。

    本地内存：
    本地内存（非直接内存）也可以被归在框架堆外内存或任务堆外内存中，在这种情况下 JVM 的直接内存限制可能会高于实际需求。

    网络内存：
    网络内存（Network Memory）同样被计算在 JVM 直接内存中。 Flink 会负责管理网络内存，保证其实际用量不会超过配置大小。 因此，调整网络内存的大小不会对其他堆外内存有实质上的影响。

![图 12](../../../images/3de31b0d7580fb0bfc65857f11001dc1c59943109243c46157496bd377e34c97.png)  

    TaskManager 内存模型

![图 13](../../../images/53c25b5e77548c73e9b2b26542e0e2cea8ea574b12b4d7ce964711bc66c0fb47.png)  

    优化：当使用 memory stateBackend 与 fs stateBackend 时 将托管内存设置为0

## FLINK 状态

    状态：定义：Flink流中许多操作都为同一个事件，但是有些操作会记住多个事件的信息比如（窗口），这些操作称为有状态的

    flink state 分为：keyed state 、operate state、broadcast state

    keyed state 只能是 keyed stream使用，keyed state