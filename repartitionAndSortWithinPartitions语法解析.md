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


repartitionAndSortWithinPartitions算子源码的使用还涉及到隐式转换，尤其在RDD到key声明为Object但是又要使用repartitionAndSortWithinPartitions时显得尤为重要

一个使用repartitionAndSortWithinPartitions及隐式转换的例子：
```
 val mapRdd: RDD[(Object, Object)] = inputRdd
      .mapPartitions { iter =>
        new MRMapPartitionClass("mapper", sparkConf, iter).getIterator()
      }

    val partitioner = new MRPartitionerClass(sparkConf, mapRdd.getNumPartitions).getPartitioner()

    implicit val caseInsensitiveOrdering = new Ordering[Object] {
      override def compare(a: Object, b: Object) = {
        val cmp = WritableComparator.get(
          a.getClass.asInstanceOf[Class[WritableComparable[_]]])
        cmp.compare(a, b)
      }
    }

    val shuffleRdd = mapRdd.repartitionAndSortWithinPartitions(
      partitioner)
```

隐式转换:

https://www.cnblogs.com/MOBIN/p/5351900.html
