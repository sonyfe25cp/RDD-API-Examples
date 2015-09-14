#先分开翻译，完事合并就行了

##map

应用一个转换函数到RDD中的每一个成员，并将结果返回成一个新的RDD

**变量清单**

* def map[U: ClassTag](f: T => U): RDD[U]

**Example**

```scala
val a = sc.parallelize(List("dog", "salmon", "salmon", "rat", "elephant"), 3)
val b = a.map(_.length)
val c = a.zip(b)
c.collect
res0: Array[(String, Int)] = Array((dog,3), (salmon,6), (salmon,6), (rat,3), (elephant,8))
```

##mapPartitions

这是一个特殊的`map`，对于每个分区只调用一次。各个分区的全部内容是可见的，内容通过输入参数(Iterator[T])以一个序列流的形式呈现。自定义函数必须返回另一个`Iterator[U]`。合并结果的迭代器自动被转换为一个新的RDD。请注意，由于我们选的分区，元组(3,4)和(6,7)从下面的结果中丢失了。

**变量列表**

* def mapPartitions[U: ClassTag](f: Iterator[T] => Iterator[U], preservesPartitioning: Boolean = false): RDD[U]

**Example 1**

```scala
val a = sc.parallelize(1 to 9, 3)
def myfunc[T](iter: Iterator[T]) : Iterator[(T, T)] = {
  var res = List[(T, T)]()
  var pre = iter.next
  while (iter.hasNext)
  {
    val cur = iter.next;
    res .::= (pre, cur)
    pre = cur;
  }
  res.iterator
}
a.mapPartitions(myfunc).collect
res0: Array[(Int, Int)] = Array((2,3), (1,2), (5,6), (4,5), (8,9), (7,8))
```
**Example 2**

```scala
val x = sc.parallelize(List(1, 2, 3, 4, 5, 6, 7, 8, 9,10), 3)
def myfunc(iter: Iterator[Int]) : Iterator[Int] = {
  var res = List[Int]()
  while (iter.hasNext) {
    val cur = iter.next;
    res = res ::: List.fill(scala.util.Random.nextInt(10))(cur)
  }
  res.iterator
}
x.mapPartitions(myfunc).collect
// some of the number are not outputted at all. This is because the random number generated for it is zero.
res8: Array[Int] = Array(1, 2, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3, 3, 3, 4, 4, 4, 4, 4, 4, 4, 5, 7, 7, 7, 9, 9, 10)
```
上面这个程序也可以用`flatMap`改写成如下：

**Example 2 using flatmap**

```scala
val x  = sc.parallelize(1 to 10, 3)
x.flatMap(List.fill(scala.util.Random.nextInt(10))(_)).collect

res1: Array[Int] = Array(1, 2, 3, 3, 3, 4, 4, 4, 4, 4, 4, 4, 4, 4, 5, 5, 6, 6, 6, 6, 6, 6, 6, 6, 7, 7, 7, 8, 8, 8, 8, 8, 8, 8, 8, 9, 9, 9, 9, 9, 10, 10, 10, 10, 10, 10, 10, 10)
```

##mapPartitionsWithContext(废弃且为开发者API)

类似于`mapPartitions`，但是可以读取mapper中处理状态信息。

**变量列表**

	def mapPartitionsWithContext[U: ClassTag](f: (TaskContext, Iterator[T]) => Iterator[U], preservesPartitioning: Boolean = false): RDD[U]
	
**Example**

```scala
val a = sc.parallelize(1 to 9, 3)
import org.apache.spark.TaskContext
def myfunc(tc: TaskContext, iter: Iterator[Int]) : Iterator[Int] = {
  tc.addOnCompleteCallback(() => println(
    "Partition: "     + tc.partitionId +
    ", AttemptID: "   + tc.attemptId ))
  
  iter.toList.filter(_ % 2 == 0).iterator
}
a.mapPartitionsWithContext(myfunc).collect

14/04/01 23:05:48 INFO SparkContext: Starting job: collect at <console>:20
...
14/04/01 23:05:48 INFO Executor: Running task ID 0
Partition: 0, AttemptID: 0, Interrupted: false
...
14/04/01 23:05:48 INFO Executor: Running task ID 1
14/04/01 23:05:48 INFO TaskSetManager: Finished TID 0 in 470 ms on localhost (progress: 0/3)
...
14/04/01 23:05:48 INFO Executor: Running task ID 2
14/04/01 23:05:48 INFO TaskSetManager: Finished TID 1 in 23 ms on localhost (progress: 1/3)
14/04/01 23:05:48 INFO DAGScheduler: Completed ResultTask(0, 1)

?
res0: Array[Int] = Array(2, 6, 4, 8)
```

