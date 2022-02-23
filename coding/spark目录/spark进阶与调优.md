# spark 进阶与调优

<br/>

## spark进阶

<br/>

## spark优化
    参考：
    https://github.com/leizuquan1990/java_spark
    

<br/>

### 1、特定情况下使用 mapPartitions
    
    诸如打开数据库连接或创建随机数生成器等操作,都是我们应当尽量避免为每个元素都配置一次的工作。
    如果使用 map 进行创建的话，会不停的创建连接、关闭连接、使用 mapPartitions 每个分区建一次连接
    缺点：单分区数据量大可能会oom。
    疑问：如何合理的创建分区数量

<br/>

### 2、join 优化：提前hash 分区 并且持久化

    如 A join B，
    1、普通join 原理：将A、B两个表，首先将A、B key 的hash值求出，再将hash值相同的数据传输到同一台机器，进行连接操作    

    2、优化的原理（尤其适用大小表join）： 将A 表先进行hash 分区 并且持久化，这样 A join B 表时，B 表的数据 会通过网络传输到A表所在的机器上，这样减少了 A表的网络传输开销

<br/>

### 3、对 键值 进行连接时 使用 mapValues、flatMapValues

    目的：为了最大化优化分区相关的潜在作用

<br/>

### 4、使用累加器时多次 触发action 导致 重复计算

    原因：多次action 导致 重复transform 操作，数据重复计算
    解决：使用cache 缓存切断依赖，newData.cache.count


<br/>

### 5、多次使用的算子 进行 cache

<br/>

### 6、并行度调优

    背景：rdd 进过transform 操作，派生下来的 RDD 则会采用与其父 RDD 相同的并行度
        1、假设你的应用有 1000 个可使用的计算核心，但所运行的步骤只有 30 个任务，你就应该提高并行度来充分利用更多的计算核心。
        2、当并行度过高时，每个分区产生的间接开销累计起来就会更大

    spark 提供了两种操作并行度的调优：
    1、重分区：
            coalesce 减少分区，shuffle 的默认值为 false。
            repartition 增加分区，该操作内部其实执行的是 coalesce 操作，参数 shuffle 的默认值为 true。
            分区减少 不shffle，分区变多 shuffle
            优化案例： source 读取时的 分区 个数为 1000个，filter 以后的rdd 分区个数 还是1000个，应该调用 coalesce 减少分区数

    spark并行度数量设置：
    1、task数量，至少设置成雨spark applcation的总cpu core数量相同（如：150 cpu 分配 150个task，同时运行完毕）
    2、官方推荐：设置成spark applcation的总cpu core数量的2～3倍，150 cpu 设置 300 ～500 
    设置并行度 new sparkConf().set("spark.default.parallelism","500")

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
        
        2、使用序列化时采用kryo序列化:序列化后的数据要更少,大概Java序列化的1/10。