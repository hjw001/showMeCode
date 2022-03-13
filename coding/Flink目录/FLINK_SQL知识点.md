# Flink sql 知识点


## flink join 的几种方式、

 传送门：https://juejin.cn/post/6844903922771951630
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

## 表类型


### 动态表 Dynamic Table

    定义：动态表的查询是一个连续查询且不会终止，查询不断更新其(动态)结果表

#### 动态表与流的关系

    将流转换为动态表。
    在动态表上计算一个连续查询，生成一个新的动态表。
    生成的动态表被转换回流。

#### Append-only 流

    定义：仅通过 INSERT 操作修改的动态表可以通过输出插入的行转换为流。

    append流案例：

 ![图 5](../../../images/acda19da435f647e0c8b9f964279296699f3ce728e64d30704cec85a022fd2b8.png)  

#### Retract 流

    定义：retract 流包含两种类型的 message： add messages 和 retract messages 。通过将INSERT 操作编码为 add message、将 DELETE 操作编码为 retract message、将 UPDATE 操作编码为更新(先前)行的 retract message 和更新(新)行的 add message，将动态表转换为 retract 流。

    案例：

![图 6](../../../images/581b72ab5ffec4bed237c198db4c7d5bd6f9235ff7a219ac221b86a06ef9eec2.png)  

#### Upsert 流

    定义：Upsert 流: upsert 流包含两种类型的 message： upsert messages 和delete messages。转换为 upsert 流的动态表需要(可能是组合的)唯一键。通过将 INSERT 和 UPDATE 操作编码为 upsert message，将 DELETE 操作编码为 delete message ，将具有唯一键的动态表转换为流。消费流的算子需要知道唯一键的属性，以便正确地应用 message。与 retract 流的主要区别在于 UPDATE 操作是用单个 message 编码的，因此效率更高。

    案例：

![图 7](../../../images/a7faa93d2d90c4ef4f6d74c4d9f368200180e975e4e6e034013f317fceddb448.png)  


