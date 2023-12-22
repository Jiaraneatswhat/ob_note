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
- 常用的算子
```scala
// map

```