##mapPartitionsWithIndex

类似于`mapPartitions`，但是它有两个参数。第一个参数是分区的索引号，第二个参数是一个迭代器，可以遍历这个分区内全部元素的迭代器。输出是一个迭代器，包含了应用任何一个转换函数之后的全部结果。

**变量列表**

	def mapPartitionsWithIndex[U: ClassTag](f: (Int, Iterator[T]) => Iterator[U], preservesPartitioning: Boolean = false): RDD[U]
	
**Example**

```scala
val x = sc.parallelize(List(1,2,3,4,5,6,7,8,9,10), 3)
def myfunc(index: Int, iter: Iterator[Int]) : Iterator[String] = {
  iter.toList.map(x => index + "," + x).iterator
}
x.mapPartitionsWithIndex(myfunc).collect()
res10: Array[String] = Array(0,1, 0,2, 0,3, 1,4, 1,5, 1,6, 2,7, 2,8, 2,9, 2,10)
```

##mapPartitionsWithSplit

这个方法已经在API中标记为废弃。请不要再使用这个方法。本文档不包含已经废弃的API。

**变量列表**

	def mapPartitionsWithSplit[U: ClassTag](f: (Int, Iterator[T]) => Iterator[U], preservesPartitioning: Boolean = false): RDD[U]
	
##mapValues[Pair]

对一个包含双元素元组的RDD，应用提供的函数来改变其中的值。然后它形成新的双元素元组（key和改变后的值），并将其存在新的RDD中。

**变量列表**

	def mapValues[U](f: V => U): RDD[(K, U)]
	
**Example**
```scala
val a = sc.parallelize(List("dog", "tiger", "lion", "cat", "panther", "eagle"), 2)
val b = a.map(x => (x.length, x))
b.mapValues("x" + _ + "x").collect
res5: Array[(Int, String)] = Array((3,xdogx), (5,xtigerx), (4,xlionx), (3,xcatx), (7,xpantherx), (5,xeaglex))
```

##mapWith（已废弃）

这是`map`的扩展版本。它有两个参数。第一个参数必须是`Int -> T`并且它被每个分区执行一次。它将会映射分区索引到已改变的`T`类型分区索引。这就是它对某些特别的针对每个分区的初始化操作友好的地方。例如，创建一个随机数生成对象。第二个函数必须符合`(U, T)->T`的形式。`T`是改变的分区的索引，`U`是RDD中的一个数据项。

**变量列表**

	def mapWith[A: ClassTag, U: ClassTag](constructA: Int => A, preservesPartitioning: Boolean = false)(f: (T, A) => U): RDD[U]
	
**Example**

```scala
// 生成9个小于1000的随机数
val x = sc.parallelize(1 to 9, 3)
x.mapWith(a => new scala.util.Random)((x, r) => r.nextInt(1000)).collect
res0: Array[Int] = Array(940, 51, 779, 742, 757, 982, 35, 800, 15)

val a = sc.parallelize(1 to 9, 3)
val b = a.mapWith("Index:" + _)((a, b) => ("Value:" + a, b))
b.collect
res0: Array[(String, String)] = Array((Value:1,Index:0), (Value:2,Index:0), (Value:3,Index:0), (Value:4,Index:1), (Value:5,Index:1), (Value:6,Index:1), (Value:7,Index:2), (Value:8,Index:2), (Value:9,Index)
```


##max
返回RDD中最大的元素。

**变量列表**

	def max()(implicit ord: Ordering[T]): T
	
**Example**

```scala
val y = sc.parallelize(10 to 30)
y.max
res75: Int = 30

val a = sc.parallelize(List((10, "dog"), (3, "tiger"), (9, "lion"), (18, "cat")))
a.min
res6: (Int, String) = (18,cat)
```
##mean[Double], meanApprox[Double]

调用`stats`并抽取平均数。这个函数的近似版本可以在某些场景下计算的更快。但是，以损失精度为代价。

**变量列表**

	def mean(): Double
	def meanApprox(timeout: Long, confidence: Double = 0.95): PartialResult[BoundedDouble]
	
**Example**

