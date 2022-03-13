# RDD 示例


<br/>

##### map 与 mapPartition 的区别

    rdd2.mapPartitions(num => num.filter(_ % 2 == 0))
    mapPartition 与 map 的区别，map是分区内一个数据一个数据处理，mapPartition（返回的是分区内的迭代器） 是以分区为单位进行批处理
    Map 算子主要目的将数据源中的数据进行转换和改变。但是不会减少或增多数据。
    MapPartitions 算子需要传递一个迭代器，返回一个迭代器，没有要求的元素的个数保持不变，所以可以增加或减少数据


<br/>

##### coalesce 减少分区

    val dataRDD1 = dataRDD.coalesce(2)

<br/>

##### repartition 增加分区
    dataRDD.repartition(4)
    该操作内部其实执行的是 coalesce 操作，参数 shuffle 的默认值为 true。无论是将分区数多的
    RDD 转换为分区数少的 RDD，还是将分区数少的 RDD 转换为分区数多的 RDD，repartition
    操作都可以完成，因为无论如何都会经 shuffle 过程。


<br/>

##### sortBy 排序
    rdd2.sortBy(num => num, false, 3)
    sortBy 会经历shuffle，sortBy 源代码实现 keyBy[K].sortByKey(ascending, numPartitions).values

<br/>

#### 两数据集求 交集、并集 subtract（not in ）、intersection (in)、union、zip(将两个rdd 以键值对存储)
    List1 = {1,2,3,4,5} 和 List1 = {3,4,5,6,7}
    list1.subtract(list2)  1 2
    list1.intersection(list2) {3,4,5}
    list1.union(list2)  {1,2,3,4,5,3,4,5,6,7}

<br/>

#### partitionBy 自定义分区器（默认 HashPartitioner）
    rdd.partitionBy(new HashPartitioner(2))


<br/>

#### reduceByKey 与 groupByKey 都是 按照key 进行聚合
    value4.reduceByKey(_+_)
    reduceByKey 和 groupByKey 都存在 shuffle 的操作，但是 reduceByKey 源码 调用了 combineByKeyWithClassTag
    这样可以在 shuffle 前对分区内相同 key 的数据进行预聚合（combine）功能，这样会减少落盘的数据量，而 groupByKey 只是进行分组，不存在数据量减少的问题，reduceByKey 性能比较高。
    从功能的角度：reduceByKey 其实包含分组和聚合的功能。GroupByKey 只能分组，不能聚合，所以在分组聚合的场合下，推荐使用 reduceByKey，如果仅仅是分组而不需要聚合。那么还是只能使用 groupByKey


<br/>

#### aggregateByKey （）里面的参数，参数表示初始值 分区内进行操作， 第二个参数表示分区间的计算规则
    dataRDD1.aggregateByKey(0)(_+_,_+_) 


<br/>

#### join()  两个 rdd（k,v） 进行join 返回 (k,[v1,v2]), leftOuterJoin rdd.leftOuterJoin(rdd5), coGroup  
    join 的源码调用了 cogroup
    val rdd: RDD[(Int, String)] = sparkContext.makeRDD(Array((1,"aaa"),(2,"bbb"),(3,"ccc")))
    val rdd5: RDD[(Int, String)] = sparkContext.makeRDD(Array((4,"cc"),(2,"acc"),(3,"bc")))
    rdd.join(rdd5).collect().foreach(println)
    (2,(bbb,acc))
    (3,(ccc,bc))

    rdd.leftOuterJoin(rdd5)
    (1,(aaa,None))
    (2,(bbb,Some(acc)))
    (3,(ccc,Some(bc)))

    rdd.cogroup(rdd5) 返回 (k,[iter1,iter2])
    (1,(CompactBuffer(aaa),CompactBuffer()))
    (2,(CompactBuffer(bbb),CompactBuffer(acc)))
    (3,(CompactBuffer(ccc),CompactBuffer(bc)))
    (4,(CompactBuffer(),CompactBuffer(cc)))
    使用 cogroup 实现 join     
    rdd.cogroup(rdd5).flatMapValues(pair => for(v <- pair._1.iterator; w <- pair._2.iterator) yield (v,w)).collect().foreach(println)

<br/>
<br/>

----

## action 算子集合

<br/>


#### reduce 聚集 RDD 中的所有元素，先聚合分区内数据，再聚合分区间数据
    rdd.reduce(_+_)

#### count 返回 RDD 中元素的个数
    rdd.count()

#### first 返回第一个元素，take 返回 前n个元素，takeOrdered 返回该 RDD 排序后的前 n 个元素组成的数组
    rdd.first()
    rdd.take(3)
    rdd.takeOrdered(2)

####