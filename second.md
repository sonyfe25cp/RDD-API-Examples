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
对两个key-value的RDD进行右外连接。注意，两个RDD中的key必须可比较才能确保结果正确。
**变量列表**

	def rightOuterJoin[W](other: RDD[(K, W)]): RDD[(K, (Option[V], W))]
	def rightOuterJoin[W](other: RDD[(K, W)], numPartitions: Int): RDD[(K, (Option[V], W))]
	def rightOuterJoin[W](other: RDD[(K, W)], partitioner: Partitioner): RDD[(K, (Option[V], W))]
	
**Example**
```scala
val a = sc.parallelize(List("dog", "salmon", "salmon", "rat", "elephant"), 3)
val b = a.keyBy(_.length)
val c = sc.parallelize(List("dog","cat","gnu","salmon","rabbit","turkey","wolf","bear","bee"), 3)
val d = c.keyBy(_.length)
b.rightOuterJoin(d).collect

res2: Array[(Int, (Option[String], String))] = Array((6,(Some(salmon),salmon)), (6,(Some(salmon),rabbit)), (6,(Some(salmon),turkey)), (6,(Some(salmon),salmon)), (6,(Some(salmon),rabbit)), (6,(Some(salmon),turkey)), (3,(Some(dog),dog)), (3,(Some(dog),cat)), (3,(Some(dog),gnu)), (3,(Some(dog),bee)), (3,(Some(rat),dog)), (3,(Some(rat),cat)), (3,(Some(rat),gnu)), (3,(Some(rat),bee)), (4,(None,wolf)), (4,(None,bear)))
```

##sample

从一个RDD中随机选择一个数据项并在一个新的RDD中返回。

**变量列表**

	def sample(withReplacement: Boolean, fraction: Double, seed: Int): RDD[T]
	
**Example**

```scala
val a = sc.parallelize(1 to 10000, 3)
a.sample(false, 0.1, 0).count
res24: Long = 960

a.sample(true, 0.3, 0).count
res25: Long = 2888

a.sample(true, 0.3, 13).count
res26: Long = 2985
```

##sampleByKey[Pair]

从key-value类型RDD中根据key来抽样样本到最终的RDD里。

**变量列表**

	def sampleByKey(withReplacement: Boolean, fractions: Map[K, Double], seed: Long = Utils.random.nextLong): RDD[(K, V)]

**Example**

```scala
val randRDD = sc.parallelize(List( (7,"cat"), (6, "mouse"),(7, "cup"), (6, "book"), (7, "tv"), (6, "screen"), (7, "heater")))
val sampleMap = List((7, 0.4), (6, 0.6)).toMap
randRDD.sampleByKey(false, sampleMap,42).collect

res6: Array[(Int, String)] = Array((7,cat), (6,mouse), (6,book), (6,screen), (7,heater))
```

##sampleByKeyExact[Pair, experimental]

这是一个实验性质的API，不对其做介绍。

**变量列表**

	def sampleByKeyExact(withReplacement: Boolean, fractions: Map[K, Double], seed: Long = Utils.random.nextLong): RDD[(K, V)]
	

##saveAsHadoopFile [Pair], saveAsHadoopDataset[Pair], saveAsNewAPIHadoopFile[Pair]

根据用户指定的Hadoop导出格式将RDD导出

**变量列表**

	def saveAsHadoopDataset(conf: JobConf)
	def saveAsHadoopFile[F <: OutputFormat[K, V]](path: String)(implicit fm: ClassTag[F])
	def saveAsHadoopFile[F <: OutputFormat[K, V]](path: String, codec: Class[_ <: CompressionCodec]) (implicit fm: ClassTag[F])
	def saveAsHadoopFile(path: String, keyClass: Class[_], valueClass: Class[_], outputFormatClass: Class[_ <: OutputFormat[_, _]], codec: Class[_ <: CompressionCodec])
	def saveAsHadoopFile(path: String, keyClass: Class[_], valueClass: Class[_], outputFormatClass: Class[_ <: OutputFormat[_, _]], conf: JobConf = new JobConf(self.context.hadoopConfiguration), codec: Option[Class[_ <: CompressionCodec]] = None)
	def saveAsNewAPIHadoopFile[F <: NewOutputFormat[K, V]](path: String)(implicit fm: ClassTag[F])
	def saveAsNewAPIHadoopFile(path: String, keyClass: Class[_], valueClass: Class[_], outputFormatClass: Class[_ <: NewOutputFormat[_, _]], conf: Configuration = self.context.hadoopConfiguration)