val a = sc.parallelize(List(9.1, 1.0, 1.2, 2.1, 1.3, 5.0, 2.0, 2.1, 7.4, 7.5, 7.6, 8.8, 10.0, 8.9, 5.5), 3)
a.mean
res0: Double = 5.3

##min
返回RDD中最小的元素。

**变量列表**

	def min()(implicit ord: Ordering[T]): T
	
**Example**

```scala
val y = sc.parallelize(10 to 30)
y.min
res75: Int = 10
val a = sc.parallelize(List((10, "dog"), (3, "tiger"), (9, "lion"), (8, "cat")))
a.min
res4: (Int, String) = (3,tiger)
```
##name, setName

使得一个RDD可以用自定义的名字。

**变量列表**

	@transient var name: String
	def setName(_name: String)

**Example**

```scala
val y = sc.parallelize(1 to 10, 10)
y.name
res13: String = null
y.setName("Fancy RDD Name")
y.name
res15: String = Fancy RDD Name
```

##partitionBy[Pair]

用RDD的keys重新分区为key-value类型RDD。分区的实现可以作为它的第一个参数传入。

**变量列表**

	def partitionBy(partitioner: Partitioner): RDD[(K, V)]
	
##partitioner

指定一个函数指针到默认的分区，将用于`groupBy`、`subtract`、`reduceByKey（来自PairedRDDFunctions）`等函数。

**变量列表**

	@transient val partitioner: Option[Partitioner]
	
##partitions
返回RDD相关的一个分区对象数组。

**变量列表**

	final def partitions: Array[Partition]

**Example**

```scala
val b = sc.parallelize(List("Gnu", "Cat", "Rat", "Dog", "Gnu", "Rat"), 2)
b.partitions
res48: Array[org.apache.spark.Partition] = Array(org.apache.spark.rdd.ParallelCollectionPartition@18aa, org.apache.spark.rdd.ParallelCollectionPartition@18ab)
```

##persist，cache

这两个函数用于调整RDD的存储级别。当释放内存时，Spark将根据存储级别标示符来决定哪些分区应该被保留。无参的方法`persist()`和`cache()`用来调整`persist(StorageLevel.MEMORY_ONLY)`。（注意：一旦存储级别改过一次，它就不能再被改变。）

**变量列表**

	def cache(): RDD[T]
	def persist(): RDD[T]
	def persist(newLevel: StorageLevel): RDD[T]

**Example**
```scala
val c = sc.parallelize(List("Gnu", "Cat", "Rat", "Dog", "Gnu", "Rat"), 2)
c.getStorageLevel
res0: org.apache.spark.storage.StorageLevel = StorageLevel(false, false, false, false, 1)
c.cache
c.getStorageLevel
res2: org.apache.spark.storage.StorageLevel = StorageLevel(false, true, false, true, 1)
```

##pipe

获取RDD每个分区的数据，并通过stdin发送给一个shell命令。这个命令的输出会被获取，并返回一个string值的RDD。

**变量列表**

	def pipe(command: String): RDD[String]
	def pipe(command: String, env: Map[String, String]): RDD[String]
	def pipe(command: Seq[String], env: Map[String, String] = Map(), printPipeContext: (String => Unit) => Unit = null, printRDDElement: (T, String => Unit) => Unit = null): RDD[String]

**Example**
```scala
val a = sc.parallelize(1 to 9, 3)
a.pipe("head -n 1").collect
res2: Array[String] = Array(1, 4, 7)
```

##randomSplit
根据一个权重数组随机将一个RDD切分到多个小的RDD中，这个数据中指定要分到小RDD中数据所占比例。每个小RDD中数据的个数只是近似得按照数组中的比例计算得到。下面的第二个例子展示了每个小RDD的大小并不精确满足权重数组的比例。一个随机数种子可以被指定。这个函数在机器学习中切分训练集和测试集非常有用。

**变量列表**

	def randomSplit(weights: Array[Double], seed: Long = Utils.random.nextLong): Array[RDD[T]]
	
**Example**

```scala
val y = sc.parallelize(1 to 10)
val splits = y.randomSplit(Array(0.6, 0.4), seed = 11L)
val training = splits(0)
val test = splits(1)
training.collect
res:85 Array[Int] = Array(1, 4, 5, 6, 8, 10)
test.collect
res86: Array[Int] = Array(2, 3, 7, 9)

val y = sc.parallelize(1 to 10)
val splits = y.randomSplit(Array(0.1, 0.3, 0.6))

val rdd1 = splits(0)
val rdd2 = splits(1)
val rdd3 = splits(2)

rdd1.collect
res87: Array[Int] = Array(4, 10)
rdd2.collect
res88: Array[Int] = Array(1, 3, 5, 8)
rdd3.collect
res91: Array[Int] = Array(2, 6, 7, 9)
```

