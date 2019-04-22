Spark 2.0 
--
Spark 2.0之前有Hash Based Shuffle，Spark 2.0之后Spark只有Sort Based Shuffle

三种ShuffleWriter
--

|BypassMergeSortShuffleWriter| UnsafeShuffleWriter| SortShuffleWriter
| :-----------:|:-----------:| :-----:|
| 和Hash Shuffle实现基本相同，区别在于map task输出会汇总为一个文件     | tungsten-sort，ShuffleExternalSorter使用Java Unsafe直接操作内存，避免Java对象多余的开销和GC 延迟，效率高 | 和Hash Shuffle的主要不同在于，map端支持Partition级别的sort，map task输出会汇总为一个文件 |

* BypassMergeSortShuffleWriter	和Hash Shuffle实现基本相同，区别在于map task输出会汇总为一个文件
* UnsafeShuffleWriter	tungsten-sort，ShuffleExternalSorter使用Java Unsafe直接操作内存，避免Java对象多余的开销和GC 延迟，效率高
* SortShuffleWriter	Sort Shuffle，和Hash Shuffle的主要不同在于，map端支持Partition级别的sort，map task输出会汇总为一个文件


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