##saveAsObjectFile

保存RDD到二进制格式。

**变量列表**

	def saveAsObjectFile(path: String)
	
**Example**
```scala
val x = sc.parallelize(1 to 100, 3)
x.saveAsObjectFile("objFile")
val y = sc.objectFile[Array[Int]]("objFile")
y.collect
res52: Array[Int] = Array(67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33)
```

##saveAsSequenceFile[SeqFile]

将RDD导出为Hadoop序列文件。

**变量列表**

	def saveAsSequenceFile(path: String, codec: Option[Class[_ <: CompressionCodec]] = None)
	
**Example**
```scala
val v = sc.parallelize(Array(("owl",3), ("gnu",4), ("dog",1), ("cat",2), ("ant",5)), 2)
v.saveAsSequenceFile("hd_seq_file")
14/04/19 05:45:43 INFO FileOutputCommitter: Saved output of task 'attempt_201404190545_0000_m_000001_191' to file:/home/cloudera/hd_seq_file

[cloudera@localhost ~]$ ll ~/hd_seq_file
total 8
-rwxr-xr-x 1 cloudera cloudera 117 Apr 19 05:45 part-00000
-rwxr-xr-x 1 cloudera cloudera 133 Apr 19 05:45 part-00001
-rwxr-xr-x 1 cloudera cloudera   0 Apr 19 05:45 _SUCCESS
```

##saveAsTextFile

保存RDD为文本文件，一次一行。

**变量列表**

	def saveAsTextFile(path: String)
	def saveAsTextFile(path: String, codec: Class[_ <: CompressionCodec])

**Example without compression**
```scala
val a = sc.parallelize(1 to 10000, 3)
a.saveAsTextFile("mydata_a")
14/04/03 21:11:36 INFO FileOutputCommitter: Saved output of task 'attempt_201404032111_0000_m_000002_71' to file:/home/cloudera/Documents/spark-0.9.0-incubating-bin-cdh4/bin/mydata_a


[cloudera@localhost ~]$ head -n 5 ~/Documents/spark-0.9.0-incubating-bin-cdh4/bin/mydata_a/part-00000
1
2
3
4
5

// Produces 3 output files since we have created the a RDD with 3 partitions
[cloudera@localhost ~]$ ll ~/Documents/spark-0.9.0-incubating-bin-cdh4/bin/mydata_a/
-rwxr-xr-x 1 cloudera cloudera 15558 Apr  3 21:11 part-00000
-rwxr-xr-x 1 cloudera cloudera 16665 Apr  3 21:11 part-00001
-rwxr-xr-x 1 cloudera cloudera 16671 Apr  3 21:11 part-00002
```
**Example with compression**
```scala
import org.apache.hadoop.io.compress.GzipCodec
a.saveAsTextFile("mydata_b", classOf[GzipCodec])

[cloudera@localhost ~]$ ll ~/Documents/spark-0.9.0-incubating-bin-cdh4/bin/mydata_b/
total 24
-rwxr-xr-x 1 cloudera cloudera 7276 Apr  3 21:29 part-00000.gz
-rwxr-xr-x 1 cloudera cloudera 6517 Apr  3 21:29 part-00001.gz
-rwxr-xr-x 1 cloudera cloudera 6525 Apr  3 21:29 part-00002.gz

val x = sc.textFile("mydata_b")
x.count
res2: Long = 10000
```

**Example writing into HDFS**
```scala
val x = sc.parallelize(List(1,2,3,4,5,6,6,7,9,8,10,21), 3)
x.saveAsTextFile("hdfs://localhost:8020/user/cloudera/test");

val sp = sc.textFile("hdfs://localhost:8020/user/cloudera/sp_data")
sp.flatMap(_.split(" ")).saveAsTextFile("hdfs://localhost:8020/user/cloudera/sp_x")
```

##stats[Double]

计算RDD中所有值的平均值、方差、标准差。

**变量列表**

	def stats(): StatCounter

