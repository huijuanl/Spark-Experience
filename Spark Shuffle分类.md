Spark 2.0 
--
Spark 2.0之前有Hash Based Shuffle，Spark 2.0之后Spark只有Sort Based Shuffle

三种ShuffleWriter
--

基类 ： **ShuffleWriter**
* BypassMergeSortShuffleWriter
* UnsafeShuffleWriter
* SortShuffleWriter


|BypassMergeSortShuffleWriter| UnsafeShuffleWriter| SortShuffleWriter
| :-----------:|:-----------:| :-----:|
| 和Hash Shuffle实现基本相同，区别在于map task输出会汇总为一个文件     | tungsten-sort，ShuffleExternalSorter使用Java Unsafe直接操作内存，避免Java对象多余的开销和GC 延迟，效率高 | 和Hash Shuffle的主要不同在于，map端支持Partition级别的sort，map task输出会汇总为一个文件 |

* BypassMergeSortShuffleWriter	和Hash Shuffle实现基本相同，区别在于map task输出会汇总为一个文件
* UnsafeShuffleWriter	tungsten-sort，ShuffleExternalSorter使用Java Unsafe直接操作内存，避免Java对象多余的开销和GC 延迟，效率高
* SortShuffleWriter	Sort Shuffle，和Hash Shuffle的主要不同在于，map端支持Partition级别的sort，map task输出会汇总为一个文件


运行时三种ShuffleWriter实现的选择
--
由SortShuffleManager类中的registerShuffle方法决定调用哪一个ShuffleWriter类：

```
 override def registerShuffle[K, V, C](
      shuffleId: Int,
      numMaps: Int,
      dependency: ShuffleDependency[K, V, C]): ShuffleHandle = {
    if (SortShuffleWriter.shouldBypassMergeSort(conf, dependency)) {
      // If there are fewer than spark.shuffle.sort.bypassMergeThreshold partitions and we don't
      // need map-side aggregation, then write numPartitions files directly and just concatenate
      // them at the end. This avoids doing serialization and deserialization twice to merge
      // together the spilled files, which would happen with the normal code path. The downside is
      // having multiple files open at a time and thus more memory allocated to buffers.
      
      // 对应BypassMergeSortShuffleWriter
      // having multiple files open at a time and thus more memory allocated to buffers.
      new BypassMergeSortShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else if (SortShuffleManager.canUseSerializedShuffle(dependency)) {
      // Otherwise, try to buffer map outputs in a serialized form, since this is more efficient:
      // 对应UnSafeShuffleWriter
      new SerializedShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else {
      // Otherwise, buffer map outputs in a deserialized form:
      
      // 对应SortShuffleWriter
      new BaseShuffleHandle(shuffleId, numMaps, dependency)
    }
  }
```


ShuffleExternalSorter
--
ShuffleExternalSorter发生在shuffle write阶段。

* 重要参数

```
   /**
   * Force this sorter to spill when there are this many elements in memory.
   */
  private final int numElementsForSpillThreshold;
  ```
设置了参数numElementsForSpillThreshold，可以通过spark.shuffle.spill.numElementsForceSpillThreshold来设置，默认是Integer.MAX_VALUE。

UnsafeShuffleWriter类会调用ShuffleExternalSorter


```
19/04/22 14:09:28 INFO ShuffleExternalSorter: Spilling data because number of spilledRecords crossed the threshold 10000
19/04/22 14:09:28 INFO ShuffleExternalSorter: Thread 100 spilling sort data of 64.3 MB to disk (647  times so far)
19/04/22 14:09:29 INFO ShuffleExternalSorter: Spilling data because number of spilledRecords crossed the threshold 10000
19/04/22 14:09:29 INFO ShuffleExternalSorter: Thread 100 spilling sort data of 64.3 MB to disk (648  times so far)
```

ExternalSorter
--
ExternalSorter发生在shuffle read阶段，一个reduce task对应一个ExternalSorter。

SortShuffleWriter会调用ExternalSorter。
SortShuffleWriter和UnsafeShuffleWriter都继承自ShuffleWriter。


