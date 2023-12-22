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
### .2 算子
#### .2.1 常用的算子
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
#### .2.2 算子的比较
- `map` 和 `mapPartitions`
	- `map` 对 `RDD` 中的每一个元素进行操作，`mapPartition` 是对 `RDD` 中的每一个分区进行操作
- `foreach` 和 `foreachPartition`
	- `foreachPartition` 一般用于向数据库写入数据
	- 类似于 `Flink` 中的 `RickSinkFunction` 的 `open` 方法
	- `foreachPartition` 一个分区只会去创建一个连接，而 `foreach` 每个元素都需要创建一个连接
	- 为什么不将连接创建在 main 方法中？
		- 因为 `main` 方法中，与 `RDD` 无关的操作交给 `Driver` 执行，`RDD` 操作由 `Executor` 执行，`main` 方法中获取的连接不能序列化传给 `Executor`
- `groupByKey` 和 `reduceByKey`
	- groupByKey 只有分组的功能
	- 