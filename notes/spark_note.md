# 1 提交 Job 流程
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
    // Hadoop 中启动 MrAppMaster 时创建 ApplicationSubmissionContext
    // Spark 先调用 createApplication 再创建 ApplicationSubmissionContext
    val newApp = yarnClient.createApplication()  
    val newAppResponse = newApp.getNewApplicationResponse()  
    appId = newAppResponse.getApplicationId()  
  
    // Set up the appropriate contexts to launch our AM  
    // cluster -> "org.apache.spark.deploy.yarn.ApplicationMaster"
    val containerContext = createContainerLaunchContext(newAppResponse)  
    val appContext = createApplicationSubmissionContext(newApp, containerContext)  
    // 提交任务 
    yarnClient.submitApplication(appContext)  
    appId  
  }
}
```
## 1.5 启动 ApplicationMaster
```scala
def main(args: Array[String]): Unit = {  
  master = new ApplicationMaster(amArgs)  
  System.exit(master.run())  
}

private[spark] class ApplicationMaster(args: ApplicationMasterArguments) extends Logging {
	// 创建 YarnRMClient 处理向 RM 注册和去注册的请求
	// YarnRMClient 创建了 AMRMClient[ContainerRequest]
	private val client = doAsUser { new YarnRMClient() }
}

// 运行 master
final def run(): Int = {  
  doAsUser {  
    runImpl()  
  }  
  exitCode  
}

