# MR 总结

参考：https://zhuanlan.zhihu.com/p/85666077


####MR 的执行流程
    https://bbs.huaweicloud.com/blogs/218023

    hive 估算reduce 的博客：
    https://blog.csdn.net/strongyoung88/article/details/105864767

    hive reduce 数量

    Reduce数量
    Reduce任务的数量，首先是取用户设置的配置reduce数量，如果在没有指定数量的时候，是由程序自动估算出来的，具体情况如下：
    
    1、Map Join的时候，没有reduce数量
    2、如果有配置mapreduce.job.reduces，则使用这个值作为reduce数量
    3、如果没有配置mapreduce.job.reduces，则进行reduce估算过程，具体的估算过程如下
    
    获取bytesPerReducer，由hive.exec.reducers.bytes.per.reducer配置，默认256Mb
    获取maxReducers，由hive.exec.reducers.max配置，默认1009
    计算samplePercentage
![图 9](../../../images/cad02e1c617bc10e8353b54485974900be5990f39ece0d89d21877f00f3238ce.png)  
    计算totalInputFileSize，总的输入文件的大小
    计算powersOfTwo，如果使用bucket，并且hive.exec.infer.bucket.sort.num.buckets.power.two=true时才生效，默认这个值为false
    再根据以下判断逻辑计算出最终的估算的reduce数量：
    double bytes = Math.max(totalInputFileSize, bytesPerReducer);
    int reducers = (int) Math.ceil(bytes / bytesPerReducer);
    reducers = Math.max(1, reducers);
    reducers = Math.min(maxReducers, reducers);

    HADOOP 2.7.3

    
    1、MapReduce作业中Map Task数目的确定：  
    1）MapReduce从HDFS中分割读取Split文件，通过Inputformat交给Mapper来处理。Split是MapReduce中最小的计算单元，一个Split文件对应一个Map Task
    2）默认情况下HDFS种的一个block，对应一个Split。
    3）当执行Wordcount时：
    （1）一个输入文件小于64MB，默认情况下则保存在hdfs上的一个block中，对应一个Split文件，所以将产生一个Map Task。
    （2）如果输入一个文件为150MB，默认情况下保存在HDFS上的三个block中，对应三个Split文件，所以将产生三个Map Task。
    （3）如果有输入三个文件都小于64MB，默认情况下会保存在三个不同的block中，也将产生三个Map Task。
    4）用户可自行指定block与split的关系，HDSF中的一个block，一个Split也可以对应多个block。Split与block的关系都是一对多的关系。
    5）总结MapReduce作业中的Map Task数目是由：
    （1）输入文件的个数与大小
    （2）hadoop设置split与block的关系来决定。
    
    2、MapReduce作业中Reduce Task数目的指定：  
    1）JobClient类中submitJobInternal方法中指定：int reduces=jobCopy.getNumReduceTasks();
    
    2）而JobConf类中，public int getNumReduceTasks(){return geInt("mapred.reduce.tasks",1)}
    因此，Reduce Task数目是由mapred.reduce.tasks指定，如果不指定则默认为1.
    这就很好解释了wordcount程序中的reduce数量为1的问题，这时候map阶段的partition（分区）就为1了

    分片大小splitSize=Math.max(minSize, Math.min(goalSize, blockSize));
    long goalSize = totalSize / (numSplits == 0 ? 1 : numSplits);，totalSize为输入文件的总大小，numSplits为用户配置的map数量（mapreduce.job.maps)
    minSize，用户配置的最小分片大小
    blockSize，文件块大小，默认128M



    优化：
    1、调大）NodeManager默认内存8G，需要根据服务器实际配置灵活调整，例如128G内存，配置为100G内存左右 
       参数：yarn.nodemanager.resource.memory-mb。
    （5）mapreduce.map.java.opts：控制MapTask堆内存大小。（如果内存不够，报：java.lang.OutOfMemoryError）
    （6）mapreduce.reduce.java.opts：控制ReduceTask堆内存大小。（如果内存不够，报：java.lang.OutOfMemoryError）


#### MR on Yarn 流程
    1、MR 向 RM 提交JOB,RM 为 提交的作业生产JOB ID（application_数字），此时 JOB 状态为：NEW
    2、MR 继续向 RM 提交 JOB 的详细信息(分片、文件系统路径等)，JOB 状态为：SUBMIT
    3、RM 检查 MR 提交程序所属 queue 是否有足够的资源，此时的状态为：accept
    4、如果 有足够的资源，RM 会为 AM 分配contianer，并运行AM，AM 成功启动后此时状态为：RUNNING
    5、然后 AM 向 RM 申请 maptask 资源运行 MR 程序的 map阶段
    6、map 调用 inputFormat 的接口，以kv的形式从文件系统读取数据，map 阶段主要是对 文件进行分片并读入数据，因此map 端出现oom，需要调大堆内存
    7、map 调用 outputCollector向环形缓冲区写入数据
    8、环形缓冲区默认大小（100M），当容量达到 80%时，进行一个溢写，写的时候会根据 partition （默认：Hash）的规则，写入分区并且做归并排序，此时可以选择做 combiner 操作，做局部汇总 减少网络IO
    9、Map 结束后，RM AM 继续向 RM 申请 continer 运行 reducetask
    10、reduce task 数量与 partition 数量保持一致，拉取对应 partition 的数据做相应的逻辑处理
    11、JOB顺利 完成以后，状态为：FINALSHED
    12、