**Example**
```scala
val x = sc.parallelize(List(1.0, 2.0, 3.0, 5.0, 20.0, 19.02, 19.29, 11.09, 21.0), 2)
x.stats
res16: org.apache.spark.util.StatCounter = (count: 9, mean: 11.266667, stdev: 8.126859)
```
##sortBy
本函数用于排序输入的RDD数据，并存储在一个新的RDD中。第一个参数要求必须指定一个函数用于映射输入数据中用于排序的key。第二个参数可选，用于指定是升序还是降序。

**变量列表**

	def sortBy[K](f: (T) ⇒ K, ascending: Boolean = true, numPartitions: Int = this.partitions.size)(implicit ord: Ordering[K], ctag: ClassTag[K]): RDD[T]
	
**Example**
```scala
val y = sc.parallelize(Array(5, 7, 1, 3, 2, 1))
y.sortBy(c => c, true).collect
res101: Array[Int] = Array(1, 1, 2, 3, 5, 7)

y.sortBy(c => c, false).collect
res102: Array[Int] = Array(7, 5, 3, 2, 1, 1)

val z = sc.parallelize(Array(("H", 10), ("A", 26), ("Z", 1), ("L", 5)))
z.sortBy(c => c._1, true).collect
res109: Array[(String, Int)] = Array((A,26), (H,10), (L,5), (Z,1))

z.sortBy(c => c._2, true).collect
res108: Array[(String, Int)] = Array((Z,1), (L,5), (H,10), (A,26))
```

##sortByKey[Ordered]

本函数用于将输入的RDD数据排序并存储在新的RDD。输出的RDD是一个被shuffle过的RDD，因为这些数据是由一个reduce函数输出的。这个程序的实现非常巧妙。首先，*它在使用一个范围分区器将数据分区*，然后，它采用标准的排序策略对每个分区独立的用`mapPartitions`进行排序。

**变量列表**

	def sortByKey(ascending: Boolean = true, numPartitions: Int = self.partitions.size): RDD[P]

**Example**
```scala
val a = sc.parallelize(List("dog", "cat", "owl", "gnu", "ant"), 2)
val b = sc.parallelize(1 to a.count.toInt, 2)
val c = a.zip(b)
c.sortByKey(true).collect
res74: Array[(String, Int)] = Array((ant,5), (cat,2), (dog,1), (gnu,4), (owl,3))
c.sortByKey(false).collect
res75: Array[(String, Int)] = Array((owl,3), (gnu,4), (dog,1), (cat,2), (ant,5))

val a = sc.parallelize(1 to 100, 5)
val b = a.cartesian(a)
val c = sc.parallelize(b.takeSample(true, 5, 13), 2)
val d = c.sortByKey(false)
res56: Array[(Int, Int)] = Array((96,9), (84,76), (59,59), (53,65), (52,4))
```

##stdev[Double], sampleStdev[Double]

调用`stats`并抽取`stdev-component`或者纠正过的`sampleStdev-component`。

**变量列表**

	def stdev(): Double
	def sampleStdev(): Double
	
**Example**
```scala
val d = sc.parallelize(List(0.0, 0.0, 0.0), 3)
d.stdev
res10: Double = 0.0
d.sampleStdev
res11: Double = 0.0

val d = sc.parallelize(List(0.0, 1.0), 3)
d.stdev
d.sampleStdev
res18: Double = 0.5
res19: Double = 0.7071067811865476

val d = sc.parallelize(List(0.0, 0.0, 1.0), 3)
d.stdev
res14: Double = 0.4714045207910317
d.sampleStdev
res15: Double = 0.5773502691896257
```

##subtract

标准的减法操作，A-B。

**变量列表**

	def subtract(other: RDD[T]): RDD[T]
	def subtract(other: RDD[T], numPartitions: Int): RDD[T]
	def subtract(other: RDD[T], p: Partitioner): RDD[T]

**Example**
```scala
val a = sc.parallelize(1 to 9, 3)
val b = sc.parallelize(1 to 3, 3)
val c = a.subtract(b)
c.collect
res3: Array[Int] = Array(6, 9, 4, 7, 5, 8)
```
##subtractByKey[Pair]

跟`subtract`类似，但是并不是提供一个函数，将会根据每组的的key来从第一个RDD中删除

