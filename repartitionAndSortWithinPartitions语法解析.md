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

* 为什么要进行隐式转换？
通过隐式转换，程序员可以在编写Scala程序时故意漏掉一些信息，让编译器去尝试在编译期间自动推导出这些信息，
这种特性可以极大的减少代码量，忽略那些冗长过于细节的代码

* 使用方式
1.将方法或变量标记为implicit
方法标记为implicit:

```
implicit def 函数名(参数名：参数类型)={...}
```

如

```
scala> implicit def intToString(x : Int) = x.toString
intToString: (x: Int)String
 
scala> foo(10)
10
```

变量标记为implicit:

```
implicit 变量名 类型名
```
如
```
implicit name : String
```


2.将方法的参数列表标记为implicit
如:
```
def person(implicit name : String) = name
```
若直接调用：
```
scala> person
<console>:26: error: could not find implicit value for parameter name: String
       person
       ^
```
报错！编译器说无法为参数name找到一个隐式值

定义一个隐式值后再调用person方法

```
scala> implicit val p = "mobin"
p: String = mobin

scala> person
res0: String = mobin
```
因为将p变量标记为implicit，所以编译器会在方法省略隐式参数的情况下去搜索作用域内的隐式值作为缺少参数。
但是如果此时你又在REPL中定义一个隐式变量，再次调用方法时就会报错

```
scala> implicit val pp = "mobin3"
pp: String = mobin3

scala> person
<console>:30: error: ambiguous implicit values:
 both value p of type => String
 and value pp of type => String
 match expected type String
       person
```
匹配失败，所以隐式转换必须满足无歧义规则，在声明隐式参数的类型是最好使用特别的或自定义的数据类型，不要使用Int,String这些常用类型，避免碰巧匹配

3.将类标记为implicit


Scala支持两种形式的隐式转换：
隐式值：用于给方法提供参数
隐式视图：用于类型间转换或针对某类型的方法能调用成功


疑问：

RangePartitioner.readObject中的
```
 val ser = sfactory.newInstance()
        Utils.deserializeViaNestedStream(in, ser) { ds =>
          implicit val classTag = ds.readObject[ClassTag[Array[K]]]()
          rangeBounds = ds.readObject[Array[K]]()
```
what does "implicit val classTag = ds.readObject[ClassTag[Array[K]]]()" mean ?
https://www.cnblogs.com/MOBIN/p/5351900.html
https://fangjian0423.github.io/2015/12/20/scala-implicit/