private def runImpl(): Unit = {  
  try {  
  
    if (isClusterMode) {  
      System.setProperty("spark.ui.port", "0")  
      System.setProperty("spark.master", "yarn")  
      System.setProperty("spark.submit.deployMode", "cluster")   
      System.setProperty("spark.yarn.app.id", appAttemptId.getApplicationId().toString())  
      attemptID = Option(appAttemptId.getAttemptId.toString)  
    }  
  
    if (isClusterMode) {  
      // 启动 Driver
      runDriver()  
    } 
  }
}
```
## 1.6 启动 Driver
```scala
private def runDriver(): Unit = {  
  addAmIpFilter(None)  
  // 启动 Driver 线程
  userClassThread = startUserApplication()  
  
  logInfo("Waiting for spark context initialization...")  
  try {  
    val sc = ThreadUtils.awaitResult(sparkContextPromise.future,  
      Duration(totalWaitTime, TimeUnit.MILLISECONDS))  
    if (sc != null) {  
      // 等待 Context 的初始化，获取到 RPC 通信环境后去 RM 注册，申请资源
      rpcEnv = sc.env.rpcEnv  
      val userConf = sc.getConf  
      val host = userConf.get("spark.driver.host")  
      val port = userConf.get("spark.driver.port").toInt  
      registerAM(host, port, userConf, sc.ui.map(_.webUrl))  

	  // driver 的通信终端
      val driverRef = rpcEnv.setupEndpointRef(  
        RpcAddress(host, port),  
        YarnSchedulerBackend.ENDPOINT_NAME)  
      createAllocator(driverRef, userConf)  
    } 
}

private def startUserApplication(): Thread = {  
  var userArgs = args.userArgs  
  // 加载 main 方法
  val mainMethod = userClassLoader.loadClass(args.userClass)  
    .getMethod("main", classOf[Array[String]])  
  
  val userThread = new Thread {  
    override def run() {  
      try {  
        if (!Modifier.isStatic(mainMethod.getModifiers)) {...} 
        // 保证是 static
        else {  
          // 调用 main 方法初始化SparkContext
          mainMethod.invoke(null, userArgs.toArray)  
          logDebug("Done running user class")  
        }  
      }  
    }  
  }  
  userThread.setContextClassLoader(userClassLoader)  
  userThread.setName("Driver")  
  userThread.start()  
  userThread  
}
```
## 1.7 初始化 SparkContext
- 在 try 代码块中初始化成员变量
	- `SparkEnv`
	- `WebUI`
	- `Hadoop` 配置，`Executor` 环境变量设置
	- `HeartbeatReceiver`
	- `TaskScheduler`
	- `DAGScheduler`
- 最后通过 `setActiveContext()` 防止开启多个 `SparkContext`
### 1.7.1 SparkEnv
```java
_env = createSparkEnv(_conf, isLocal, listenerBus)  
SparkEnv.set(_env)

private[spark] def createSparkEnv(  
    conf: SparkConf,  
    isLocal: Boolean,  
    listenerBus: LiveListenerBus): SparkEnv = {  
  SparkEnv.createDriverEnv(conf, isLocal, listenerBus, SparkContext.numDriverCores(master, conf))  
}

// 为 Driver 创建环境
// SparkEnv.scala
private[spark] def createDriverEnv(  
    conf: SparkConf,  
    isLocal: Boolean,  
    listenerBus: LiveListenerBus,  
    numCores: Int,  
    mockOutputCommitCoordinator: Option[OutputCommitCoordinator] = None): SparkEnv = {  

  val bindAddress = conf.get(DRIVER_BIND_ADDRESS)  
  val advertiseAddress = conf.get(DRIVER_HOST_ADDRESS)  
  val port = conf.get(DRIVER_PORT)  

  create(  
    conf,  
    SparkContext.DRIVER_IDENTIFIER,  
    bindAddress,  
    advertiseAddress,  
    Option(port),  
    isLocal,  
    numCores,  
    ioEncryptionKey,  
    listenerBus = listenerBus,  
    mockOutputCommitCoordinator = mockOutputCommitCoordinator  
  )  
}

// 为 Driver 或 Executor 创建环境
private def create(  
    conf: SparkConf,  
    executorId: String,  
    bindAddress: String,  
    advertiseAddress: String,  
    port: Option[Int],  
    isLocal: Boolean,  
    numUsableCores: Int,  
    ioEncryptionKey: Option[Array[Byte]],  
    listenerBus: LiveListenerBus = null,  
    mockOutputCommitCoordinator: Option[OutputCommitCoordinator] = None): SparkEnv = {  
  
  val isDriver = executorId == SparkContext.DRIVER_IDENTIFIER  
  // 创建 SecurityManager
  val securityManager = new SecurityManager(conf, ioEncryptionKey, authSecretFileConf)  
  if (isDriver) {  
    securityManager.initializeAuth()  
  }  
  
  val systemName = if (isDriver) driverSystemName else executorSystemName  
  // 创建 RpcEnv
  val rpcEnv = RpcEnv.create(systemName, bindAddress, advertiseAddress, port.getOrElse(-1), conf,  
    securityManager, numUsableCores, !isDriver)  
  
  if (isDriver) {  
    conf.set(DRIVER_PORT, rpcEnv.address.port)  
  }  
  
  // 序列化器
  val closureSerializer = new JavaSerializer(conf)  
  // BroadcastManager
  val broadcastManager = new BroadcastManager(isDriver, conf)  
  // ShuffleManager
  val shuffleManager = Utils.instantiateSerializerOrShuffleManager[ShuffleManager](  
    shuffleMgrClass, conf, isDriver)  
  // BlockManager
  val blockManager = new BlockManager(  
    executorId,  
    rpcEnv,  
    blockManagerMaster,  
    serializerManager,  
    conf,  
    memoryManager,  
    mapOutputTracker,  
    shuffleManager,  
    blockTransferService,  
    securityManager,  
    externalShuffleClient)  

  // 实例化 SparkEnv
  val envInstance = new SparkEnv(  
    executorId,  
    rpcEnv,  
    serializer,  
    closureSerializer,  
    serializerManager,  
    mapOutputTracker,  
    shuffleManager,  
    broadcastManager,  
    blockManager,  
    securityManager,  
    metricsSystem,  
    memoryManager,  
    outputCommitCoordinator,  
    conf)  
  
  envInstance  
}
```
### 1.7.2 Executor 的内存

```java
_executorMemory = _conf.getOption(EXECUTOR_MEMORY.key)  
  .orElse(Option(System.getenv("SPARK_EXECUTOR_MEMORY")))  
  .orElse(Option(System.getenv("SPARK_MEM"))  
  .map(warnSparkMem))  
  .map(Utils.memoryStringToMb)  
  // 默认 1g
  .getOrElse(1024)
```
### 1.7.3 HeartbeatReceiver
```JAVA
// 接收 Executor 的心跳
_heartbeatReceiver = env.rpcEnv.setupEndpoint(  
  HeartbeatReceiver.ENDPOINT_NAME, new HeartbeatReceiver(this))

// NETWORK_TIMEOUT = 120s
private val executorTimeoutMs = sc.conf.get(  
  config.STORAGE_BLOCKMANAGER_HEARTBEAT_TIMEOUT  
).getOrElse(Utils.timeStringAsMs(s"${sc.conf.get(Network.NETWORK_TIMEOUT)}s"))

// EXECUTOR_HEARTBEAT_INTERVAL = 10s 
private val executorHeartbeatIntervalMs = sc.conf.get(config.EXECUTOR_HEARTBEAT_INTERVAL)
```
### 1.7.4 TaskScheduler
```java
val (sched, ts) = SparkContext.createTaskScheduler(this, master)  
_schedulerBackend = sched  
_taskScheduler = ts  
_dagScheduler = new DAGScheduler(this)

// 创建 TaskScheduler 以及 SchedulerBackend
private def createTaskScheduler(  
    sc: SparkContext,  
    master: String): (SchedulerBackend, TaskScheduler) = {
    
    master match {
	    case masterUrl =>
		try {  
		      // YarnClusterManager
		      val scheduler = cm.createTaskScheduler(...)  
			  val backend = cm.createSchedulerBackend(...)  
			  cm.initialize(scheduler, backend)  
  (backend, scheduler)  
}
```
## 1.8 AM 向 RM 申请资源
```scala
// 1.6 继续执行
// 向 RM 注册
private def registerAM(  
    host: String,  
    port: Int,  
    _sparkConf: SparkConf,  
    uiAddress: Option[String]): Unit = {   
  client.register(host, port, yarnConf, _sparkConf, uiAddress, historyAddress) 
  registered = true  
}

def register(  
    driverHost: String,  
    driverPort: Int,  
    conf: YarnConfiguration,  
    sparkConf: SparkConf,  
    uiAddress: Option[String],  
    uiHistoryAddress: String): Unit = {  
  // 创建实现类  
  amClient = AMRMClient.createAMRMClient()  
  amClient.init(conf)  
  amClient.start()  
  
  synchronized {  
    // AM 向 RM 注册
    amClient.registerApplicationMaster(driverHost, driverPort, trackingUrl)  
    registered = true  
  }  
}
```
## 1.9 创建 Allocator
```scala
private def createAllocator(driverRef: RpcEndpointRef, _sparkConf: SparkConf): Unit = {  
  val appId = client.getAttemptId().getApplicationId().toString()  
  val driverUrl = RpcEndpointAddress(driverRef.address.host, driverRef.address.port, CoarseGrainedSchedulerBackend.ENDPOINT_NAME).toString  
  // private val launcherPool = ThreadUtils.newDaemonCachedThreadPool(  "ContainerLauncher", sparkConf.get(CONTAINER_LAUNCH_MAX_THREADS))
  // 执行 Executor 的 ThreadPoolExecutor，默认 25 个线程
  allocator = client.createAllocator(  
    yarnConf,  
    _sparkConf,  
    driverUrl,  
    driverRef,  
    securityMgr,  
    localResources)  
  
  // Initialize the AM endpoint after the allocator has been initialized. This ensures  
  // that when the driver sends an initial executor request (e.g. after an AM restart), 
  // the allocator is ready to service requests.  
  rpcEnv.setupEndpoint("YarnAM", new AMEndpoint(rpcEnv, driverRef))  
  
  allocator.allocateResources()
}
```
## 1.10 分配资源启动 ExecutorRunnable
```scala
def allocateResources(): Unit = synchronized {  

  // AMRMClientImpl.java
  val allocateResponse = amClient.allocate(progressIndicator)  
  
  val allocatedContainers = allocateResponse.getAllocatedContainers()  
  
  if (allocatedContainers.size > 0) { 
    handleAllocatedContainers(allocatedContainers.asScala)  
  }   
}

def handleAllocatedContainers(allocatedContainers: Seq[Container]): Unit = {  
  runAllocatedContainers(containersToUse)  
}

private def runAllocatedContainers(containersToUse: ArrayBuffer[Container]): Unit = {  
    if (runningExecutors.size() < targetNumExecutors) {  
      numExecutorsStarting.incrementAndGet()  
      if (launchContainers) {  
        launcherPool.execute(new Runnable {  
          override def run(): Unit = {  
            try {  
              new ExecutorRunnable(  
                Some(container),  
                conf,  
                sparkConf,  
                driverUrl,  
                executorId,  
                executorHostname,  
                executorMemory,  
                executorCores,  
                appAttemptId.getApplicationId.toString,  
                securityMgr,  
                localResources  
              ).run()  
              updateInternalState()  
            }
          }  
        })  
      } 
    } 
  }  
}

private[yarn] class ExecutorRunnable(
	container: Option[Container],  
	conf: YarnConfiguration,  
	sparkConf: SparkConf,  
	masterAddress: String,  
	executorId: String,  
	hostname: String,  
	executorMemory: Int,  
	executorCores: Int,  
	appId: String,  
	securityMgr: SecurityManager,  
	localResources: Map[String, LocalResource]) extends Logging {
	
	var rpc: YarnRPC = YarnRPC.create(conf)  
	var nmClient: NMClient = _
}

def run(): Unit = {  
  logDebug("Starting Executor Container")  
  nmClient = NMClient.createNMClient()  
  nmClient.init(conf)  
  nmClient.start()  
  startContainer()  
}
```
## 1.11 开启 Container，封装启动 Executor 的命令
```scala
def startContainer(): java.util.Map[String, ByteBuffer] = {  
  // 封装启动 Executor 的命令 
  val commands = prepareCommand()  

  try {  
    nmClient.startContainer(container.get, ctx)  
  }  
}

private def prepareCommand(): List[String] = {
   ...
   // 执行bin/java ... org.apache.spark.executor.CoarseGrainedExecutorBackend
   val commands = prefixEnv ++
     Seq(Environment.JAVA_HOME.$$() + "/bin/java", "-server") ++
     javaOpts ++
     Seq("org.apache.spark.executor.CoarseGrainedExecutorBackend",
       "--driver-url", masterAddress,
       "--executor-id", executorId,
       "--hostname", hostname,
       "--cores", executorCores.toString,
       "--app-id", appId) ++
     userClassPath ++
     Seq(
       s"1>${ApplicationConstants.LOG_DIR_EXPANSION_VAR}/stdout",
       s"2>${ApplicationConstants.LOG_DIR_EXPANSION_VAR}/stderr")

   // TODO: it would be nicer to just make sure there are no null commands here
   commands.map(s => if (s == null) "null" else s).toList
 }
```
## 1.12 启动 CoarseGrainedExecutorBackend
```scala
def main(args: Array[String]): Unit = {  
  val createFn: (RpcEnv, Arguments, SparkEnv, ResourceProfile) =>  
    CoarseGrainedExecutorBackend = { case (rpcEnv, arguments, env, resourceProfile) =>  
    new CoarseGrainedExecutorBackend(rpcEnv, arguments.driverUrl, arguments.executorId,  
      arguments.bindAddress, arguments.hostname, arguments.cores,  
      env, arguments.resourcesFileOpt, resourceProfile)  
  }  
  run(parseArguments(args, this.getClass.getCanonicalName.stripSuffix("$")), createFn)  
  System.exit(0)  
}

def run(  
    arguments: Arguments,  
    backendCreateFn: (RpcEnv, Arguments, SparkEnv, ResourceProfile) =>  
      CoarseGrainedExecutorBackend): Unit = {  
  
  
  SparkHadoopUtil.get.runAsSparkUser { () =>  
  
    // Bootstrap to fetch the driver's Spark properties.  
    val executorConf = new SparkConf  
    val fetcher = RpcEnv.create(  
      "driverPropsFetcher",  
      arguments.bindAddress,  
      arguments.hostname,  
      -1,  
      executorConf,  
      new SecurityManager(executorConf),  
      numUsableCores = 0,  
      clientMode = true)  

    // 找到 Driver 的 ref
    var driver: RpcEndpointRef = null  
  
    // Create SparkEnv using properties we fetched from the driver.  
    val env = SparkEnv.createExecutorEnv(driverConf, arguments.executorId, arguments.bindAddress,arguments.hostname, arguments.cores, cfg.ioEncryptionKey, isLocal = false)  
    // 设置 Executor 的终端
    val backend = backendCreateFn(env.rpcEnv, arguments, env, cfg.resourceProfile)  
    env.rpcEnv.setupEndpoint("Executor", backend)  
    arguments.workerUrl.foreach { url =>  
      env.rpcEnv.setupEndpoint("WorkerWatcher",  
        new WorkerWatcher(env.rpcEnv, url, isChildProcessStopping = backend.stopping))  
    }  
  }  
}
```
## 1.13 设置 RPC 终端
- `RpcEnv`: `Spark` 使用 `Netty`
- `RpcEndpoint`：`Spark` 每个节点均会实现 `RpcEndpoint` 接口
	- 调用顺序：`Constructor -> onStart -> receive -> onStop`
- `RpcEndpointRef`: `RpcEndpoint` 的引用，向 `RpcEndpoint` 发送消息时需要先获取到
- `Dispatcher`：
	- 将发送远程消息或者从远程 RPC 接收到的消息分发至 `InBox` 或 `OutBox`
	- 如果指令接收方是自己则存入 `InBox`，如果指令接收方不是自己，则放入 `OutBox`
- `InBox`: `RpcEndpoint` 对应一个 `InBox`
	- `Dispatcher` 向 `Inbox` 存入消息时，都将对应 `EndpointData` 加入内部 `ReceiverQueue` 中
	- `Dispatcher` 创建时启动一个单独线程进行轮询 `ReceiverQueue`，进行收件箱消息消费
- `TransportClient`:负责消费 `OutBox` 的消息，发给 `TransportServer` 后，再通过 `Dispatcher` 发往 `Inbox`
```scala
override def setupEndpoint(name: String, endpoint: RpcEndpoint): RpcEndpointRef = {	  
    // NettyRpcEnv的对象创建时会创建一个Dispatcher
    dispatcher.registerRpcEndpoint(name, endpoint)
}

 def registerRpcEndpoint(name: String, endpoint: RpcEndpoint): NettyRpcEndpointRef = {
   val addr = RpcEndpointAddress(nettyEnv.address, name)
   val endpointRef = new NettyRpcEndpointRef(nettyEnv.conf, addr, nettyEnv)
   synchronized {

     // 定义MessageLoop变量，Executor的终端是IsolatedRpcEndpoint类型，创建一个DedicatedMessageLoop对象
     var messageLoop: MessageLoop = null
     try {
       messageLoop = endpoint match {
         case e: IsolatedRpcEndpoint =>
           new DedicatedMessageLoop(name, e, this)
         case _ =>
           sharedLoop.register(name, endpoint)
           sharedLoop
       }
       endpoints.put(name, messageLoop)
     }
   }
   endpointRef
}

// MessageLoop有两个实现类:服务于多个RPC端点的SharedMessageLoop和以及服务单个端点DedicatedMessageLoop
private class DedicatedMessageLoop(
    name: String,
    endpoint: IsolatedRpcEndpoint,
    dispatcher: Dispatcher)
  extends MessageLoop(dispatcher) {
  // 定义了一个Inbox对象
  private val inbox = new Inbox(name, endpoint)
  // 创建ThreadPool
  override protected val threadpool = if (endpoint.threadCount() > 1) {
    ThreadUtils.newDaemonCachedThreadPool(s"dispatcher-$name", endpoint.threadCount())
  }

  (1 to endpoint.threadCount()).foreach { _ =>
	// 从 inbox 中取出 Runnable对象
    threadpool.submit(receiveLoopRunnable)
  }

  // Mark active to handle the OnStart message.
  setActive(inbox)
}
```
## 1.14 ExecutorBackend 向 Driver 注册
```scala
override def onStart(): Unit = {  
  // 通过 driverUrl 获取 Driver 的 Ref
  rpcEnv.asyncSetupEndpointRefByURI(driverUrl).flatMap { ref =>  
    // This is a very fast action so we can use "ThreadUtils.sameThread"  
    driver = Some(ref)  
    env.executorBackend = Option(this)  
    // 向 Driver 发送注册 Executor 的消息
    ref.ask[Boolean](RegisterExecutor(executorId, self, hostname, cores, extractLogUrls, extractAttributes, _resources, resourceProfile.id))  
  }(ThreadUtils.sameThread).onComplete {  
    case Success(_) =>  
      self.send(RegisteredExecutor)  
  }(ThreadUtils.sameThread)  
}

// NettyRpcEnv
override def ask[T: ClassTag](message: Any, timeout: RpcTimeout): Future[T] = {  
  askAbortable(message, timeout).future  
}

private[netty] def askAbortable[T: ClassTag](  
    message: RequestMessage, timeout: RpcTimeout): AbortableRpcFuture[T] = {  
  val promise = Promise[Any]()  
  val remoteAddr = message.receiver.address  
  var rpcMsg: Option[RpcOutboxMessage] = None  
  
  def onFailure(e: Throwable): Unit = {  
    if (!promise.tryFailure(e)) {  
      e match {  
        case e : RpcEnvStoppedException => logDebug(s"Ignored failure: $e")  
        case _ => logWarning(s"Ignored failure: $e")  
      }  
    }  
  }  
  
  def onSuccess(reply: Any): Unit = reply match {  
    case RpcFailure(e) => onFailure(e)  
    case rpcReply =>  
      if (!promise.trySuccess(rpcReply)) {  
        logWarning(s"Ignored message: $reply")  
      }  
  }  
  
  def onAbort(t: Throwable): Unit = {  
    onFailure(t)  
    rpcMsg.foreach(_.onAbort())  
  }  
  
  try {  
    if (remoteAddr == address) {...}
    else {  
      // 创建 rpcMessage 发往 OutBox
      val rpcMessage = RpcOutboxMessage(message.serialize(this),  
        onFailure,  
        (client, response) => onSuccess(deserialize[Any](client, response)))  
      rpcMsg = Option(rpcMessage)  
      postToOutbox(message.receiver, rpcMessage)  
    }  
}

// SparkContext 会创建 Driver 的 RPC 终端
// CoarseGrainedSchedulerBackend.scala
// 处理注册 Driver 的消息
override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {  
  
  case RegisterExecutor(executorId, executorRef, hostname, cores, logUrls,  
      attributes, resources, resourceProfileId) =>  
    if (executorDataMap.contains(executorId)) {...} 
    else if (scheduler.excludedNodes.contains(hostname) ||  
        isExecutorExcluded(executorId, hostname)) {...} 
	else {        
      val executorAddress = if (executorRef.address != null) {  
          executorRef.address  
        } else {  
          context.senderAddress  
        }  
      // 将 Executor 加入到 Map 中
      CoarseGrainedSchedulerBackend.this.synchronized {  
        executorDataMap.put(executorId, data)   
      }  
      // 回复SUCCESS
      context.reply(true)  
    }
}
```
## 1.15 注册成功后，创建 Executor 准备开启 Task
```scala
// ExecutorBackend 进行处理(1.14)
case Success(_) =>  
      self.send(RegisteredExecutor) 

// 收到 Driver 注册成功的消息后，向自己发送 RegisteredExecutor 消息
override def receive: PartialFunction[Any, Unit] = {  
  case RegisteredExecutor =>  
    logInfo("Successfully registered with driver")  
    try {  
      // 创建Executor，向 Driver 发送 LaunchedExecutor 消息
      executor = new Executor(executorId, hostname, env, getUserClassPath, isLocal = false, resources = _resources)  
      driver.get.send(LaunchedExecutor(executorId))  
    }
}

// Driver 在 receive 方法中处理启动请求
override def receive: PartialFunction[Any, Unit] = {
	case LaunchedExecutor(executorId) =>  
  executorDataMap.get(executorId).foreach { data =>  
    data.freeCores = data.totalCores  
  }  
  makeOffers(executorId)
}

private def makeOffers(executorId: String): Unit = {  
  // Make sure no executor is killed while some task is launching on it  
  val taskDescs = withLock {  
    // Filter out executors under killing  
    if (isExecutorActive(executorId)) {  
      val executorData = executorDataMap(executorId)  
      // Executor 可用资源的抽象
      val workOffers = IndexedSeq(  
        new WorkerOffer(
	        executorId, 
	        executorData.executorHost, 
	        executorData.freeCores,  
	        Some(executorData.executorAddress.hostPort),  
            executorData.resourcesInfo.map { 
	            case (rName, rInfo) =>  
	            (rName, rInfo.availableAddrs.toBuffer)  
	          }, 
	        executorData.resourceProfileId))  
      scheduler.resourceOffers(workOffers, false)  
    } else {  
      Seq.empty  
    }  
  }  
  if (taskDescs.nonEmpty) { 
    // 准备启动 Task 
    launchTasks(taskDescs)  
  }  
}
```
## 1.16 Driver 通知 Executor 执行任务
```scala
override def receive: PartialFunction[Any, Unit] = {
	case LaunchTask(data) =>  
  if (executor == null) {  
    exitExecutor(1, "Received LaunchTask command but executor was null")  
  } else {  
    val taskDesc = TaskDescription.decode(data.value)  
    logInfo("Got assigned task " + taskDesc.taskId)  
    taskResources(taskDesc.taskId) = taskDesc.resources  
    executor.launchTask(this, taskDesc)  
  }
}

// Executor.scala
def launchTask(context: ExecutorBackend, taskDescription: TaskDescription): Unit = {  
  val taskId = taskDescription.taskId  
  // 创建 TaskRunner，继承了 Runnable
  val tr = createTaskRunner(context, taskDescription)  
  threadPool.execute(tr)  
}

override def run(): Unit = {  
 
  val ser = env.closureSerializer.newInstance()  
  logInfo(s"Running $taskName")  
  
  try {  
    task = ser.deserialize[Task[Any]](  
		taskDescription.serializedTask, 
		Thread.currentThread.getContextClassLoader)  
		
    val value = Utils.tryWithSafeFinally { 
      // run 方法会执行 runTask 
      val res = task.run(  
        taskAttemptId = taskId,  
        attemptNumber = taskDescription.attemptNumber,  
        metricsSystem = env.metricsSystem,  
        cpus = taskDescription.cpus,  
        resources = taskDescription.resources,  
        plugins = plugins)  
      threwException = false  
      res  
    } 
}
```
# 2 RDD 的依赖
- `Resilient Distributed Datasets`
```java
abstract class Dependency[T] extends Serializable {
  // 依赖的 RDD
  def rdd: RDD[T]
}
```
## 2.1 窄依赖
```java
// RDD与上游RDD的分区是一对一的关系
@DeveloperApi
abstract class NarrowDependency[T](_rdd: RDD[T]) extends Dependency[T] {
  def getParents(partitionId: Int): Seq[Int]
  override def rdd: RDD[T] = _rdd
}
```
### 2.1.1 OneToOneDependency

![[narrow_dep.svg]]

```java
class OneToOneDependency[T](rdd: RDD[T]) extends NarrowDependency[T](rdd) {  
  override def getParents(partitionId: Int): List[Int] = List(partitionId)  
}
```
### 2.1.2 RangeDependency

![[narrow_dep2.svg]]

```java
class RangeDependency[T](rdd: RDD[T], inStart: Int, outStart: Int, length: Int)  
  extends NarrowDependency[T](rdd) {  
  // inStart: 父RDD的分区范围起始值
  // outStart: 子RDD的分区范起始值
  // 索引为partitionId的子RDD对应于索引为partitionId - outStart + inStart的父RDD的分区
  override def getParents(partitionId: Int): List[Int] = {  
    if (partitionId >= outStart && partitionId < outStart + length) {  
      List(partitionId - outStart + inStart)  
    } else {  
      Nil  
    }  
  }  
}
```
## 2.2 宽依赖
```java
class ShuffleDependency[K: ClassTag, V: ClassTag, C: ClassTag](
    @transient private val _rdd: RDD[_ <: Product2[K, V]],
    val partitioner: Partitioner, // 分区器
    val serializer: Serializer = SparkEnv.get.serializer, // SparkEnv序列化器，默认org.apache.spark.serializer.JavaSerializer
    val keyOrdering: Option[Ordering[K]] = None,
    val aggregator: Option[Aggregator[K, V, C]] = None, // 对map的输出进行聚合的聚合器
    val mapSideCombine: Boolean = false,
    val shuffleWriterProcessor: ShuffleWriteProcessor = new ShuffleWriteProcessor)
  extends Dependency[Product2[K, V]] with Logging {...}
```
## 2.3 分区器
### 2.3.1 HashPartitioner
```scala
class HashPartitioner(partitions: Int) extends Partitioner {

  def numPartitions: Int = partitions

  def getPartition(key: Any): Int = key match {
    case null => 0
    case _ => Utils.nonNegativeMod(key.hashCode, numPartitions)
  }

  override def hashCode: Int = numPartitions
}

def nonNegativeMod(x: Int, mod: Int): Int = {
  val rawMod = x % mod
  rawMod + (if (rawMod < 0) mod else 0)
}
```
### 2.3.2 RangePartitioner
#### 2.3.2.1 Reservoir Sampling
- 选取前 $k$ 个元素
- 从 $k + i (i >= 0)$ 个元素开始，以概率 $$\frac{k}{k+i}$$ 的概率来决定是否保留该元素，若保存，则随机丢弃前 k 个元素中的一个
- 添加第 $k+1$ 个元素，前 $k$ 个元素被保留的概率为 $$1-\frac{k}{k+1}\times\frac{1}{k}$$，遍历到第 $n$ 个元素时，前 $k$ 个元素中任一个被保留的概率为: $$1\times(1-\frac{k}{k+1}\times\frac{1}{k})\times(1-\frac{k}{k+2}\times\frac{1}{k})\times\dots(1-\frac{k}{n-1}\times\frac{1}{k})\times(1-\frac{k}{n}\times\frac{1}{k})=\frac{k}{n}$$
- 第 $i$ 个元素在之后的第 $m$ 次遍历中被剔除的概率是$$\frac{k}{i+m}\times\frac{1}{k}$$, 则第 $i$ 个元素在 $n$ 次遍历中被保留的概率为: $$1\times(1-\frac{k}{i+1}\times\frac{1}{k})\times(1-\frac{k}{i+2}\times\frac{1}{k})\times\dots(1-\frac{k}{n-1}\times\frac{1}{k})\times(1-\frac{k}{n}\times\frac{1}{k})=\frac{k}{n}$$
#### 2.3.2.2 rangeBounds()
```scala
// 计算范围边界
private var rangeBounds: Array[K] = {
      // 采样大小
      val sampleSize = math.min(samplePointsPerPartitionHint.toDouble * partitions, 1e6) // samplePointsPerPartitionHint: 20
      // 保证分区数少时能收集更多样本
      val sampleSizePerPartition = math.ceil(3.0 * sampleSize / rdd.partitions.length).toInt
      // 将K和每个分区的采样数传入 sketch 方法
      val (numItems, sketched) = RangePartitioner.sketch(rdd.map(_._1), sampleSizePerPartition)
      // sketch返回(总记录数，Array[(PartitionId, 对应分区记录数, 样本)])
      if (numItems == 0L) {...} 
      else {
        val fraction = math.min(sampleSize / math.max(numItems, 1L), 1.0)
        val candidates = ArrayBuffer.empty[(K, Float)]
        val imbalancedPartitions = mutable.Set.empty[Int]
        sketched.foreach { case (idx, n, sample) =>
          if (fraction * n > sampleSizePerPartition) { // 超过sampleSizePerPartition再进行一次采样
            imbalancedPartitions += idx
          } else {
            // The weight is 1 over the sampling probability.
            val weight = (n.toDouble / sample.length).toFloat
            for (key <- sample) {
              candidates += ((key, weight))
            }
          }
        }
        if (imbalancedPartitions.nonEmpty) {
          // 重采样
          val imbalanced = new PartitionPruningRDD(rdd.map(_._1), imbalancedPartitions.contains)
          val seed = byteswap32(-rdd.id - 1)
          val reSampled = imbalanced.sample(withReplacement = false, fraction, seed).collect()
          val weight = (1.0 / fraction).toFloat
          candidates ++= reSampled.map(x => (x, weight))
        }
        // 计算边界
        RangePartitioner.determineBounds(candidates, math.min(partitions, candidates.size))
      }
    }
  }
```
#### 2.3.2.3 采样
```scala
/**
  * @param rdd 需要采集数据的RDD
  * @param sampleSizePerPartition 每个partition采集的数据量
  * @return <采样RDD数据总量,<partitionId, 当前分区的数据量，当前分区采集的数据量>>
  */
def sketch[K : ClassTag](
    rdd: RDD[K],
    sampleSizePerPartition: Int): (Long, Array[(Int, Long, Array[K])]) = {
  val shift = rdd.id
  val sketched = rdd.mapPartitionsWithIndex { (idx, iter) =>
    // 根据RDD的id获得seed
    val seed = byteswap32(idx ^ (shift << 16))
    // 使用水塘抽样算法进行抽样，返回Tuple2<Partition中抽取的样本量，Partition中包含的数据量>
    val (sample, n) = SamplingUtils.reservoirSampleAndCount(
      iter, sampleSizePerPartition, seed)
    Iterator((idx, n, sample))
  }.collect()
  val numItems = sketched.map(_._2).sum
  (numItems, sketched)
}

def reservoirSampleAndCount[T: ClassTag](
    input: Iterator[T],
    k: Int, // 每个分区抽样的数据量
    seed: Long = Random.nextLong())
  : (Array[T], Long) = {
  val reservoir = new Array[T](k)
  var i = 0
  while (i < k && input.hasNext) {
    val item = input.next()
    reservoir(i) = item
    i += 1
  }
  if (i < k) {
    // 不够k个数据时直接返回
    val trimReservoir = new Array[T](i)
    System.arraycopy(reservoir, 0, trimReservoir, 0, i)
    (trimReservoir, i)
  } else {
    // 否则继续采样
    var l = i.toLong
    val rand = new XORShiftRandom(seed) // 随机数生成器
    while (input.hasNext) {
      val item = input.next()
      l += 1

      val replacementIndex = (rand.nextDouble() * l).toLong // [0, l)中随机一个
      if (replacementIndex < k) {
        reservoir(replacementIndex.toInt) = item
      }
    }
    (reservoir, l)
  } 
}
```
#### 2.3.2.4 determineBounds()
```scala
def determineBounds[K : Ordering : ClassTag](
    candidates: ArrayBuffer[(K, Float)], // (K, weight)
    partitions: Int): Array[K] = {
  val ordering = implicitly[Ordering[K]]
  val ordered = candidates.sortBy(_._1) // 通过key排序
  val numCandidates = ordered.size
  val sumWeights = ordered.map(_._2.toDouble).sum // 总权重
  val step = sumWeights / partitions // 步长
  var cumWeight = 0.0
  var target = step
  val bounds = ArrayBuffer.empty[K]
  var i = 0
  var j = 0
  var previousBound = Option.empty[K]
  while ((i < numCandidates) && (j < partitions - 1)) {
    val (key, weight) = ordered(i)
    cumWeight += weight
    if (cumWeight >= target) {
      // Skip duplicate values.
      if (previousBound.isEmpty || ordering.gt(key, previousBound.get)) {
        bounds += key // 达到target即step时将key记录为bounds
        target += step
        j += 1
        previousBound = Some(key)
      }
    }
    i += 1
  }
  bounds.toArray
}
```
# 3 Stage


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
		- 一个很长的任务链中间的 `shuffle` 阶段后，防止重复shuffle`
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
	- 用到了按分区溢写，Reducer 个数需要限制，不能是预聚合算子
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