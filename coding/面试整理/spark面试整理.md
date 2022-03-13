# Spark 基础 及面试

<br/>
<br/>

## 一、spark 版本
    spark 2.4.3 scala 2.11.12 jdk8

## 二、spark 角色及其作用（https://www.cnblogs.com/frankdeng/p/9301485.html）
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

![图 1](../../../images/9829fbab5281015933688d2d40c2ed0e259b2f817673db186466905cc9b35f34.png)  

<br/>
<br/>

## 三、spark 运行流程

    1、构建Spark Application 的运行环境(启动 SparkCOntenxt)，SparkContext 向资源管理器(Yarn、Stan) 申请运行Excutor的资源
    2、Yarn 分配给Executor资源并启动StandloneExecutorBackEnd，Executor运气情况随着心跳发送给资源管理器
    3、SparkContext 构建成DAG图，将DAG 划分成 stage，并把Taskset发送给Task Scheduler。Executor向SparkContext申请Task
    4、Task Scheduler将Task发放给Executor运行同时SparkContext将应用程序代码发放给Executor。
    5、Task在Executor上运行，运行完毕释放所有资源。

![图 2](../../../images/fcf5b64a3c4fe3f18bc40a965befeadabad6b12f316ff870404bbb36cde6bfcd.png)  


<br/>
<br/>


## 五、spark 优化 记录

1、特定情况下使用 mapPartitions
    
    诸如打开数据库连接或创建随机数生成器等操作,都是我们应当尽量避免为每个元素都配置一次的工作。
    如果使用 map 进行创建的话，会不停的创建连接、关闭连接、使用 mapPartitions 每个分区建一次连接
    缺点：单分区数据量大可能会oom。
    疑问：如何合理的创建分区数量

2、join 优化：提前hash 分区 并且持久化

    如 A join B，
    1、普通join 原理：将A、B两个表，首先将A、B key 的hash值求出，再将hash值相同的数据传输到同一台机器，进行连接操作    

    2、优化的原理（尤其适用大小表join）： 将A 表先进行hash 分区 并且持久化，这样 A join B 表时，B 表的数据 会通过网络传输到A表所在的机器上，这样减少了 A表的网络传输开销

3、对 键值 进行连接时 使用 mapValues、flatMapValues

    目的：为了最大化优化分区相关的潜在作用

4、使用累加器时多次 触发action 导致 重复计算

    原因：多次action 导致 重复transform 操作，数据重复计算
    解决：使用cache 缓存切断依赖，newData.cache.count


5、多次使用的算子 进行 cache

6、并行度调优

    背景：rdd 进过transform 操作，派生下来的 RDD 则会采用与其父 RDD 相同的并行度
        1、假设你的应用有 1000 个可使用的计算核心，但所运行的步骤只有 30 个任务，你就应该提高并行度来充分利用更多的计算核心。
        2、当并行度过高时，每个分区产生的间接开销累计起来就会更大

    spark 提供了两种操作并行度的调优：
    1、重分区：
            coalesce 减少分区，shuffle 的默认值为 false。
            repartition 增加分区，该操作内部其实执行的是 coalesce 操作，参数 shuffle 的默认值为 true。
            分区减少 不shffle，分区变多 shuffle
            优化案例： source 读取时的 分区 个数为 1000个，filter 以后的rdd 分区个数 还是1000个，应该调用 coalesce 减少分区数

7、内存优化

    spark 内存分为三部分：
        1、rdd存储：
            当调用 RDD 的 persist() 或 cache() 方法时，这个 RDD 的分区会被存储到缓存区中。
            Spark 会根据 spark.storage.memoryFraction 限制用来缓存的内存占整个 JVM 堆空间的比例大小。如果超出限制，旧的分区数据会被移出内存。
        2、shuffle与聚合结果缓存区
            这些缓存区用来存储聚合操作的中间结果，以及数据混洗操作中直接输出的部分缓存数据
        3、用户代码
            比如 数组存储数据等需要使用内存，用户代码可以访问 JVM 堆空间中除分配给 RDD 存储和数据混洗存储以外的全部剩余空间。
    默认情况下   rdd存储内存占比 60%    shuffle与聚合结果缓存区 20%   用户代码 20%
    
        1、rdd 存储级别优化1⃣️：cache 会以 only_memory 操作，RDD 分区时空间不够，旧的分区就会直接被删除。当用到这些分区数据时，再进行重算。
           因此当内存不够时：使用MEMORY_AND_DISK 的存储级别会更好，因为在这种存储等级下，内存中放不下的旧分区会被写入磁盘，当再次需要用到的时候再从磁盘上读取回来。这样的代价有可能比重算各分区要低很多，也可以带来更稳定的性能表现。
 
           rdd 存储级别优化2⃣️：使用 MEMORY_ONLY_SER 或者 MEMORY_AND_DISK_SER 将缓存序列化后存储，而非直接存储，这种缓存方式会把大量对象序列化为一个巨大的缓存区对象。如果你需要以对象的形式缓存大量数据（比如数 GB 的数据），或者是注意到了长时间的垃圾回收暂停，可以考虑配置这个选项
           原因：jvm垃圾回收的代价与堆里的对象数目相关，而不是和数据的字节数相关，虽然序列化对象会消耗一些代价，不过这可以显著减少 JVM 的垃圾回收时间



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
    schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)   把封装好的taskSet，加入到任务调度队列中。
    （SchedulerBackend）backend.reviveOffers() 这个地方就是向资源管理器发出请求，请求任务的调度

