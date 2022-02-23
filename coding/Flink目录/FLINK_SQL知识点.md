# Flink sql 知识点


## flink join 的几种方式

 regular join
 interval join
 temproal table join

### regular join

    定义：两条无边界的流 进行join，任何一侧数据流有更改都是可见的，直接影响整个 join 结果
    语法：
    SELECT columns
    FROM t1  [AS <alias1>]
    [LEFT/INNER/FULL OUTER] JOIN t2
    ON t1.column1 = t2.key-name1

### interval join

    定义：interval Join 利用窗口的给两个输入表设定一个 Join 的时间界限，超出时间范围的数据则对 join 不可见并可以被清理掉
    SELECT columns
    FROM t1  [AS <alias1>]
    [LEFT/INNER/FULL OUTER] JOIN t2
    ON t1.column1 = t2.key-name1 AND t1.timestamp BETWEEN t2.timestamp  AND  BETWEEN t2.timestamp + + INTERVAL '10' MINUTE;

### temproal table join

    定义：temproal table 更新对另一表在该时间节点以前的记录是不可见的，适用于维表join

    SELECT columns
    FROM t1  [AS <alias1>]
    [LEFT] JOIN t2 FOR SYSTEM_TIME AS OF t1.proctime [AS <alias2>]
    ON t1.column1 = t2.key-name1

