## hive 优化

### 查询 优化
    1、使用 group by 代替  distinct
    2、order by 使用 distribute by + sort by  代替
    
    limit 优化，一般情况下，Limit语句还是需要执行整个查询语句，然后再返回部分结果。有一个配置属性可以开启，避免这种情况—对数据源进行抽样
    hive.limit.optimize.enable = true;

### order by、distribute by 、cluster by、sort by、partition by 区别
    order by 
    distribute by 控制map的数据如何输出给reduce，默认是 将hash值相同的key 输出到同一个reduce
    sort by 对单reduce 排序
    cluster by = distribute by  + sort by 只能升序


## 文件存储格式
    采用 orc存储
    ORC是列式存储，有多种文件压缩方式，并且有着很高的压缩比。
    使用ORDC可以HDFS存储资源，查询任务的输入数据量减少，使用的MapTask也就减少了。
    # 向量化，基于orc格式，可大大减少典型查询操作(如扫描，过滤器，聚合和联接)的 CPU 使用率
    set hive.vectorized.execution.enabled=true

#### 1、小文件合并
    是否合并Map输出文件，默认值为真
    hive.merge.mapfiles=true


#### map join 

    原理：broadcast join，将小表分发到各个节点，在map 阶段进行join，省去了reduce阶段
    hive.auto.convert.join=true
    hive.auto.convert.join ： 是否自动转换为mapjoin
    hive.mapjoin.smalltable.filesize : 小表的最大文件大小，默认为25000000，即25M
    hive.auto.convert.join.noconditionaltask ： 是否将多个mapjoin合并为一个
    hive.auto.convert.join.noconditionaltask.size ： 多个mapjoin转换为1个时，所有小表的文件大小总和的最大值

    hive.vectorized.execution.mapjoin.native.multikey.only.enabled
    此标志应设置为 true，以使矢量 Map 联接哈希表能够使用 MapJoin 对整数联接查询使用 max/max 过滤。

    hive.vectorized.execution.mapjoin.minmax.enabled
    此标志应设置为 true，以允许在使用 MapJoin 的查询中使用本机快速矢量 Map 联接哈希表。
    hive.vectorized.execution.mapjoin.native.fast.hashtable.enabled

    

#### 2、map 端 oom、combiner
    map端调大内存   set mapreduce.map.memory.mb=4096; 
    map端调大堆内存  set mapred.map.child.java.opts
    mapreduce.map.memory.mb是向RM申请的内存资源大小，这些资源可用用于各种程序语言编写的程序, mapred.map.child.java.opts 一般只用于配置JVM参数
    开启map 端的 combiner 设置 hive.map.aggr=true

    设置本地化运行，小于128M的文件，直接本地模式在单台机器上处理所有的任务，
    hive.exec.mode.local.auto
    hive.exec.mode.local.auto.inputbytes.max = 134217728 （128M）
    set hive.exec.mode.local.auto.input.files.max = 5 默认4

