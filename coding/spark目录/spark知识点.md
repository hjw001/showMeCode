# spark 知识点

## spark 角色介绍及其作用

    1、Application 基于spark的用户程序，包含 Driver program 和集群中多个 executor

    2、Driver（驱动器）：
            2.1 概述：运行application的main()函数并自动创建SparkContext、创建RDD、以及 rdd 的 transform 和 action ，以及任务的调度和切分
            2.2 功用：把用户程序转成作业（JOB）,跟踪Excutor 的运行情况，为执行节点调度任务

    3、SparkContext: Spark的主要入口点，代表对计算集群的一个连接，是整个应用的上下文，负责与ClusterManager通信，进行资源申请、任务的分配和监控等。
    4、ClusterManager：在集群上获得资源的外部服务（spark standalone，mesos，yarm），Standalone模式：Spark原生的资源管理，由Master负责资源，YARN模式：Yarn中的ResourceManager
    5、Worker Node：集群中任何可运行Application代码的节点，负责控制计算节点，启动Executor或者Driver（Standalone模式：Worder，Yarn模式：NodeManager）

    6、Executor（执行器）：
            6.1 概述：executor 是一个 JVM 进程，负责 Spark作业中运行任务，任务间相互独立，Spark 应用启动时，Executor节点被同时启动，并且始终伴随着整个 Spark 应用的生命周期而存在。
            6.2 功用：负责运行组成 Spark 应用的任务，并将结果返回给驱动器进程；通过自身的块管理器（Block Manager）为用户程序中要求缓存的RDD提供内存式存储。RDD是直接缓存在Executor进程内的，因此任务可以在运行时充分利用缓存数据加速运算。
    7、RDD：弹性分布式数据集，是spark 的基本运算单元，通过scala集合转化读取数据集生成或者由其他RDD进过算子操作得到
    8、Job：可以被拆分成Task并行计算的单元，一般为Spark Action触发的一次执行作业
    9、Stage：每个Job会被拆分成很多组Task，每组任务被称为Stage，也可称TaskSet，该属于经常在日志中看到
    10、Task：被送到executor上执行的工作单元

<br/>
<br/>

## 三、spark 运行流程

    1、构建Spark Application 的运行环境(启动 SparkCOntenxt)，SparkContext 向资源管理器(Yarn、Stan) 申请运行Excutor的资源
    2、Yarn 分配给Executor资源并启动StandloneExecutorBackEnd，Executor运气情况随着心跳发送给资源管理器
    3、SparkContext 构建成DAG图，将DAG 划分成 stage，并把Taskset发送给Task Scheduler。Executor向SparkContext申请Task
    4、Task Scheduler将Task发放给Executor运行同时SparkContext将应用程序代码发放给Executor。
    5、Task在Executor上运行，运行完毕释放所有资源。

![图 4](../../images/646fee520a9af4ed83fb28a1f4197627987a6ba936b960040bee2cb2da4e1279.png)  

<br/>
<br/>

## 四、DAGSchedule 
    https://www.jianshu.com/p/bb4805847738
1、定义：高级调度层，实现面向阶段的调度

<br/>

2、如何被调用：
    
    spark 程序每一个action 算子最终提交一个Job是调用了SparkContext的runJob方法实现的，
    在该方法中通过dagSchedualer.runJob()正式向集群提交一个Job任务
    dagScheduler 将最后一个算子（finalRDD）调用 createResultStage 方法 封装成 finalStage, 调用 submitStage(finalStage)
    submitStage 在这个方法中，首先弹栈获得栈顶的RDD，并使用循环反复调用当前RDD所依赖的父RDD，并判断其父RDD是宽依赖还是窄依赖；
    如果是宽依赖，则创建一个新的stage，（stage 分为 ShuffleMapStage 和 ResultStage），并将其加入到missingStage缓存中；如果是窄依赖的话，则将当前的RDD在压入栈中；
    运行完以上动作之后，接着使用递归操作，重复调用submitStage()方法，直到没有父Stage的时候，即方法返回结果为Nil的时候，开始调用 submitMissingTask 将一个stage（即一个Taskset）提交给TaskScheduler去；

<br/>

3、作用：

    它将阶段作为taskset提交给底层的TaskScheduler实现（划分Stage）
    除了提出阶段的DAG之外，DAGScheduler还根据当前缓存状态确定运行每个任务的首选位置，并将这些位置传递给底层的TaskScheduler
    缓存跟踪:DAGScheduler计算出哪些rdd被缓存以避免重新计算它们，并且同样记住哪些shuffle map阶段已经产生了输出文件，以避免重做shuffle的map端。
    阶段(Stage)是计算作业中间结果的任务集，其中每个任务在同一个RDD的分区上计算相同的功能。阶段在shuffle边界上被分开，这引入了一个障碍(我们必须等待前一个阶段完成获取输出)。有两种类型的阶段:ResultStage(执行操作的最后阶段)和ShuffleMapStage(为shuffle编写映射输出文件)。阶段通常在多个作业之间共享，如果这些作业重用相同的rdd。

## 五、TaskSchedule 主要由 TaskScheduleImpl 实现了调度
    https://blog.51cto.com/u_9269309/2091219
1、定义:spark 的 Task 调度

<br/>

2、如何被调用：

    dagSchedule 会调用 taskSchedule.submitTasks 将分装的 taskSet 提交给 taskSchedule
    createTaskSetManager 会将 taskSet 添加到 TaskManager 然后吧任务加入到任务调度池中
    schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)   把封装好的taskSet，加入到任务调度队列中。（SchedulerBackend）backend.reviveOffers() 这个地方就是向资源管理器发出请求，请求任务的调度