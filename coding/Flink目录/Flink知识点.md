# flink知识点

## flink

    Flink 程序在运行时主要有 TaskManager，JobManager，Client 三种角色。 JobManager 扮演着集群中的管理者 Master 的角色，它是整个集群的协调者，负责接收 Flink Job，协调检查点，Failover 故障恢复等，同时管理 Flink 集群中从节点 TaskManager。 TaskManager 是实际负责执行计算的 Worker，在其上执行 Flink Job 的一组 Task，每个 TaskManager 负责管理其所在节点上的资源信息，如内存、磁盘、网络，在启动的时候将资 源的状态向 JobManager 汇报。 Client是 Flink程序提交的客户端，当用户提交一个 Flink程序时，会首先创建一个Client， 该 Client 首先会对用户提交的 Flink 程序进行预处理，并提交到 Flink 集群中处理，所以 Client 需要从用户提交的 Flink 程序配置中获取 JobManager 的地址，并建立到 JobManager 的连接， 将 Flink Job 提交给 JobManager。

## 运行时架构
    https://nightlies.apache.org/flink/flink-docs-release-1.14/zh/docs/concepts/flink-architecture/

    Flink 运行时由两种类型的进程组成：一个 JobManager 和一个或者多个 TaskManager。

    JobManager 作用：JobManager 具有许多与协调 Flink 应用程序的分布式执行有关的职责：它决定何时调度下一个 task（或一组 task）、对完成的 task 或执行失败做出反应、协调 checkpoint、并且协调从失败中恢复等等
    JobManager 由以下角色组成：
        ResourceManager

        ResourceManager 负责 Flink 集群中的资源提供、回收、分配 - 它管理 task slots，这是 Flink 集群中资源调度的单位（请参考TaskManagers）。Flink 为不同的环境和资源提供者（例如 YARN、Kubernetes 和 standalone 部署）实现了对应的 ResourceManager。在 standalone 设置中，ResourceManager 只能分配可用 TaskManager 的 slots，而不能自行启动新的 TaskManager。

        Dispatcher

        Dispatcher 提供了一个 REST 接口，用来提交 Flink 应用程序执行，并为每个提交的作业启动一个新的 JobMaster。它还运行 Flink WebUI 用来提供作业执行信息。

        JobMaster

        JobMaster 负责管理单个JobGraph的执行。Flink 集群中可以同时运行多个作业，每个作业都有自己的 JobMaster。


    TaskManagers 作用：执行作业流的 task，并且缓存和交换数据流


    Tasks 和 subtask 
    对于分布式执行，Flink 将算子的 subtasks 链接成 tasks。每个 task 由一个线程执行。将算子链接成 task 是个有用的优化：它减少线程间切换、缓冲的开销，并且减少延迟的同时增加整体吞吐量。
    一个算子就是一个Task,同时一个算子的并行度是几, 这个Task就有几个SubTask

    Task Slots 与资源
    一个TaskManager 是一个JVM进程，为了控制TaskManager 能够接受多少个Task，因此有了Task slot
    每个 task slot 代表 TaskManager 中资源的固定子集。例如，具有 3 个 slot 的 TaskManager，会将其托管内存 1/3 用于每个 slot

![图 9](../../../images/748f509fc56e057527238dd65ddaa89ed578dd5c8b1674d05c3bb39467494ef0.png)  

    
    运行流程：

![图 10](../../../images/a6f4992834071751a70b195259bf5986e956352bfe14e964df802e74a524e23a.png)  

    Yarn运行流程：
    1.Flink任务提交后，Client向HDFS上传Flink的Jar包和配置
    2.向Yarn ResourceManager提交任务，ResourceManager分配Container资源
    3.通知对应的NodeManager启动ApplicationMaster，ApplicationMaster启动后加载Flink的Jar包和配置构建环境，然后启动JobManager
    4.ApplicationMaster向ResourceManager申请资源启动TaskManager
    5.ResourceManager分配Container资源后，由ApplicationMaster通知资源所在节点的NodeManager启动TaskManager
    6.NodeManager加载Flink的Jar包和配置构建环境并启动TaskManager
    TaskManager启动后向JobManager发送心跳包，并等待JobManager向其分配任务

![图 11](../../../images/37466d8629407dde9e7a1e2332eb2f8239d6ad7a2b75f47a1222f8577c5cc4ea.png)  



![图 8](../../../images/9a02cbae4892d05d22e891b35e16e80788f1d15393417850d0126b8d2649d003.png)  


### flink 分区策略

    定义：算子进行transfrom 操作时，分区器决定了在实际运行中的数据流分发模式， 将数据切分交给 Task 计算，每个Task 负责计算一部分数据流 