#### 4、参数优化

    内存参数：
        mapreduce.map.cpu.vcores 默认值 1 设置为 1
        mapreduce.map.memory.mb 默认1024 设置为 4096
        mapreduce.map.java.opts -Xms3800m -Xmx3800m

        reduce 同上

        mapreduce.map.output.compress 默认值 false 设置为true Map输出结果在网络传输前被压缩

        mapreduce.job.reduce.slowstart.completedmaps 默认值 0.05 设置为 0.9 Map完成指定比例后开启reduce执行
        mapreduce.job.ubertask.enable 默认值False 设置为 true 开启JVM重用

        #动态分区优化
        SET hive.optimize.sort.dynamic.partition=true;
        设置这个参数后，map 在写入reduce 时会将 分区字段排序，由于分区字段是排序的，因此每个reducer只需要保持一个文件写入器（file writer）随时处于打开状态，在收到来自特定分区的所有行后，关闭记录写入器（record writer），从而减小内存压力。

        知识点：一个yarn container 对应一个核

     JVM 参数：
        -XX:+UseNUMA -Xmx3800m -Djava.net.preferIPv4Stack=true -XX:ParallelGCThreads=2 -XX:CICompilerCount=2 -XX:+UseParallelOldGC
        NUMA:定义：非一致性内存访问（Non-Uniform Memory Access、NUMA）是一种计算机内存的设计方式
            优点：NUMA 引入了本地内存和远程内存，CPU 访问本地内存的延迟会小于访问远程内存
        ParallelGCThreads:垃圾回收线程数，小于8核时，设为核数或者两倍
        UseParallelOldGC: 
                定义：老年代ParallelOldGC回收器也是一种多线程的回收器，是一种关注吞吐量的回收器，使用的是标记压缩算法
                标记压缩算法:定义:标记压缩的最终效果等于标记清除算法完成之后在进行一次内存碎片整理
                    优点:
                        1、消除了标记-清除算法当中，内存区域分散的缺点，我们需要给新对象分配内存时，JVM只需要持有一个内存的起始地址即可。
                        2、消除了复制算法当中，内存减半的高额代价。
                    缺点:
                        1、从效率上来说，标记-整理算法要低于复制算法。
                        2、移动对象的同时，如果对象被其他对象引用，则还需要调整引用的地址。
                        3、移动过程中，需要全程暂停用户应用程序。即：STW

    //150w数据 对应 250M 数据 
    倾斜优化：
        join 优化
        原来：依赖元数据，在编译期判定是否数据倾斜，如果倾斜的key超过N个，这部分数据会输出，另请一个计划 将这部分数据做 map join，最终结果将两个计划做 union all
        
        set hive.optimize.skewjoin.compiletime=true; #编译期判定是否数据倾斜，依赖元数据信息,与hive.optimize.skewjoin.compiletime一起设为true
        set hive.optimize.skewjoin=true;
        set hive.skewjoin.key=1000000;  #判定倾斜key的行数
        set hive.skewjoin.mapjoin.map.tasks=10000; #数据倾斜执行计划中，map任务最大个数
        set hive.skewjoin.mapjoin.min.split=33554432; #数据倾斜执行计划中，每个map的处理的最小数据量 32M

        group by 优化
        set hive.groupby.skewindata=true;

    压缩参数：
        常见的压缩技术有：bzip2、gzip、izo、snappy
        采用 snappy 压缩比高，压缩速度快
        设置map输出压缩
        set hive.exec.compress.intermediate=true ;
        set mapreduce.map.output.compress=true ;
        set mapreduce.map.output.compress.codec=org.apache.hadoop.io.compress.SnappyCodec ;
        设置reduce输出压缩
        set hive.exec.compress.output=true;
        set mapreduce.output.fileoutputformat.compress=true ;
        set mapreduce.output.fileoutputformat.compress.codec=org.apache.hadoop.io.compress.SnappyCodec ;


