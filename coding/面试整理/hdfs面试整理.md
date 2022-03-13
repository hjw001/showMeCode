
## 角色

    block： 文件上传前需需要把文件切割存储，简单来说切割后的文件被称为块（block）。老版本默认64M/新版本默认128M。块太大map任务少，处理数据慢。块太(小又)多，寻址时间长，NN的压力也会激增。
    fsimage： 在NameNode启动时对整个文件系统的快照。
    editlog： NameNode启动后，对文件系统的系列操作，都会存储在editlog中。
    edit logs： 若干个editlog被称为edit logs。
    RPC： 通讯协议，client向namenode发送请求时就是使用的RPC。



### NameNode

    NameNode负责：文件元数据信息的操作以及处理客户端的请求
    NameNode管理：HDFS文件系统的命名空间NameSpace。
    NameNode维护：文件系统树（FileSystem）以及文件树中所有的文件和文件夹的元数据信息（matedata）
            维护文件到块的对应关系和块到节点的对应关系
    NameNode文件：namespace镜像文件（fsimage），操作日志文件（edit log）这些信息被Cache在RAM中，当然这两个文件也会被持久化存储在本地硬盘。
    NameNode记录：每个文件中各个块所在的数据节点的位置信息。
            但它并不永久保存块的位置信息，因为这些信息在系统启动时由数据节点重建。
            从数据节点重建：在nameNode启动时，DataNode向NameNode进行注册时发送给NameNode

### Hdfs读写流程
    读流程：
    client 向NameNode 提交读取请求
    NameNode 返回所有block位置信息以及文件副本的保存位置，给client
    client 调用 FSDataOututStream方法读取最适合的副本文件节点的数据 （本地→同机架→数据中心）。

![图 14](../../../images/2cb1246bb3fcba8650c3638a8392e2af2ba9d3a0ca73a93c632650997113bf78.png)  


    写流程：
    1、client 向NameNode发起文件创建请求
    2、NameNode对准备上传的文件名称和路径做校验，确定是否拥有写权限、是否有重名文件，并将操作写入到edit log文件中。
    3、NameNode 响应client 可以写入后，client 请求上传 第一个 block
    4、NameNode 返回 client DataNode的地址，client 向 第一个  DataNode建立 block 通道，并且DataNode 之间建立 pipeline 连接
    5、client 向 第一个DataNode 传输 package，然后沿通道一直传输到最后一个副本节点
    6、全部数据执行完成后调用FSDataOutputStream的 close 方法关闭流。




![图 15](../../../images/c7a9a8f3931fbc8ae8545d9715ae8d458787902410cc59d1d55417cc5f8a0822.png)  
