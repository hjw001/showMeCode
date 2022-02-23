# Yarn 总结

参考：https://zhuanlan.zhihu.com/p/335881182


### 角色及其作用

ResourceManager：主要负责资源管理和调度, 有两个组件（ApplicationMaster、scheduler）
        
    applicationManager 监控application Master运行状态（容错）,跟踪分配给Application Master的进度和状态。
    scheduler 主要负责分配container 给 application Master，分配的算法有（Fair schedule 公平调度、FIFO schedule先进先出、Capacity schedule 容量调度）

NodeManager：节点管理器，负责维护本节点的资源情况和任务管理

    定期向 ResourceManager 汇报资源使用情况，接收 application master 提交的task，以及 task 的启动和关闭请求

ApplicationMaster：用户提交的每个program都会对应一个ApplicationMaster，主要负责监控应用，任务容错（重启失败的task）等。
    
    同时与ResourceManager 与 NodeManager 交互，向ResourceManager 申请资源，请求NodeMnanager 启动或关闭task

Container:容器是资源调度的单位，它是内存、cpu、磁盘、和IO的集合。
    
    Application Master会给task分配Container，task只能只用分配给它的Container的资源。

### YARN 运行流程

    1、client 向RM 发起请求，RM 为 提交的作业生产JOB ID，此时 JOB 状态为：NEW
    2、client 继续向 RM 提交 JOB 的详细信息，JOB 状态为：SUBMIT
    3、RM 检查要运行 AppLication Master 的 queue 是否有足够的资源，此时 JOB 状态为：accept
    4、AM 启动成功后，与RM 申请运行程序的资源，并检查状态
    5、如果JOB按照预期完成。此时，JOB的状态为FINISHED。如果运行过程中出现故障，此时，JOB的状态为FAILED。如果客户端主动kill掉作业，此时，JOB的状态为KILLED。


### YARN 内存配置与优化

基础知识：
    
    1、每个job 都会有一个 application Master，运行在container中
    2、每个container 都是一个独立的jvm进程 用于运行task

container 内存、cpu配置：
    
    1、yarn.scheduler.minimum-allocation-mb   默认：1024  container 最小分配内存
    1、yarn.scheduler.maximum-allocation-mb   默认：1024  container 最大分配内存
    1、yarn.scheduler.minimum-allocation-vcores   默认：1  container 最小分配核数
    1、yarn.scheduler.maximum-allocation-mb   默认：4  container 最大分配核数

当NodeManager 内存不够时，会使用虚拟内存，不过，虚拟内存，最多为内存的2.1倍