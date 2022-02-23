# flink知识点

## flink

    Flink 程序在运行时主要有 TaskManager，JobManager，Client 三种角色。 JobManager 扮演着集群中的管理者 Master 的角色，它是整个集群的协调者，负责接收 Flink Job，协调检查点，Failover 故障恢复等，同时管理 Flink 集群中从节点 TaskManager。 TaskManager 是实际负责执行计算的 Worker，在其上执行 Flink Job 的一组 Task，每个 TaskManager 负责管理其所在节点上的资源信息，如内存、磁盘、网络，在启动的时候将资 源的状态向 JobManager 汇报。 Client是 Flink程序提交的客户端，当用户提交一个 Flink程序时，会首先创建一个Client， 该 Client 首先会对用户提交的 Flink 程序进行预处理，并提交到 Flink 集群中处理，所以 Client 需要从用户提交的 Flink 程序配置中获取 JobManager 的地址，并建立到 JobManager 的连接， 将 Flink Job 提交给 JobManager。