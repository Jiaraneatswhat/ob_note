# 1. 提交 Job 流程
## 1.1 执行脚本
- spark-submit.sh
```shell
if [ -z "${SPARK_HOME}" ]; then  # -z 字符串长度为0
  source "$(dirname "$0")"/find-spark-home
fi

exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```
- spark-class.sh
```shell
if [ -z "${SPARK_HOME}" ]; then
  source "$(dirname "$0")"/find-spark-home # 执行 source
fi

. "${SPARK_HOME}"/bin/load-spark-env.sh # 导入 spark-env 的设置

# Find the java binary
if [ -n "${JAVA_HOME}" ]; then
  RUNNER="${JAVA_HOME}/bin/java" # 定义运行 java 的 runner
fi

# 调用org.apache.spark.launcher的main方法
# 被 spark-class 调用，负责调用其他类，对参数进行解析，生成执行命令，将命令返回给exec"${CMD[@]}”
build_command() {
  # java -cp(classpath) 指定jar包路径
  "$RUNNER" -Xmx128m $SPARK_LAUNCHER_OPTS -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main "$@"
  printf "%d\0" $? # 执行完成打印成功的状态码\0
}

# ${arr[@]}表示所有元素，${#arr[@]}表示数组长度
# 执行java -cp
CMD=("${CMD[@]:0:$LAST}")
exec "${CMD[@]}"
```
## 1.2 运行 SparkSubmit 解析参数
### 1.2.1 main()
```scala
def main(args: Array[String]): Unit = {
  val submit = new SparkSubmit()
  submit.doSubmit(args) // 创建SparkSubmit对象调用doSubmit()
}

def doSubmit(args: Array[String]): Unit = {
  // 创建一个SparkSubmitArguments对象解析参数
  val appArgs = parseArguments(args)
  appArgs.action match {
    // 匹配到SUBMIT，调用submit()
    case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog)
  }
}

private def submit(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {
  def doRunMain(): Unit = {
    if (args.proxyUser != null) {
      try {
        proxyUser.doAs(new PrivilegedExceptionAction[Unit]() {
          override def run(): Unit = {
            // 调用runMain()
            runMain(args, uninitLog)
          }
}
```
### 1.2.2 rumMain()
```scala
private def runMain(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {
  /*
   * 判断是本地/Yarn/Mesos等模式，部署模式等
   * clusterManager -> YARN
   * deployMode -> CLUSTER
   * childMainClass = YARN_CLUSTER_SUBMIT_CLASS
   * "org.apache.spark.deploy.yarn.YarnClusterApplication"
   * 
   * prepareSubmitEnvironment用于准备submit的环境，返回一个四元组:
   * childArgs: 子进程的参数
   * childClasspath: 子进程的类路径
   * sparkConf
   * childMainClass: 子进程的类 
   */
  val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)
  val loader = getSubmitClassLoader(sparkConf)
  // 添加需要的jar包
  for (jar <- childClasspath) {
    addJarToClasspath(jar, loader)
  }

  var mainClass: Class[_] = null

  try {
    mainClass = Utils.classForName(childMainClass)
  }

  val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
    // 继承了SparkApplication的话返回mainClass的对象
    mainClass.getConstructor().newInstance().asInstanceOf[SparkApplication]
  } else {
    // 否则创建JavaMainApplication的对象
    new JavaMainApplication(mainClass)
  }
  try {
    // 启动 SparkApplication
    app.start(childArgs.toArray, sparkConf)
  }
}
```
## 1.3 启动客户端： YarnClusterApplication
```scala
// YarnClusterApplication 继承了 SparkApplication，调用其 start 方法
// Client.scala中
private[spark] class YarnClusterApplication extends SparkApplication { 
  override def start(args: Array[String], conf: SparkConf): Unit = {  
    new Client(new ClientArguments(args), conf).run()  
  }  
}

// 创建一个Client对象
private[spark] class Client(
	val args: ClientArguments,  
    val sparkConf: SparkConf) extends Logging {
    // 创建 YarnClientImpl 对象
    private val yarnClient = YarnClient.createYarnClient
    // Driver 的内存，默认1g
    private val amMemory = if (isClusterMode) {  
  sparkConf.get(DRIVER_MEMORY).toInt}
  // executor 内存，默认1g
  private val executorMemory = sparkConf.get(EXECUTOR_MEMORY)
}

// 运行Client
def run(): Unit = {  
  this.appId = submitApplication()  
}
```
## 1.4 向 RM 提交任务
```scala
def submitApplication(): ApplicationId = {  
  var appId: ApplicationId = null  
  try {  
    launcherBackend.connect()  
    // 初始化 YarnClientImpl Service 的初始化
    yarnClient.init(hadoopConf)  
    yarnClient.start()  
  
    // Get a new application from our RM  
    val newApp = yarnClient.createApplication()  
    val newAppResponse = newApp.getNewApplicationResponse()  
    appId = newAppResponse.getApplicationId()  
  
    new CallerContext("CLIENT", sparkConf.get(APP_CALLER_CONTEXT),  
      Option(appId.toString)).setCurrentContext()  
  
    // Verify whether the cluster has enough resources for our AM  
    verifyClusterResources(newAppResponse)  
  
    // Set up the appropriate contexts to launch our AM  
    val containerContext = createContainerLaunchContext(newAppResponse)  
    val appContext = createApplicationSubmissionContext(newApp, containerContext)  
  
    // Finally, submit and monitor the application  
    logInfo(s"Submitting application $appId to ResourceManager")  
    yarnClient.submitApplication(appContext)  
    launcherBackend.setAppId(appId.toString)  
    reportLauncherState(SparkAppHandle.State.SUBMITTED)  
  
    appId  
  } catch {  
    case e: Throwable =>  
      if (appId != null) {  
        cleanupStagingDir(appId)  
      }  
      throw e  
  }  
}
```



























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
	- Driver 端定义的变量会发给每个 `Executor`，`Executor` 更新副本的值不会对 Driver 产生影响，需要将其注册成累加器