**变量列表**
	
	def subtractByKey[W: ClassTag](other: RDD[(K, W)]): RDD[(K, V)]
	def subtractByKey[W: ClassTag](other: RDD[(K, W)], numPartitions: Int): RDD[(K, V)]
	def subtractByKey[W: ClassTag](other: RDD[(K, W)], p: Partitioner): RDD[(K, V)]

**Example**
```scala
val a = sc.parallelize(List("dog", "tiger", "lion", "cat", "spider", "eagle"), 2)
val b = a.keyBy(_.length)
val c = sc.parallelize(List("ant", "falcon", "squid"), 2)
val d = c.keyBy(_.length)
b.subtractByKey(d).collect
res15: Array[(Int, String)] = Array((4,lion))
```

##sum[Double], sumApprox[Double]

集散RDD中所有值的和。`approx`版本在某些场景下可以计算的很快，但是它是以损失精度为代价的。

**变量列表**

	def sum(): Double
	def sumApprox(timeout: Long, confidence: Double = 0.95): PartialResult[BoundedDouble]

**Example**

```scala
val x = sc.parallelize(List(1.0, 2.0, 3.0, 5.0, 20.0, 19.02, 19.29, 11.09, 21.0), 2)
x.sum
res17: Double = 101.39999999999999
```

##take

从RDD哄抽取前`n`个元素，并返回一个数组。（看起来这个很容易实现，但是在Spark中确实是一个很有难度的实现，因为不同的元素可能在不同的分区中）

**变量列表**

	def take(num: Int): Array[T]

**Example**

```scala
val b = sc.parallelize(List("dog", "cat", "ape", "salmon", "gnu"), 2)
b.take(2)
res18: Array[String] = Array(dog, cat)

val b = sc.parallelize(1 to 10000, 5000)
b.take(100)
res6: Array[Int] = Array(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100)
```
##takeOrdered

采用RDD内部的排序方法进行排序，并返回前`n`个元素作为一个数组。

**变量列表**

	def takeOrdered(num: Int)(implicit ord: Ordering[T]): Array[T]

**Example**

```scala
val b = sc.parallelize(List("dog", "cat", "ape", "salmon", "gnu"), 2)
b.takeOrdered(2)
res19: Array[String] = Array(ape, cat)
```

##takeSample

与`sample`有着以下几点的不同：

* 它可以返回一个具体数目的样本（它的第二个参数来决定）。
* 它会返回一个数组而不是RDD。
* 它返回的元素是随机排序的。

**变量列表**

	def takeSample(withReplacement: Boolean, num: Int, seed: Int): Array[T]

**Example**

```scala
val x = sc.parallelize(1 to 1000, 3)
x.takeSample(true, 100, 1)
res3: Array[Int] = Array(339, 718, 810, 105, 71, 268, 333, 360, 341, 300, 68, 848, 431, 449, 773, 172, 802, 339, 431, 285, 937, 301, 167, 69, 330, 864, 40, 645, 65, 349, 613, 468, 982, 314, 160, 675, 232, 794, 577, 571, 805, 317, 136, 860, 522, 45, 628, 178, 321, 482, 657, 114, 332, 728, 901, 290, 175, 876, 227, 130, 863, 773, 559, 301, 694, 460, 839, 952, 664, 851, 260, 729, 823, 880, 792, 964, 614, 821, 683, 364, 80, 875, 813, 951, 663, 344, 546, 918, 436, 451, 397, 670, 756, 512, 391, 70, 213, 896, 123, 858)
```

##toDebugString

返回一个字符串，其中包含关于RDD的debug信息和相关依赖。

**变量列表**

	def toDebugString: String

**Example**

```scala
val a = sc.parallelize(1 to 9, 3)
val b = sc.parallelize(1 to 3, 3)
val c = a.subtract(b)
c.toDebugString
res6: String = 
MappedRDD[15] at subtract at <console>:16 (3 partitions)
  SubtractedRDD[14] at subtract at <console>:16 (3 partitions)
    MappedRDD[12] at subtract at <console>:16 (3 partitions)
      ParallelCollectionRDD[10] at parallelize at <console>:12 (3 partitions)
    MappedRDD[13] at subtract at <console>:16 (3 partitions)
      ParallelCollectionRDD[11] at parallelize at <console>:12 (3 partitions)
```
##toJavaRDD

把RDD包装成JavaRDD格式并返回。

