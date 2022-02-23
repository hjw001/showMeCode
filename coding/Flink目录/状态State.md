# flink状态

    定义：Flink流中许多操作都为同一个事件，但是有些操作会记住多个事件的信息比如（窗口），这些操作称为有状态的
    
    文章传送门：
    https://cloud.tencent.com/developer/article/1833751

## FLink 状态分类

    主要分为：
    keyed State
    Operator State 
    Broadcast State （Operator State 一种特殊类型）

### keyed State

    keyed state 只能在KeyedStrem上使用，keyed state支持状态如下：
    ValueState<T>: 保存一个可以更新和检索的值（如上所述，每个值都对应到当前的输入数据的 key，因此算子接收到的每个 key 都可能对应一个值）。 这个值可以通过 update(T) 进行更新，通过 T value() 进行检索。

    ListState<T>: 保存一个元素的列表。可以往这个列表中追加数据，并在当前的列表上进行检索。可以通过 add(T) 或者 addAll(List<T>) 进行添加元素，通过 Iterable<T> get() 获得整个列表。还可以通过 update(List<T>) 覆盖当前的列表。

    ReducingState<T>: 保存一个单值，表示添加到状态的所有值的聚合。接口与 ListState 类似，但使用 add(T) 增加元素，会使用提供的 ReduceFunction 进行聚合。

    AggregatingState<IN, OUT>: 保留一个单值，表示添加到状态的所有值的聚合。和 ReducingState 相反的是, 聚合类型可能与 添加到状态的元素的类型不同。 接口与 ListState 类似，但使用 add(IN) 添加的元素会用指定的 AggregateFunction 进行聚合。

    MapState<UK, UV>: 维护了一个映射列表。 你可以添加键值对到状态中，也可以获得反映当前所有映射的迭代器。使用 put(UK，UV) 或者 putAll(Map<UK，UV>) 添加映射。 使用 get(UK) 检索特定 key。 使用 entries()，keys() 和 values() 分别检索映射、键和值的可迭代视图。你还可以通过 isEmpty() 来判断是否包含任何键值对。

![图 2](../../images/982580d608d3d45e3aa26ac91fdcf52f94ac45082fb21e936dca18dbc4e5f1dd.png)  

![图 3](../../images/1e270d4eb83e990158691ffcb40359f59b15d88ef965ffd26f3e06f995c17678.png)  


### Operator State

    实现 checkpointedFuction 接口来使用 Operator State
    CheckpointedFunction 接口 需要实现如下两个方法：
    // Checkpoint时会调用这个方法，我们要实现具体的snapshot逻辑，比如将哪些本地状态持久化
    void snapshotState(FunctionSnapshotContext context) throws Exception;

    // 初始化和恢复时调用
    void initializeState(FunctionInitializationContext context) throws Exception;

### Broadcast state 广播状态

    首先将一个流 设置为广播流
    // 一个 map descriptor，它描述了用于存储规则名称与规则本身的 map 存储结构
    MapStateDescriptor<String, Rule> ruleStateDescriptor = new MapStateDescriptor<>(
                "RulesBroadcastState",
                BasicTypeInfo.STRING_TYPE_INFO,
                TypeInformation.of(new TypeHint<Rule>() {}));
            
    // 广播流，广播规则并且创建 broadcast state
    BroadcastStream<Rule> ruleBroadcastStream = ruleStream
                        .broadcast(ruleStateDescriptor);

    然后将另一个非广播的流 connect 这个广播流：
        如果流是一个 keyed 流，那就是 KeyedBroadcastProcessFunction 类型；
        如果流是一个 non-keyed 流，那就是 BroadcastProcessFunction 类型。
    
    DataStream<String> output = colorPartitionedStream
                 .connect(ruleBroadcastStream)
                 .process(
                     
                     // KeyedBroadcastProcessFunction 中的类型参数表示：
                     //   1. key stream 中的 key 类型
                     //   2. 非广播流中的元素类型
                     //   3. 广播流中的元素类型
                     //   4. 结果的类型，在这里是 string
                     
                     new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {
                         // 模式匹配逻辑
                     }
                 );