## .3 SparkSql
- `RDD`, `DataSet`, `DataFrame`
```scala
// DF 是特殊的 DS
type DataFrame = Dataset[Row]
// DS -> DF
ds.toDF // 返回的是 Dataset[Row]
// DS -> RDD
ds.rdd
lazy val rdd: RDD[T] = {  
  val objectType = exprEnc.deserializer.dataType  
  rddQueryExecution.toRdd.mapPartitions { rows =>  
    rows.map(_.get(0, objectType).asInstanceOf[T])  
  }  
}
// RDD -> DS
import spark.implicits._
rdd.toDS // 需要隐式转换才能使用toDS

```
- 读取数据
```scala
SparkSession spark
spark.read.json("path").load()
spark.read.format("json").load("src")
spark.read.load("src") // 默认的文件格式 parquet
```
- 写出数据
```scala
ds.write.json("dst").save
ds.write.format("json").save
ds.write.save("dst") // 默认的文件格式 parquet
```
## .4 提交流程
## .5 内核 Shuffle
- HashShuffle
	- 类似 `MR` 中的 `shuffle`，会产生许多小文件
	- `小文件个数 = Mapper数 * Reducer数`
- 优化后的 HashShuffle
	- 相同 `CPU` 执行的 `Task` 共用内存和文件
	- 将相同 `CPU` 执行的结果进行合并
- SortShuffle
	- 溢写磁盘前排序，最后合并产生一个文件，同时产生一个 `index` 文件，告诉 `Reducer` 去哪里拉取对应分区数据
	- `产生的文件个数 = Mapper 数 * 2`
- BypassMergeSortShuffle
	- 类似 Hash 按分区溢写，类似 Sort 合并文件
	- 用到了按分区溢写，Reducer 个数需要限制
```scala
def shouldBypassMergeSort(conf: SparkConf, dep: ShuffleDependency[_, _, _]): Boolean = {  
  // We cannot bypass sorting if we need to do map-side aggregation. 
  // 不排序，所以不能是预聚合算子 
  if (dep.mapSideCombine) {  
    false  
  } else {  
    // spark.shuffle.sort.bypassMergeThreshold = 200
    // 下游 Reducer 个数不能超过200
    val bypassMergeThreshold: Int = conf.get(config.SHUFFLE_SORT_BYPASS_MERGE_THRESHOLD)  
    dep.partitioner.numPartitions <= bypassMergeThreshold  
  }  
}
```