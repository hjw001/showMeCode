## hive 优化

### sql 优化
    1、使用 group by 代替  distinct

#### 1、小文件合并
    map 之前开启小文件合并，小文件多会导致maptask 过多，set hive.merge.mapredfiles=true;
    每个map 的最大输入大小
    set mapred.max.split.size=1024000000;
    每个map 的最小输入大小
    set mapred.min.split.size=1024000000;
    set hive.input.format =org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;#执行Map前进行小文件合并

#### map join 
     hive.auto.convert.join=true

#### 2、map 端 oom、combiner
    map端调大内存   set mapreduce.map.memory.mb=4096; 
    map端调大堆内存  set mapred.map.child.java.opts
    mapreduce.map.memory.mb是向RM申请的内存资源大小，这些资源可用用于各种程序语言编写的程序, mapred.map.child.java.opts 一般只用于配置JVM参数
    开启map 端的 combiner 设置 hive.map.aggr=true


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

#### 4、动态分区优化及其原理

    1、目标表是 ORC 或 Parquet时，Parquet和ORC是列式批处理文件格式。这些格式要求在写入文件之前将批次的行（batches of rows）缓存在内存中
    2、当 直接 insert overwrite partition select 没有 join 以及聚合时，只有map阶段，此时执行计划是，假设 木笔哦啊每个 map 