**变量列表**
	
	def toJavaRDD(): JavaRDD[T]
	
**Example**

```scala
val c = sc.parallelize(List("Gnu", "Cat", "Rat", "Dog"), 2)
c.toJavaRDD
res3: org.apache.spark.api.java.JavaRDD[String] = ParallelCollectionRDD[6] at parallelize at <console>:12
```

##toLocalIterator

转换一个RDD到scala迭代器。

**变量列表**

	def toLocalIterator: Iterator[T]
	
**Example**

```scala
val z = sc.parallelize(List(1,2,3,4,5,6), 2)
val iter = z.toLocalIterator
iter.next
res51: Int = 1
iter.next
res52: Int = 2
```

##top

利用默认的`$T$`排序来决定前`$k$`个值并以数组形式返回。

**变量列表**

	def top(num: Int)(implicit ord: Ordering[T]): Array[T]
	
**Example**

```scala
val c = sc.parallelize(Array(6, 9, 4, 7, 5, 8), 2)
c.top(2)
res28: Array[Int] = Array(9, 8)
```

##toString

返回一个人类可读的RDD文本描述。

**变量列表**

	override def toString: String
	
**Example**

```scala
val z = sc.parallelize(List(1,2,3,4,5,6), 2)
z.toString
res61: String = ParallelCollectionRDD[80] at parallelize at <console>:21

val randRDD = sc.parallelize(List( (7,"cat"), (6, "mouse"),(7, "cup"), (6, "book"), (7, "tv"), (6, "screen"), (7, "heater")))
val sortedRDD = randRDD.sortByKey()
sortedRDD.toString
res64: String = ShuffledRDD[88] at sortByKey at <console>:23
```

##treeAggregate

计算相同的对象作为一个聚集，除非它聚集RDD的元素以一个多层树的模式。另一个不同是，它不使用第二个`reduce`函数的初始化值。默认使用一个深度为2的树，可以通过调整`depth`参数调整。

**变量列表**

	def treeAggregate[U](zeroValue: U)(seqOp: (U, T) ⇒ U, combOp: (U, U) ⇒ U, depth: Int = 2)(implicit arg0: ClassTag[U]): U
	
**Example**

```scala
val z = sc.parallelize(List(1,2,3,4,5,6), 2)

// lets first print out the contents of the RDD with partition labels
def myfunc(index: Int, iter: Iterator[(Int)]) : Iterator[String] = {
  iter.toList.map(x => "[partID:" +  index + ", val: " + x + "]").iterator
}

z.mapPartitionsWithIndex(myfunc).collect
res28: Array[String] = Array([partID:0, val: 1], [partID:0, val: 2], [partID:0, val: 3], [partID:1, val: 4], [partID:1, val: 5], [partID:1, val: 6])

z.treeAggregate(0)(math.max(_, _), _ + _)
res40: Int = 9

// Note unlike normal aggregrate. Tree aggregate does not apply the initial value for the second reduce
// This example returns 11 since the initial value is 5
// reduce of partition 0 will be max(5, 1, 2, 3) = 5
// reduce of partition 1 will be max(4, 5, 6) = 6
// final reduce across partitions will be 5 + 6 = 11
// note the final reduce does not include the initial value
z.treeAggregate(5)(math.max(_, _), _ + _)
res42: Int = 11
```

##treeReduce

跟`reduce`一样，但是它采用多层树模式对RDD进行reduce操作。

**变量列表**

	def  treeReduce(f: (T, T) ⇒ T, depth: Int = 2): T
	
**Example**

```scala
val z = sc.parallelize(List(1,2,3,4,5,6), 2)
z.treeReduce(_+_)
res49: Int = 21
```

##union, ++

标准操作：A union B。

**变量列表**

	def ++(other: RDD[T]): RDD[T]
	def union(other: RDD[T]): RDD[T]

**Example**

```scala
val a = sc.parallelize(1 to 3, 1)
val b = sc.parallelize(5 to 7, 1)
(a ++ b).collect
res0: Array[Int] = Array(1, 2, 3, 5, 6, 7)
```

##unpersist

反持久化RDD（从硬盘和内存中删掉所有数据项）。这个RDD依然存在，如果它被一个运算使用，Spark将会自动基于依赖关系重新生成它。

