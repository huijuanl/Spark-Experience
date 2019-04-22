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

shuffle内存管理和消费模型
--
1，在 Spark 中，使用抽象类 MemoryConsumer 来表示需要使用内存的消费者。在这个类中定义了分配，释放以及 Spill 内存数据到磁盘的一些方法或者接口。具体的消费者可以继承 MemoryConsumer 从而实现具体的行为。 因此，在 Spark Task 执行过程中，会有各种类型不同，数量不一的具体消费者。如在 Spark Shuffle 中使用的 ExternalAppendOnlyMap, ExternalSorter 等等（具体后面会分析）。

2，MemoryConsumer 会将申请，释放相关内存的工作交由 TaskMemoryManager 来执行。当一个 Spark Task 被分配到 Executor 上运行时，会创建一个 TaskMemoryManager。在 TaskMemoryManager 执行分配内存之前，需要首先向 MemoryManager 进行申请，然后由 TaskMemoryManager 借助 MemoryAllocator 执行实际的内存分配。 

3，Executor 中的 MemoryManager 会统一管理内存的使用。由于每个 TaskMemoryManager 在执行实际的内存分配之前，会首先向 MemoryManager 提出申请。因此 MemoryManager 会对当前进程使用内存的情况有着全局的了解。

MemoryManager，TaskMemoryManager 和 MemoryConsumer 之前的对应关系，如下图。总体上，一个 MemoryManager 对应着至少一个 TaskMemoryManager （具体由 executor-core 参数指定），而一个 TaskMemoryManager 对应着多个 MemoryConsumer (具体由任务而定)。 

了解了以上内存消费的整体过程以后，有两个问题需要注意下：

* 1，当有多个 Task 同时在 Executor 上执行时， 将会有多个 TaskMemoryManager 共享 MemoryManager 管理的内存。那么 MemoryManager 是怎么分配的呢？答案是每个任务可以分配到的内存范围是 [1 / (2 * n), 1 / n]，其中 n 是正在运行的 Task 个数。因此，多个并发运行的 Task 会使得每个 Task 可以获得的内存变小。

* 2，前面提到，在 MemoryConsumer 中有 Spill 方法，当 MemoryConsumer 申请不到足够的内存时，可以 Spill 当前内存到磁盘，从而避免无节制的使用内存。但是，对于堆内内存的申请和释放实际是由 JVM 来管理的。因此，在统计堆内内存具体使用量时，考虑性能等各方面原因，Spark 目前采用的是抽样统计的方式来计算 MemoryConsumer 已经使用的内存，从而造成堆内内存的实际使用量不是特别准确。从而有可能因为不能及时 Spill 而导致 OOM。

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


参考链接
--
https://tech.youzan.com/spark_memory_1/
