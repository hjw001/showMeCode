# FLINK 面试题


## Exactly_Once
    一 source
    支持断点续读(保存偏移量)
    如 kafkasource，可以记录偏移量，可以将偏移量保存到状态中（operatorState）

    二 checkpoint
    state
    三 sink
    3.1 支持事物(两段提交)
    3.2 幂等性(将数据覆盖，如redisSink)

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

## 反压机制

    1、背景：运行在TaskManager上Transform和Sink之间都会有ResultPartition和InputGate这俩个组件，ResultPartition用来发送数据，而InputGate用来接收数据。
    flink 读写都会使用BufferPool（缓存池）（一个TaskManager共用一个BufferPool 每个读写ResultPartition/InputGate都会去申请自己的LocalBuffer
    当下游的算子处理速度 比上游发送速度慢时，上游算子就不会再发送，
    导致问题：flink 的 消费速度变满由原来的 3000qps 变成 300qps，checkpoint 也会变慢 ，原本一次checkpoint执行只需要30~40ms，反压后一次checkpoint需要2min+。
    反压可能会导致checkpoint 失败，如果某个Task出现了反压，Barrier流动的速度就会变慢，导致Checkpoint整体时间变长，如果反压很严重，还有可能导致Checkpoint超时失败，然后会重启作业。
    Flink 不需要一种特殊的机制来处理反压，因为 Flink 中的数据传输相当于已经提供了应对反压的机制。

## Savepoint & checkpoint

    1.1 Checkpoint
        Checkpoint 是 Flink 用来从故障中恢复的机制，快照下了整个应用程序的状态，当然也包括输入源读取到的位点。程序一旦意外崩溃，Flink 将通过从 Checkpoint 加载应用程序状态并从恢复的读取位点继续应用程序的处理，将应用程序恢复到最后一次快照的状态，就像什么事情都没发生一样。重新执行，避免数据丢失重复，保证数据精准一次性，整个生命周期由flink自动管理，定期创建定期删除

        默认情况下，检查点不会被保留，需要配置
        //程序异常退出或人为cancel掉，不删除checkpoint数据(默认是会删除)
        env.getCheckpointConfig().enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

    1.2 Savepoint

        保存点在Flink中叫做Savepoint，是基于检查点机制的应用完整快照备份机制，用来保存状态，可以在其他集群或者时间点从savepoint中恢复作业，通常存储在所选的分布式文件系统或数据存储中。
        适用于应用升级，集群迁移，flink版本升级，主要关注数据的可移植性，并支持对作业做任何更改而状态能保持兼容，这使得生成和恢复的成本更高。
        需要手动备份且需要用户手动管理

