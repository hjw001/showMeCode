# Window 和 WaterMark 知识点

## Window 知识点

    定义：将有界或无界的数据流，按照时间边界进行切分，即切分无限的数据集到有限的数据集

### Window Assigner

    定义：某个元素分配到哪个窗口去,调用 assignWindows 将数据按照规则分配到指定窗口,并返回窗口集合
    window 本身只是一个id 标识符，窗口元素 以key Value state存储，key为 window，value为元素集合

### Window trigger（触发器）

    定义：窗口何时能被计算或清除，每个窗口都有自己的trigger
    触发器有两个枚举值（boolean 类型）：
        fire 表示是否触发计算
        purge 表示是否清除窗口数据
        CONTINUE 表示不对window做任何操作；
        FIRE_AND_PURGE 表示要触发window的computation操作然后清理window的窗口数据；
        FIRE 表示仅仅触发window的computation操作但不清理window的窗口数据；
        PURGE 表示不触发window的computation操作但是要清理window的窗口数据

### Window Evictor

    定义：在Trigger触发之后，在窗口被处理之前，Evictor（如果有Evictor的话）会用来剔除窗口中不需要的元素，相当于一个filter。

## WaterMark 知识点
传送门：

https://developer.aliyun.com/article/682873?spm=a2c6h.13262185.0.0.3fe58a3bDWnVz9

### WaterMark定义

    WaterMark是为了使用 事件时间语义，Flink应用程序需要知道事件时间所对应的字段
    通过使用 TimestampAssigner API 从元素中的某个字段去访问/提取时间戳
    然后将提取出来的时间戳 使用WatermarkGenerator配置 waterMark的生成方式
    使用 Flink API 时需要设置一个同时包含 TimestampAssigner 和 WatermarkGenerator 的 WatermarkStrategy。WatermarkStrategy 工具类中也提供了许多常用的 watermark 策略

### WatermarkGenerator（生产水印，自定义水印）

    提供两个方法：
        onEvent：每个元素都会调用这个方法，如果我们想依赖每个元素生成一个水印，然后发射到下游(可选，就是看是否用output来收集水印)，我们可以实现这个方法.
        onPeriodicEmit：（周期发射水印）每条数据都生成一个水印的话，会影响性能，所以这里还有一个周期性生成水印的方法。这个水印的生成周期可以这样设置：env.getConfig().setAutoWatermarkInterval(5000L);

    BoundedOutOfOrdernessWatermarks 延迟水印源码：
    延迟水印的当前水印 等于 最大时间 - 延迟时间

        public void onEvent(T event, long eventTimestamp, WatermarkOutput output) {
		    maxTimestamp = Math.max(maxTimestamp, eventTimestamp);
	    }

        public void onPeriodicEmit(WatermarkOutput output) {
            output.emitWatermark(new Watermark(maxTimestamp - outOfOrdernessMillis - 1));
        }

    waterMark 的值等于 这一次与上一次 time 相比最大的值 - 设置的延时时间，因此下一次eventTime 大于上一次的time  waterMark会更新

### TimestampAssignerSupplier （注册水印）

    调用 extractTimestamp 将 eventTime 注册成水印

## waterMark 触发 window 计算

    EventTimeTrigger 源码：

    public TriggerResult onElement(
            Object element, long timestamp, TimeWindow window, TriggerContext ctx)
            throws Exception {
        if (window.maxTimestamp() <= ctx.getCurrentWatermark()) {
            // if the watermark is already past the window fire immediately
            return TriggerResult.FIRE;
        } else {
            ctx.registerEventTimeTimer(window.maxTimestamp());
            return TriggerResult.CONTINUE;
        }
    }

    当窗口的结束时间 <= 当前waterMark时，触发窗口计算

## Window Function

    window fuction 分为增量计算和全量计算

![图 1](../../images/3012331b713bc918d3736655b4062bb94090f47478d6b8a44b3a465deea2ef01.png)  