##reduce

这个函数提供Spark中众所周知的`reduce`功能。需要注意的是，任何一个函数`f`，应该是可交换的用于产生可再现的结果。

**变量列表**

	def reduce(f: (T, T) => T): T
	
**Example**
```scala
val a = sc.parallelize(1 to 100, 3)
a.reduce(_ + _)
res41: Int = 5050
```

##reduceByKey[Pair], reduceByKeyLocally[Pair], reduceByKeyToDriver[Pair]

这个函数提供Spark中众所周知的`reduce`功能。需要注意的是，任何一个函数`f`，应该是可交换的用于产生可再现的结果。

**变量列表**

	def reduceByKey(func: (V, V) => V): RDD[(K, V)]
	def reduceByKey(func: (V, V) => V, numPartitions: Int): RDD[(K, V)]
	def reduceByKey(partitioner: Partitioner, func: (V, V) => V): RDD[(K, V)]
	def reduceByKeyLocally(func: (V, V) => V): Map[K, V]
	def reduceByKeyToDriver(func: (V, V) => V): Map[K, V]

**Example**

```scala
val a = sc.parallelize(List("dog", "cat", "owl", "gnu", "ant"), 2)
val b = a.map(x => (x.length, x))
b.reduceByKey(_ + _).collect
res86: Array[(Int, String)] = Array((3,dogcatowlgnuant))

val a = sc.parallelize(List("dog", "tiger", "lion", "cat", "panther", "eagle"), 2)
val b = a.map(x => (x.length, x))
b.reduceByKey(_ + _).collect
res87: Array[(Int, String)] = Array((4,lion), (3,dogcat), (7,panther), (5,tigereagle))
```

##repartition

这个函数可根据传入的参数`numPartitions`来改变分区数目。

**变量列表**
	
	def repartition(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T]


**Example**
```scala
val rdd = sc.parallelize(List(1, 2, 10, 4, 5, 2, 1, 1, 1), 3)
rdd.partitions.length
res2: Int = 3
val rdd2  = rdd.repartition(5)
rdd2.partitions.length
res6: Int = 5
```

##repartitionAndSortWithinPartitions

根据给定的`partitioner`对RDD重新分区，在每个新分区中按照它们的key进行排序。

**变量列表**

	def repartitionAndSortWithinPartitions(partitioner: Partitioner): RDD[(K, V)]

**Example**

```scala
// 首先我们做范围分区，不带排序
val randRDD = sc.parallelize(List( (2,"cat"), (6, "mouse"),(7, "cup"), (3, "book"), (4, "tv"), (1, "screen"), (5, "heater")), 3)
val rPartitioner = new org.apache.spark.RangePartitioner(3, randRDD)
val partitioned = randRDD.partitionBy(rPartitioner)
def myfunc(index: Int, iter: Iterator[(Int, String)]) : Iterator[String] = {
  iter.toList.map(x => "[partID:" +  index + ", val: " + x + "]").iterator
}
partitioned.mapPartitionsWithIndex(myfunc).collect

res0: Array[String] = Array([partID:0, val: (2,cat)], [partID:0, val: (3,book)], [partID:0, val: (1,screen)], [partID:1, val: (4,tv)], [partID:1, val: (5,heater)], [partID:2, val: (6,mouse)], [partID:2, val: (7,cup)])


//重新分区，带排序 
val partitioned = randRDD.repartitionAndSortWithinPartitions(rPartitioner)
def myfunc(index: Int, iter: Iterator[(Int, String)]) : Iterator[String] = {
  iter.toList.map(x => "[partID:" +  index + ", val: " + x + "]").iterator
}
partitioned.mapPartitionsWithIndex(myfunc).collect

res1: Array[String] = Array([partID:0, val: (1,screen)], [partID:0, val: (2,cat)], [partID:0, val: (3,book)], [partID:1, val: (4,tv)], [partID:1, val: (5,heater)], [partID:2, val: (6,mouse)], [partID:2, val: (7,cup)])
```

##rightOuterJoin[Pair]










































































