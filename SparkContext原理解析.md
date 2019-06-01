SparkContext总结

* 一个SparkContext表示与一个Spark集群的连接
* 在Spark集群上，能创建RDDs，累加器，广播变量
* 每个jvm仅仅有只有一个SparkContext是可能活动的

SparkContext中做的事情
* 创建SparkEnv
* 创建Spark UI
* 创建任务调度器TaskSchedulerlmpl
* 任务调度调用start()方法
* COORDINATOR