map 通过 partitioner（默认：hash） 的组建，将相同的k 写入同一个分区

    1、client 向RM 发起请求，RM 为 提交的作业生产JOB ID，此时 JOB 状态为：NEW
    2、client 继续向 RM 提交 JOB 的详细信息，JOB 状态为：SUBMIT
    3、RM 检查要运行 AppLication Master 的 queue 是否有足够的资源，此时 JOB 状态为：accept
    4、AM 启动成功后，与RM 申请运行程序的资源，并检查状态
    5、如果JOB按照预期完成。此时，JOB的状态为FINISHED。如果运行过程中出现故障，此时，JOB的状态为FAILED。如果客户端主动kill掉作业，此时，JOB的状态为KILLED。


#### 总结
    1、client 向 ResourceManager 提交job，携带了 文件地址、job.xml、wcJar
    2、首先读取文件的组件 InputFormat（默认 TextInputFormat） 会通过 getSplits() 获取切片数，多数来哥 splits 就有多少个 MapTask【默认情况 splits 与 block 块一对一的关系】
    3、将输入文件切分为splits之后，由RecordReader对象（默认LineRecordReader）进行读取，以\n作为分隔符，读取一行数据，返回<key，value>。Key表示每行首字符偏移值，value表示这一行文本内容。
    4、读取split返回<key,value>，进入用户自己继承的Mapper类中，执行用户重写的map函数。RecordReader读取一行用户重写的map调用一次，并输出一个<key,value>。
    5、map 输出的数据会写入内存，内存中这片区域叫环形缓冲区，作用收集map 结果，减少io，环形缓冲区其实是一个数组，数组中存放着key、value的序列化数据和key、value的元数据信息，包括partition、key的起始位置、value的起始位置以及value的长度。环形结构是一个抽象概念
    6、缓冲区 在 数据达到 默认的 （100M * 0.8） 时 会进行溢写，这80M 会被锁住 开始写入磁盘，新的数据可以继续往 20M 里面写。
    7、合并溢写文件：每次溢写会在磁盘上生成一个临时文件（写之前判断是否有combiner），如果map的输出结果真的很大，有多次这样的溢写发生，磁盘上相应的就会有多个临时文件存在。当整个数据处理结束之后开始对磁盘中的临时文件进行merge合并，因为最终的文件只有一个，写入磁盘，并且为这个文件提供了一个索引文件，以记录每个reduce对应数据的偏移量。至此map整个阶段结束


#### MapReduce OOM 分析

参考：https://www.daimajiaoliu.com/daima/479e022779003ec

    1、Mapper/Reducer阶段JVM堆内存溢出参数调优
    mapreduce.map.java.opts=-Xmx2048m(默认参数，表示jvm堆内存,注意是mapreduce不是mapred)
    mapreduce.map.memory.mb=2304(container的内存）
    
    Reducer:
    mapreduce.reduce.java.opts=-=-Xmx2048m(默认参数，表示jvm堆内存)
    mapreduce.reduce.memory.mb=2304(container的内存）

    因为在yarn container这种模式下，map/reduce task是运行在Container之中的，所以上面提到的mapreduce.map(reduce).memory.mb大小都大于mapreduce.map(reduce).java.opts值的大小。
    mapreduce.{map|reduce}.java.opts能够通过Xmx设置JVM最大的heap的使用，一般设置为0.75倍的memory.mb，因为需要为java code等预留些空间

## MapReduce 执行流程

1.1. Client：编写mapreduce程序，配置作业，提交作业的客户端 ；
1.2. ResourceManager：集群中的资源分配管理 ；
1.3. NodeManager：启动和监管各自节点上的计算资源 ；
1.4. ApplicationMaster：每个程序对应一个AM，负责程序的任务调度，本身也是运行在NM的Container中 ；
1.5. HDFS：分布式文件系统，保存作业的数据、配置信息等等。

1、客户端编RM发起提交请求，RM为提交的作业生产JOB ID，此时 JOB 状态为：NEW；
2、然后客户端继续向RM提交 JOB 详细信息包括(分片、文件系统路径等)，此时 JOB的状态为：SUBMIT
3、RM 检查 MR 提交程序所属 queue 是否有足够的资源，此时的状态为：accept
4、如果 有足够的资源，RM 会为 AM 分配contianer，并运行AM，AM 成功启动后此时状态为：RUNNING
5、然后 AM 向 RM 申请 maptask 资源运行 MR 程序的 map阶段
6、map 调用 inputFormat 的接口，以kv的形式从文件系统读取数据，map 阶段主要是对 文件进行分片并读入数据
7、map阶段完成以后，调用outputCollector 接口 向环形缓冲区写入数据
8、环形缓冲区默认大小（100M 建议优化成 1.5G),当容量达到 80%时，进行一个溢写，写的时候会根据 partition（默认：HashPartition）的规则，写入分区并且做归并排序，此时可以选择做 combiner 操作，做局部汇总 减少网络IO,然后压缩输出到 本地磁盘，这样 Map端就结束了
9、Map 结束后，AM 继续向 RM 申请 continer 运行 reducetask
10、Reduce 会开启几个复制线程 默认5个 （建议值20） 去复制map结果中对应的partition的数据，复制时候reduce还会进行排序操作和合并文件操作，这些操作完了就会按照程序 reduce计算了。
11、JOB顺利 完成以后，任务的状态为：FINALSHED