## 面试题


#### 1、stage 根据什么决定task 数
    一个stage 会被封装成一个 taskset，stage 根据分区划分成一个个 task

#### 2、spark 为什么比 MR 快
    1、IO操作：Spark基于内存，MR 基于磁盘，MR 每次shuffle 都写入磁盘，Spark shuffle 不一定写到磁盘，而是可以缓存到内存中 便于后续迭代操作
    2、shuffle：MR 每次写reduce 都要进行shuffle，spark 宽依赖才会
    3、jvm 优化：MR是以进程的方式运行在yarn集群中，比如一个job有1024个MapTask，这个时候就需要开启1024个进程取处理这 1024个task，每启动一个task就会启动一次jvm。Spark的任务是以线程方式运行在进程中的，只在启动Executor进程时启动一 次jvm，每次执行一个task都是复用Executor进程中的线程（Executor中维护着一个线程池）。Spark和MR相比节省了大量启动 jvm的时间

#### 3、spark shuffle 的两种方式 hash shuffle 与 sort shuffle

spark1.2 前默认的 shuffle 是 hash shuffle，1.2 之后 默认的是 sort shuffle

hash shuffle 原理

    与 mr 一样，key 的hash值 % 分区数 来进行分区,有 Map有 N 个 task Reduce 有 M个task，会生成 MN个小文件
    合并机制的hash shuffle：会将一个 executor 的文件进行合并，Map 有 N 个 Executor Reduce 有 M个task，会生成 MN个小文件

sort shuffle：
    
    sort shuffle 运行机制主要分为两种：普通运行机制，另一种bypass运行机制
    bypass 运行机制触发条件：
    1）当shuffle read task 的数量小于spark.shuffle.sort.bypassMergeThreshold参数的值时（默认是200个）
    2）不是聚合类算子（reduceByKey）

    普通shuffle：
        1）写入内存数据结构
            数据会先写入一个内存数据结构中（默认是5M），此时根据不同的shuffle算子，可能选用不同的数据结构。
            如果是聚合类操作，选用map数据结构，一边聚合一边写入内存，如果是join，那么就选用Array的数据结构，直接写入内存。
            接着，每写一条数据进入内存数据结构之后，就会判断一下，是否达到了某个临界阈值。如果达到临界阈值的话，那么就会尝试将内存数据结构中的数据溢写到磁盘（会先写到内存缓冲区），然后清空内存数据结构。
        2）排序
        在溢写到磁盘文件之前，会先根据key对内存数据结构中已有的数据进行排序。
        3）溢写
        对于排序之后的数据 ，会分批写入磁盘文件，数据会以每批一万条写入磁盘文件。首先会将数据缓存在内存缓冲区中，当内存缓冲区满了之后再一次写入磁盘文件，这样可以减少磁盘IO次数，提升性能。
        4）merge归并排序
        一个task将所有数据写入内存数据结构的过程中，会发生多次磁盘溢写操作，也就是产生多个临时文件。最后将所有的临时磁盘文件都进行合并,写入最终的磁盘文件中。再写一份索引文件，标识了下游各个task需要的数据在这个磁盘文件中的start offset和end offset
        注意：一个map task会产生一个索引文件和磁盘大文件

    sortshuffle的bypass机制，和sortshuffle的普通机制不一样的地方是：
        （1）写机制（shuffle write）直接写入内存缓冲区，没有内存数据结构了，因为bypass的触发条件之一就是不能是聚合类算子。
        （2）不会进行排序，节省了这部分的性能开销

    sortShuffle的普通机制和bypass机制生成文件个数都是：
    第一个stage有N个task 最后生成N个数据文件和N个索引文件

![图 3](../../../images/0897818fab826e5768b254440de726fce212ba1012c846f36d9c1083bbd32be1.png)  

![图 4](../../../images/d8951e76f9bcaab19fda7da8955a582b8c0e584bbdff08f6709414a990c0d31d.png)  

#### 4、spark cache、persist 和 checkpoint 的区别

    cache()调用的persist()，是使用默认存储级别的快捷设置方法
    cache()是persist()的简化方式，调用persist的无参版本，也就是调用persist(StorageLevel.MEMORY_ONLY)