## 总结

    1、文件格式调优：
    采用 orc存储
    ORC是列式存储，有多种文件压缩方式，并且有着很高的压缩比。使用ORDC可以HDFS存储资源，查询任务的输入数据量减少，使用的MapTask也就减少了。
    #基于orc格式的向量化查询，可大大减少典型查询操作(如扫描，过滤器，聚合和联接)的 CPU 使用率
    set hive.vectorized.execution.enabled=true；

    2、本地运行模式
    设置本地化运行，小于128M的文件，直接本地模式在单台机器上处理所有的任务，
    hive.exec.mode.local.auto=true
    hive.exec.mode.local.auto.inputbytes.max = 134217728 （128M）
    set hive.exec.mode.local.auto.input.files.max = 5 默认4

    3、Map端调优

        2、1:小文件合并优化，小于1g的文件会被合并，大于1g会被切片成1g
        hive.input.format = org.apache.hadoop.hive.ql.io.HiveInputFormat （默认的CombineHiveInputFormat对LZO文件无效）
        mapreduce.input.fileinputformat.split.minsize=1073741824 （1g 8个分片）
        mapreduce.input.fileinputformat.split.maxsize=1073741824 （1g）

        2、2:内存调优:分配给map的container内存设置为4g，堆内存最大最小都设置为 3800M，这样可以保证 GC不会扩展或缩小JVM堆内存，核数默认为1核即可
        mapreduce.reduce.memory.mb=4096 
        mapreduce.map.java.opts=-Xms3800m -Xmx3800m

        2、3:Map Join优化：
        默认左边的表为小表
        hive.mapjoin.check.memory.rows = 1百万; 默认参数 10w;内存估算，默认每10万条估算一次
        hive.skewjoin.mapjoin.min.split = 32; 数据倾斜执行计划中，每个map的处理的最小数据量
        hive.mapjoin.smalltable.filesize=25000000; 小表阈值，超过为大表

        2、4:Map 输出压缩：
        mapreduce.map.output.compress = true; Map输出结果在网络传输前被压缩
        set mapreduce.map.output.compress.codec=org.apache.hadoop.io.compress.SnappyCodec; 
        采用snappy压缩,压缩速度快 409M/s,22%压缩比,而磁盘IO速度为 50M/s

        2、5:Map Combiner
        hive.map.aggr=true = true; map提前聚合，减少传输数据io以及reduce 处理数据量

    
    3、shuffle端优化

        3、1:
        mapreduce.task.io.sort.factor = 64；默认值 10，shuffle排序处理器，可以使合并处理更快，并减少磁盘的访问
        mapreduce.task.io.sort.mb = 1537; 默认值100， Shuffle排序内存
        mapreduce.reduce.shuffle.parallelcopies = 20;默认值5,Shuffle数据拷贝线程,Reduce 的复制线程,复制map结果中对应的parition的数据
        mapreduce.shuffle.connection-keep-alive.enable = true; Shuflle数据拉取连接

    4、Reduce 端优化

        Reduce 基本上是做逻辑处理的阶段，由于相同的 key 值会分发到 同一个 Reduce 任务重，Key值过多会导致数据倾斜

        倾斜优化：
        设置超过 100W的单 key做优化，100W 数据量 对应 50 - 200+ M
        join 优化
        原来：依赖元数据，在编译期判定是否数据倾斜，如果倾斜的key超过N个，这部分数据会输出，另请一个计划 将这部分数据做 map join，最终结果将两个计划做 union all
        
        set hive.optimize.skewjoin.compiletime=true; #编译期判定是否数据倾斜，依赖元数据信息,与hive.optimize.skewjoin.compiletime一起设为true
        set hive.optimize.skewjoin=true;
        set hive.skewjoin.key=1000000;  #判定倾斜key的行数
        set hive.skewjoin.mapjoin.map.tasks=10000; #数据倾斜执行计划中，map任务最大个数
        set hive.skewjoin.mapjoin.min.split=33554432; #数据倾斜执行计划中，每个map的处理的最小数据量 32M

        group by 优化
        set hive.groupby.skewindata=true;
    
    5、JVM 优化:
        使用 java -XX:+PrintCommandLineFlags -version 查看 JVM默认参数
        垃圾回收器要使用关注吞吐量的多线程回收器,ParallelGC 与 ParallelOldGC
        默认的 JVM 使用的 ParallelGC 是使用的 复制算法,会造成一定的空间浪费。

        JVM 参数：
        -XX:+UseNUMA -Xmx3800m -Djava.net.preferIPv4Stack=true -XX:ParallelGCThreads=2 -XX:CICompilerCount=2 -XX:+UseParallelOldGC
        NUMA:定义：非一致性内存访问（Non-Uniform Memory Access、NUMA）是一种计算机内存的设计方式
            优点：NUMA 引入了本地内存和远程内存，CPU 访问本地内存的延迟会小于访问远程内存
        ParallelGCThreads:垃圾回收线程数，小于8核时，设为核数或者两倍
        UseParallelOldGC: 
                定义：老年代ParallelOldGC回收器也是一种多线程的回收器，是一种关注吞吐量的回收器，使用的是标记压缩算法
                标记压缩算法:定义:标记压缩的最终效果等于标记清除算法完成之后在进行一次内存碎片整理
                    优点:
                        1、消除了标记-清除算法当中，内存区域分散的缺点，我们需要给新对象分配内存时，JVM只需要持有一个内存的起始地址即可。
                        2、消除了复制算法当中，内存减半的高额代价。
                    缺点:
                        1、从效率上来说，标记-整理算法要低于复制算法。
                        2、移动对象的同时，如果对象被其他对象引用，则还需要调整引用的地址。
                        3、移动过程中，需要全程暂停用户应用程序。即：STW