**变量列表**

	def unpersist(blocking: Boolean = true): RDD[T]
	
**Example**

```scala
val y = sc.parallelize(1 to 10, 10)
val z = (y++y)
z.collect
z.unpersist(true)
14/04/19 03:04:57 INFO UnionRDD: Removing RDD 22 from persistence list
14/04/19 03:04:57 INFO BlockManager: Removing RDD 22
```

##values

从包含的元组中抽取数值，并返回到一个新的RDD中。

**变量列表**

	def values: RDD[V]
	
**Example**

```scala
val a = sc.parallelize(List("dog", "tiger", "lion", "cat", "panther", "eagle"), 2)
val b = a.map(x => (x.length, x))
b.values.collect
res3: Array[String] = Array(dog, tiger, lion, cat, panther, eagle)
```

##variance[Double], sampleVariance[Double]

调用`stats`并抽取`variance`元素或者纠正过的`sampleVariance`元素。

**变量列表**

	def variance(): Double
	def sampleVariance(): Double

**Example**

```scala
val a = sc.parallelize(List(9.1, 1.0, 1.2, 2.1, 1.3, 5.0, 2.0, 2.1, 7.4, 7.5, 7.6, 8.8, 10.0, 8.9, 5.5), 3)
a.variance
res70: Double = 10.605333333333332

val x = sc.parallelize(List(1.0, 2.0, 3.0, 5.0, 20.0, 19.02, 19.29, 11.09, 21.0), 2)
x.variance
res14: Double = 66.04584444444443

x.sampleVariance
res13: Double = 74.30157499999999
```

##zip

连接两个RDD，通过合并每个分区中第`i`个。返回的RDD中将包含双元素元组，它可以理解为是个key-value对，它们由`pairRDDFuncations`的扩展提供。

**变量列表**

	def zip[U: ClassTag](other: RDD[U]): RDD[(T, U)]

**Example**

```scala
val a = sc.parallelize(1 to 100, 3)
val b = sc.parallelize(101 to 200, 3)
a.zip(b).collect
res1: Array[(Int, Int)] = Array((1,101), (2,102), (3,103), (4,104), (5,105), (6,106), (7,107), (8,108), (9,109), (10,110), (11,111), (12,112), (13,113), (14,114), (15,115), (16,116), (17,117), (18,118), (19,119), (20,120), (21,121), (22,122), (23,123), (24,124), (25,125), (26,126), (27,127), (28,128), (29,129), (30,130), (31,131), (32,132), (33,133), (34,134), (35,135), (36,136), (37,137), (38,138), (39,139), (40,140), (41,141), (42,142), (43,143), (44,144), (45,145), (46,146), (47,147), (48,148), (49,149), (50,150), (51,151), (52,152), (53,153), (54,154), (55,155), (56,156), (57,157), (58,158), (59,159), (60,160), (61,161), (62,162), (63,163), (64,164), (65,165), (66,166), (67,167), (68,168), (69,169), (70,170), (71,171), (72,172), (73,173), (74,174), (75,175), (76,176), (77,177), (78,...

val a = sc.parallelize(1 to 100, 3)
val b = sc.parallelize(101 to 200, 3)
val c = sc.parallelize(201 to 300, 3)
a.zip(b).zip(c).map((x) => (x._1._1, x._1._2, x._2 )).collect
res12: Array[(Int, Int, Int)] = Array((1,101,201), (2,102,202), (3,103,203), (4,104,204), (5,105,205), (6,106,206), (7,107,207), (8,108,208), (9,109,209), (10,110,210), (11,111,211), (12,112,212), (13,113,213), (14,114,214), (15,115,215), (16,116,216), (17,117,217), (18,118,218), (19,119,219), (20,120,220), (21,121,221), (22,122,222), (23,123,223), (24,124,224), (25,125,225), (26,126,226), (27,127,227), (28,128,228), (29,129,229), (30,130,230), (31,131,231), (32,132,232), (33,133,233), (34,134,234), (35,135,235), (36,136,236), (37,137,237), (38,138,238), (39,139,239), (40,140,240), (41,141,241), (42,142,242), (43,143,243), (44,144,244), (45,145,245), (46,146,246), (47,147,247), (48,148,248), (49,149,249), (50,150,250), (51,151,251), (52,152,252), (53,153,253), (54,154,254), (55,155,255)...
```

