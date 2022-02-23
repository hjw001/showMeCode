#clickhouse 总结




###replacingMergeTree 引擎

### 参考 https://cloud.tencent.com/developer/article/1680925

#### 1、去重
    https://mp.weixin.qq.com/s?__biz=MzA4MDIwNTY4MQ==&mid=2247483804&idx=1&sn=b304f7f88d064cc08f87fa5eaafec0b7&chksm=9fa68382a8d10a9440d3ce2a92a04c4a74aeda2d959049f04f1a414c1fb8034b97d9f7243c21&scene=21#wechat_redirect
    replacingMergeTree 根据order by 可以合并相邻id重复数据，

    第一，使用ORDER BY作为特殊判断标识，而不是PRIMARY KEY。关于这一点网上有一些误传，但是如果理解了ORDER BY与PRIMARY KEY的作用，以及合并逻辑之后，都能够推理出应该是由ORDER BY决定。 ORDER BY的作用， 负责分区内数据排序;
    PRIMARY KEY的作用， 负责一级索引生成;
    Merge的逻辑， 分区内数据排序后，找到相邻的数据，做特殊处理。
    第二，只有在触发合并之后，才能触发特殊逻辑。以去重为例，在没有合并的时候，还是会出现重复数据。
    第三，只对同一分区内的数据有效。以去重为例，只有属于相同分区的数据才能去重，跨越不同分区的重复数据不能去重。


#### 2、更新
    使用 sign 标识数据是否删除 1 、-1

