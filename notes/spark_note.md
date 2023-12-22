Resilient Distributed Datasets
# 复习
## .1 入门
- 端口号：
	- `Yarn`：8032    `web`: 8088
	- `Spark` 历史服务：18080
	- `Driver` (`Spark Shell`): 4040
- wordcout
```java
sc.textFile("src", 2)
	.flatMap(line -> line.split(","))
	.mapToPair(w -> (w, 1))
	.reduceByKey((x, y) -> x + y)
	.saveAsTextFile("dst")
```
## .2 SparkCore RDD
### .2.1 五大属性
- 一个分区列表
```scala
protected def getPartitions: Array[Partition]
```
- 分区计算函数
```scala
// 由子类实现计算分区
def compute(split: Partition, context: TaskContext): Iterator[T]
```
- RDD 之间的依赖
```scala
protected def getDependencies: Seq[Dependency[_]] = deps
```
- 分区器(可选):  `JavaPairRDD, RDD[(K, V)]` 才有分区器
```scala
// 子类重写决定分区
@transient val partitioner: Option[Partitioner] = None
```
- 计算的优先位置(可选)
```scala
protected def getPreferredLocations(split: Partition): Seq[String] = Nil
```
### .2.2 算子
#### .2.2.1 常用的算子
	- `map`
	- `flatMap`
	-  `mapValues`
	- `filter`
	- `foreach`
- 重分区算子：`repartition`
- 聚合算子：`sortByKey`，`reduceByKey`，`groupByKey`
- RDD 交互：`intersection`，`union`，`join`
- 行动算子：
	- `collect`(会将整个 `RDD` 的数据存到 `Driver` 的内存中，不建议使用)
	- `count`
	- `reduce`
- 有 `shuffle` 的算子
	- `groupByKey`
	- `sortBy`
	- `sortByKey` 
	- `repartition`
	- `reduceByKey`
#### .2.2.2 算子的比较
- `map` 和 `mapPartitions`
	- `map` 对 `RDD` 中的每一个元素进行操作，`mapPartition` 是对 `RDD` 中的每一个分区进行操作
- `foreach` 和 `foreachPartition`
	- `foreachPartition` 一般用于向数据库写入数据
	- 类似于 `Flink` 中的 `RickSinkFunction` 的 `open` 方法
	- `foreachPartition` 一个分区只会去创建一个连接，而 `foreach` 每个元素都需要创建一个连接
	- 为什么不将连接创建在 main 方法中？
		- 因为 `main` 方法中，与 `RDD` 无关的操作交给 `Driver` 执行，`RDD` 操作由 `Executor` 执行，`main` 方法中获取的连接不能序列化传给 `Executor`
- `groupByKey` 和 `reduceByKey`
	- `groupByKey` 只有分组的功能
	- `reduceByKey` 不但可以分组，也可以进行聚合操作
		- 会先进行局部聚合，效率要高于 `groupByKey` + 逻辑聚合操作
- aggregate 算子
```scala
/*
 * zeroValue: 每个分区 seqOp 累加的初始值以及最终不同分区组合结果 comOp 的初始值
 * seqOp: 用于累加分区内数据的算子
 * combOp: 用于组合不同分区累加结果的算子
 * 
def aggregate[U: ClassTag](zeroValue: U)(seqOp: (U, T) => U, combOp: (U, U) => U): U = withScope {...}
*/

val rdd = sc.makeRDD(Array("12", "234", "345", "4567"), 2)
rdd.aggregate("0")((a, b) => Math.max(a.length, b.length).toString,
				   (x, y) => x + y)
// "0" 和 “12” 比较，得到 "2" 和 "234" 比较得到 "3"
// 另一个分区的结果是 "4", 字符串最终拼接，加上初始值得到 "034"
// 考虑到并行度为2，有可能先产生 “4”，返回 “043”

rdd.aggregate("")((a, b) => Math.min(a.length, b.length).toString,
				   (x, y) => x + y)
// "" 和 “12” 比较，返回 “0 ”，“0” 和 “234” 比较返回 “1”
// 同理另一个分区返回 “1”, 最终结果 "11"
```
### .2.3 血缘关系
- 窄依赖：父 `RDD` 和子 `RDD` 的分区是一对一的关系
- 宽依赖：父 `RDD` 的一个分区会被多个子 `RDD` 分区所继承
### .2.4 任务切分
- 从后往前进行切分，从行动算子向前找宽依赖划分 `Stage`
- 提交时先提交父阶段
- `Application` 指一个 `Spark` 应用程序
	- 一个 `Application` 由多个 `Job` 组成
	- 一个 `Job` 由多个 `Task` 组成
	- `Application `启动时会创建一个 `SparkContext` 作为入口点
- `Job` 数与行动算子有关系
	- 一般情况下相等
	- 如果配置了 `Checkpoint`，会启动一个新 `Job`，此时个数不相等
- `Job` 可以分为多个 `Stage`：`Stage个数  = 宽依赖数 + 1`
- `Stage` 可以分为多个 `Task`，`Task` 个数取决于 `RDD` 的分区数
### .2.5 分区器
- `HashPartitioner`
- `RangePartitioner`
- 自定义分区器
### .2.6 持久化
- cache 和 checkpoint
	- 存储位置不同：`cache` 存储在内存中，`checkpoint` 存储在 HDFS 中
	- `cache` 不会切断血缘关系，`ck` 会切断
	- 计算到 `cache` 的位置时，会将执行中间结果进行缓存，后面会从缓存位置继续执行任务
	- 计算到 `ck` 的位置时，提交一个新任务从头执行到当前位置，保存在 `HDFS` 中
	- 使用场景
		- `RDD` 复用：减少计算次数
		- 一个很长的任务链中间的 `shuffle` 阶段后，`防止重复shuffle`
```java
rdd.cache()
rdd.checkPoint("hdfs://xxx")
```
### .2.7 共享变量
- 广播变量(共享读操作)
	- 当多个分区多个 Task 都要用到同一份数据时，为了避免数据的重复发送，选择广播变量的方式，会将广播变量发给每个节点，作为只读值处理
- 累加器(共享写操作)
	- 