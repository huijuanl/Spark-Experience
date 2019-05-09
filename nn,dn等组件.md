* NameNode

* DataNode

* Driver

* applicationMaster


 1）用户程序创建SparkContext时，新创建的SparkContext实例会连接到Cluster Manager，Cluster Manager会根据用户提交时设置的CPU和内存信息为本次提交分配计算资源，启动Executor；

 2）Driver会将用户程序划分为不同的执行阶段，每个执行阶段由一组完全相同的Task组成，这些Task分别作用于待处理数据的不同分区；在阶段划分完成和Task创建后，Driver会向Executor发送Task；

 3）Executor在接收到Task后，会下载Task运行时依赖，在准备好Task的执行环境后开始执行Task，并将Task的运行状态汇报给Driver；

 4）Driver会根据收到的Task的运行状态来处理不同的状态更新，Task分为两种：一种是Shuffle Map Task，它实现数据的重新洗牌，洗牌的结果保存到Executor所在节点的文件系统中；另外一种是Result Task，它负责生成结果数据；

 5）Driver会不断调用Task，将Task发送到Executor执行，在所有的Task都正确执行或超过执行次数的限制仍然没有执行成功时停止；

* AM

* NodeManager


* Yarn Client 和 Yarn Cluster的区别：

对于yarn-client和yarn-cluster的唯一区别在于：

1. yarn-client的Driver运行在本地，而AppMaster运行在yarn的一个节点上，它们之间进行远程通信，AppMaster只负责资源申请和释放(当然还有DelegationToken的刷新)，然后等待Driver的完成；

2. yarn-cluster的Driver则运行在AppMaster所在的container里，Driver和AppMaster是同一个进程的两个不同线程，它们之间也会进行通信，AppMaster同样等待Driver的完成，从而释放资源。

AM和Driver

首先区分下AppMaster和Driver，任何一个yarn上运行的任务都必须有一个AppMaster，而任何一个Spark任务都会有一个Driver，Driver就是运行SparkContext(它会构建TaskScheduler和DAGScheduler)的进程，当然在Driver上你也可以做很多非Spark的事情，这些事情只会在Driver上面执行，而由SparkContext上牵引出来的代码则会由DAGScheduler分析，并形成Job和Stage交由TaskScheduler，再由TaskScheduler交由各Executor分布式执行。

所以Driver和AppMaster是两个完全不同的东西，Driver是控制Spark计算和任务资源的，而AppMaster是控制yarn app运行和任务资源的，只不过在Spark on Yarn上，这两者就出现了交叉，而在standalone模式下，资源则由Driver管理。在Spark on Yarn上，Driver会和AppMaster通信，资源的申请由AppMaster来完成，而任务的调度和执行则由Driver完成，Driver会直接跟Executor通信，让其执行具体的任务。

NameNode和DataNode都被设计成可以在普通商用计算机上运行。这些计算机通常运行的是GNU/Linux操作系统。
HDFS采用Java语言开发，因此任何支持Java的机器都可以部署NameNode和DataNode。一个典型的部署场景是集群中的一台机器运行一个NameNode实例，其他机器分别运行一个DataNode实例。当然，并不排除一台机器运行多个DataNode实例的情况。
集群中单一的NameNode的设计则大大简化了系统的架构。NameNode是所有HDFS元数据的管理者，用户数据永远不会经过NameNode。




参考链接：https://blog.csdn.net/Gamer_gyt/article/details/51758881
https://blog.csdn.net/u012050154/article/details/52484270 
https://blog.csdn.net/u012050154/article/details/52484270
https://www.zybuluo.com/sasaki/note/252413
https://www.jianshu.com/p/6b796a5c3e80
https://www.jianshu.com/p/b9ec3c2ff8dd

AM和Driver的区别：https://my.oschina.net/kavn/blog/1540548
