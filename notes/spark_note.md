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

```