##zipPartitions

类似于`zip`。但是它对压缩过程提供了更多操作。

**变量列表**

	def zipPartitions[B: ClassTag, V: ClassTag](rdd2: RDD[B])(f: (Iterator[T], Iterator[B]) => Iterator[V]): RDD[V]
	def zipPartitions[B: ClassTag, V: ClassTag](rdd2: RDD[B], preservesPartitioning: Boolean)(f: (Iterator[T], Iterator[B]) => Iterator[V]): RDD[V]
	def zipPartitions[B: ClassTag, C: ClassTag, V: ClassTag](rdd2: RDD[B], rdd3: RDD[C])(f: (Iterator[T], Iterator[B], Iterator[C]) => Iterator[V]): RDD[V]
	def zipPartitions[B: ClassTag, C: ClassTag, V: ClassTag](rdd2: RDD[B], rdd3: RDD[C], preservesPartitioning: Boolean)(f: (Iterator[T], Iterator[B], Iterator[C]) => Iterator[V]): RDD[V]
	def zipPartitions[B: ClassTag, C: ClassTag, D: ClassTag, V: ClassTag](rdd2: RDD[B], rdd3: RDD[C], rdd4: RDD[D])(f: (Iterator[T], Iterator[B], Iterator[C], Iterator[D]) => Iterator[V]): RDD[V]
	def zipPartitions[B: ClassTag, C: ClassTag, D: ClassTag, V: ClassTag](rdd2: RDD[B], rdd3: RDD[C], rdd4: RDD[D], preservesPartitioning: Boolean)(f: (Iterator[T], Iterator[B], Iterator[C], Iterator[D]) => Iterator[V]): RDD[V]

**Example**
```scala
val a = sc.parallelize(0 to 9, 3)
val b = sc.parallelize(10 to 19, 3)
val c = sc.parallelize(100 to 109, 3)
def myfunc(aiter: Iterator[Int], biter: Iterator[Int], citer: Iterator[Int]): Iterator[String] =
{
  var res = List[String]()
  while (aiter.hasNext && biter.hasNext && citer.hasNext)
  {
    val x = aiter.next + " " + biter.next + " " + citer.next
    res ::= x
  }
  res.iterator
}
a.zipPartitions(b, c)(myfunc).collect
res50: Array[String] = Array(2 12 102, 1 11 101, 0 10 100, 5 15 105, 4 14 104, 3 13 103, 9 19 109, 8 18 108, 7 17 107, 6 16 106)
```

##zipWithIndex

通过元素索引位置压缩RDD中元素。索引从0开始。如果这个RDD通过多个分区扩展，那么一个spark任务将开始执行这个操作。

**变量列表**

	def zipWithIndex(): RDD[(T, Long)]
	
**Example**
```scala
val z = sc.parallelize(Array("A", "B", "C", "D"))
val r = z.zipWithIndex
res110: Array[(String, Long)] = Array((A,0), (B,1), (C,2), (D,3))

val z = sc.parallelize(100 to 120, 5)
val r = z.zipWithIndex
r.collect
res11: Array[(Int, Long)] = Array((100,0), (101,1), (102,2), (103,3), (104,4), (105,5), (106,6), (107,7), (108,8), (109,9), (110,10), (111,11), (112,12), (113,13), (114,14), (115,15), (116,16), (117,17), (118,18), (119,19), (120,20))
```

##zipWithUniqueId

这个与`zipWithIndex`不同，因为只提供了一个为唯一ID到每个数据元素，但是这些ID可能与索引数目不一致。这个操作不会启动一个新的spark任务，即使这个RDD跨越多个分区。

对比下面例子和`zipWithIndex`第二个例子的结果，你就可以看出它们的差别。

**变量列表**

	def zipWithUniqueId(): RDD[(T, Long)]

**Example**

```scala
val z = sc.parallelize(100 to 120, 5)
val r = z.zipWithUniqueId
r.collect

res12: Array[(Int, Long)] = Array((100,0), (101,5), (102,10), (103,15), (104,1), (105,6), (106,11), (107,16), (108,2), (109,7), (110,12), (111,17), (112,3), (113,8), (114,13), (115,18), (116,4), (117,9), (118,14), (119,19), (120,24))
```
































