## FLINK stateBackend

    定义：由 Flink 管理的 keyed state 是一种分片的键/值存储，每个 keyed state 的工作副本都保存在负责该键的 taskmanager 本地中。另外，Operator state 也保存在机器节点本地。Flink 定期获取所有状态的快照，并将这些快照复制到持久化的位置，例如分布式文件系统。

### FLINK stateBackend 分类

    https://nightlies.apache.org/flink/flink-docs-release-1.14/zh/docs/learn-flink/fault_tolerance/

    Flink 有两种 state backend 的实现 – 一种基于 RocksDB 内嵌 key/value 存储将其工作状态保存在磁盘上的，另一种基于堆的 state backend，将其工作状态保存在 Java 的堆内存中。这种基于堆的 state backend 有两种类型：FsStateBackend，将其状态快照持久化到分布式文件系统；MemoryStateBackend，它使用 JobManager 的堆保存状态快照。

    当使用基于堆的 state backend 保存状态时，访问和更新涉及在堆上读写对象。但是对于保存在 RocksDBStateBackend 中的对象，访问和更新涉及序列化和反序列化，所以会有更大的开销。但 RocksDB 的状态量仅受本地磁盘大小的限制。还要注意，只有 RocksDBStateBackend 能够进行增量快照，这对于具有大量变化缓慢状态的应用程序来说是大有裨益的。


![图 4](../../images/7129f19fa28e117d43ad9bbb3346d2b094722190947518fcec6ae393e125379f.png)  


### FLINK checkpoint

    定义：checkpoint 是一种自动自行的快照，其目的是能从故障中恢复

    二次提交定义：有两个角色收集者 与 参与责，进行需要提交事物时,
    收集者向所有参与者发送询问并等待响应，只有当所有的参与者都返回yes，收集者才向参与者发送commit 命令，参与者commit以后回复收集者

    checkpoint详细流程：
    1. 当开始做checkpoint时，JobManager就在数据流中打入一个屏障（barrier），作为检查点的界限。屏障随着算子链向下游传递，每到达一个算子都会触发checkpoint，将状态快照写入stateBackEnd，同时barrier传递到下一个算子中
    2. 下游的算子收到barrier之后开始执行snapshotState()方法
    3. 每个subtask在Checkpoint完成之后会向JobManager发送ACK，通知当前的subtask完成快照
    4. JobManager在所有subtask完成Checkpoint之后，会向所有subtask推送一个NotifyCheckpointComplete消息，表明checkpoint成功

    二次提交（TwoPhaseCommitSinkFunction）：
    TwoPhaseCommitSinkFunction 实现了RichSinkFunction，CheckpointedFunction，CheckpointListener

    beginTransaction()：开始一个事务，返回事务信息的句柄。
    preCommit()：预提交（即提交请求）阶段的逻辑。
    commit()：正式提交阶段的逻辑。
    abort()：取消事务。

    TwoPhaseCommit详细流程：
    1. CheckpointCoordinator,检查点协调器，负责协调flink算子的state分布式快照，在触发快照的时候，CheckpointCoordinator会向算子中注入Barrier消息。每当需要做checkpoint时，JobManager就在数据流中打入一个屏障（barrier），作为检查点的界限。
    2. 收到barrier之后开始执行snapshotState()方法
    3. 在进行checkpoint的同时，让下游sink开始 snapshotState()方法中调用 preCommit，没真正提交事物
    4. 然后判断是否结束，如果没有结束则调用 beginTransaction，开启下一个事务 （也是在 sink 的snapshotState 中）
    5. 完成Checkpoint后，所有的Subtask都会收到JobManager发来的NotifyCheckpointComplete消息， 在该方法中调用了最后真正的commit()方法完成事务型的提交，完成两阶段提交的过程

![图 5](../../images/6a09be23f4394994bf7b63b64882e3a1d8de9dd0ef27a43d7e877cd60ab967ff.png)  
