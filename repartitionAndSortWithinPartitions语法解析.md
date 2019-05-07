在Spark源代码里，很多RDD算子总是有withScope ，而从源代码里，我们看到withScope的实现是RDDOperationScope.withScope[U](sc)(body)

```
private[spark] def withScope[U](body: => U): U = RDDOperationScope.withScope[U](sc)(body)
```

repartitionAndSortWithinPartitions算子源码：

```
def repartitionAndSortWithinPartitions(partitioner: Partitioner): RDD[(K, V)] = self.withScope {
    new ShuffledRDD[K, V, V](self, partitioner).setKeyOrdering(ordering)
  }
```

self.withScope功能：用来做DAG可视化的（DAG visualization on SparkUI）

sparkUI中能展示更多的信息。所以把所有创建的RDD的方法都包裹起来，同时用RDDOperationScope 记录 RDD 的操作历史和关联，就能达成目标。
