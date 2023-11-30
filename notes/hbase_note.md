# 1. 启动
## 1.1 脚本
### 1.1.1 start-hbase.sh
```shell
# hbase-site.xml将分布式设置为true
distMode=`$bin/hbase --config "$HBASE_CONF_DIR" org.apache.hadoop.hbase.util.HBaseConfTool hbase.cluster.distributed | head -n 1`

# 没有传入启动命令，默认为start
if [ "$1" = "autostart" ]
then
  commandToRun="--autostart-window-size ${AUTOSTART_WINDOW_SIZE} --autostart-window-retry-limit ${AUTOSTART_WINDOW_RETRY_LIMIT} autostart"
else
  commandToRun="start"
fi

if [ "$distMode" == 'false' ] 
then
  "$bin"/hbase-daemon.sh --config "${HBASE_CONF_DIR}" $commandToRun master
else
  "$bin"/hbase-daemon.sh --config "${HBASE_CONF_DIR}" $commandToRun master
  "$bin"/hbase-daemons.sh --config "${HBASE_CONF_DIR}" \
    --hosts "${HBASE_REGIONSERVERS}" $commandToRun regionserver
fi
```
### 1.1.2 hbase-daemon.sh
```shell
case $startStop in
(start)
    echo running $command, logging to $HBASE_LOGOUT
    $thiscmd --config "${HBASE_CONF_DIR}" \
        foreground_start $command $args < /dev/null > ${HBASE_LOGOUT} 2>&1  &
    disown -h -r
    sleep 1; head "${HBASE_LOGOUT}"
  ;;

(foreground_start)
	# trap命令用于指定在接收到信号后将要采取的动作
    trap cleanAfterRun SIGHUP SIGINT SIGTERM EXIT
    ...
  ;;
esac

cleanAfterRun() {

  if [ -f ${HBASE_ZNODE_FILE} ]; then
    if [ "$command" = "master" ]; then
      # 调用hbse master启动HMaster
      HBASE_OPTS="$HBASE_OPTS $HBASE_MASTER_OPTS" $bin/hbase master clear > /dev/null 2>&1
    fi
    rm ${HBASE_ZNODE_FILE}
  fi
}
```
### 1.1.3 hbase-daemons.sh
```shell
# 通过regionservers.sh启动RegionServer
command=$2
case $command in
  (*)
    exec "$bin/regionservers.sh" $args
    ;;
esac
```
### 1.1.4 regionservers.sh
```shell
# 从conf/regionservers中获取regionserver地址
HOSTLIST=$HBASE_REGIONSERVERS
if [ "$HOSTLIST" = "" ]; then
  if [ "$HBASE_REGIONSERVERS" = "" ]; then
    export HOSTLIST="${HBASE_CONF_DIR}/regionservers"
  else
    export HOSTLIST="${HBASE_REGIONSERVERS}"
  fi
fi
regionservers=`cat "$HOSTLIST"`
# ssh到每个server上去启动
if [ "$regionservers" = "localhost" ]; then
else
  for regionserver in `cat "$HOSTLIST"`; do
    if ${HBASE_SLAVE_PARALLEL:-true}; then
      ssh $HBASE_SSH_OPTS $regionserver $"${@// /\\ }" \
        2>&1 | sed "s/^/$regionserver: /" &
    else # run each command serially
      ssh $HBASE_SSH_OPTS $regionserver $"${@// /\\ }" \
        2>&1 | sed "s/^/$regionserver: /"
    fi
  done
fi
```
### 1.1.5 hbase
```shell
elif [ "$COMMAND" = "master" ] ; then
  CLASS='org.apache.hadoop.hbase.master.HMaster'
  if [ "$1" != "stop" ] && [ "$1" != "clear" ] ; then
    HBASE_OPTS="$HBASE_OPTS $HBASE_MASTER_OPTS"
  fi
elif [ "$COMMAND" = "regionserver" ] ; then
  CLASS='org.apache.hadoop.hbase.regionserver.HRegionServer'
  if [ "$1" != "stop" ] ; then
    HBASE_OPTS="$HBASE_OPTS $HBASE_REGIONSERVER_OPTS"
  fi
```
## 1.2 HMaster.main()
```java
public static void main(String [] args) {
    new HMasterCommandLine(HMaster.class).doMain(args);
}

public void doMain(String args[]) {
    try {
      int ret = ToolRunner.run(HBaseConfiguration.create(), this, args);
    }
}

public static int run(Configuration conf, Tool tool, String[] args) 
    throws Exception{
    GenericOptionsParser parser = new GenericOptionsParser(conf, args);
    //set the configuration back, so that Tool can configure itself
    tool.setConf(conf);
    
    //get the args w/o generic hadoop args
    String[] toolArgs = parser.getRemainingArgs();
    return tool.run(toolArgs);
}

// HMasterCommandLine
public int run(String args[]) throws Exception {
    if ("start".equals(command)) {
      // 启动master
      return startMaster();
    } 
}
```
## 1.3 startMaster()
```java
private int startMaster() {
    try {
      // If 'local', defer to LocalHBaseCluster instance.  Starts master
      // and regionserver both in the one JVM.
      if (LocalHBaseCluster.isLocal(conf)) {...} 
        else {
        logProcessInfo(getConf());
        // 通过反射创建HMaster对象
        HMaster master = HMaster.constructMaster(masterClass, conf);

        // HMaster继承了HRegionServer继承了HasThread
        master.start();
        master.join();
    } 
    return 0;
}
```
## 1.4 创建 HMaster 实例
```java
public HMaster(final Configuration conf)  
    throws IOException, KeeperException {  
  // 调用父类的构造器
  super(conf);  
}

public HRegionServer(Configuration conf) throws IOException {  
  super("RegionServer");  // thread name  
  TraceUtil.initTracer(conf);  
  try {  
    // 检测否有足够的内存分配给Memstore和Block Cache使用
    // DEFAULT_MEMSTORE_SIZE = 0.4f 分配给memstore的内存
    // HFILE_BLOCK_CACHE_SIZE_DEFAULT = 0.4f 分配给block cache的内存
    MemorySizeUtil.checkForClusterFreeHeapMemoryLimit(this.conf);
    // HMaster和HRegionServer创建各自的RpcService
    rpcServices = createRpcServices();  
    // 根据主机名端口和启动时间确定服务名 
    serverName = ServerName.valueOf(hostName, this.rpcServices.isa.getPort(), this.startcode);  
  
    if (!isMasterNotCarryTable) {
      // 实例化BlockCache
      CacheConfig.instantiateBlockCache(conf);  
	    }  
    };  
	// 获取HBase在hdfs上的各个存储目录: 比如WAL预写日志, 数据存储路径等
    initializeFileSystem();  
   
    // Some unit tests don't need a cluster, so no zookeeper at all  
    if (!conf.getBoolean("hbase.testing.nocluster", false)) {  
      // Open connection to zookeeper and set primary watcher 
      // 连接到ZK
      zooKeeper = new ZKWatcher(conf, getProcessName() + ":" +  
    // 启动rpcServices, 等待regionserver端和客户端的请求
    this.rpcServices.start(zooKeeper);  
    }
}
```
## 1.5 启动 HMaster
```java
public void run() {
    try {
      if (!conf.getBoolean("hbase.testing.nocluster", false)) {
        Threads.setDaemonThreadRunning(new Thread(() -> {
          try {
            int infoPort = putUpJettyServer();
            // 启动ActiveMasterManager
            startActiveMasterManager(infoPort);
          }
        }));
      }
      // 成为active master后启动HRegionServer
      super.run();
    } 
}

private void startActiveMasterManager(int infoPort) throws KeeperException {
      // 没有active的master时阻塞
      while (!activeMasterManager.hasActiveMaster()) {
        LOG.debug("Waiting for master address and cluster state znode to be written.");
        Threads.sleep(timeout);
      }
    }
    try {
          // 尝试成为ActiveMaster, 在zk上创建active节点
      if (activeMasterManager.blockUntilBecomingActiveMaster(timeout, status)) {
          // 成为active master后执行
          finishActiveMasterInitialization(status);
      }
    }
}

private void finishActiveMasterInitialization(MonitoredTask status)
      throws IOException, InterruptedException, KeeperException {
   
    this.fileSystemManager = new MasterFileSystem(conf);
    // MasterWalManager有一个属性成员SplitLogManager
    this.walManager = new MasterWalManager(this);
    
    // HBase 集群启动的时候会存储 clusterID，当 hmaster 成为 active 的时候，会将这个 clusterID 信息写入到 zk 中
    ClusterId clusterId = fileSystemManager.getClusterId();

	// HMaster 用来维护 online RegionServer
    this.serverManager = createServerManager(this);
    // HBase的状态机
    createProcedureExecutor();

  initMetaProc = optProc.orElseGet(() -> {
	// schedule an init meta procedure if meta has not been deployed yet
	// 向procedureExecutor添加一个InitMetaProcedure任务，创建meta表
	InitMetaProcedure temp = new InitMetaProcedure();
	procedureExecutor.submitProcedure(temp);
	return temp;
      });
    }
    waitForRegionServers(status);
  }
```
## 1.6 启动 RegionServer
```java
public void run() {
    try {
      // 初始化zk, RpcClient
      preRegistrationInitialization();
    } 
    try {

      while (keepLooping()) {
        // 向master注册，master进行响应
        RegionServerStartupResponse w = reportForDuty();
        if (w == null) {
        } else {
          // 处理注册结果，若成功，在zk上创建节点
          handleReportForDutyResponse(w);
          break;
        }
      }

      // We registered with the Master.  Go into run mode.
      long lastMsg = System.currentTimeMillis();
      long oldRequestCount = -1;
      // The main run loop.
      while (!isStopped() && isHealthy()) {
        long now = System.currentTimeMillis();
        if ((now - lastMsg) >= msgInterval) {
          // 发送心跳
          tryRegionServerReport(lastMsg, now);
          lastMsg = System.currentTimeMillis();
        }
    }
}

private RegionServerStartupResponse reportForDuty() throws IOException {
      
      // 创建RegionServerStartupRequest的Builder
      RegionServerStartupRequest.Builder request = RegionServerStartupRequest.newBuilder();
      // 调用MasterRpcService的方法发送请求给master，master返回RegionServerStartupResponse
      result = this.rssStub.regionServerStartup(null, request.build());
    }
    return result;
}

protected void handleReportForDutyResponse(final RegionServerStartupResponse c)
  throws IOException {
    try {
      // Set our ephemeral znode up in zookeeper now we have a name.
      // 获取到name后在zk上/hbase/rs节点下创建znode
      createMyEphemeralNode();
  }

```
# 2. 存储结构
# 3. RegionServer 架构

![[RegionServer_framework.svg]]
## 3.1 BlockCache
### 3.1.1 instantiateBlockCache()
```java
// 创建HMaster实例时初始化
public static synchronized BlockCache instantiateBlockCache(Configuration conf) {
    // 堆内缓存
    LruBlockCache onHeapCache = getOnHeapCacheInternal(conf);
    // EXTERNAL_BLOCKCACHE_DEFAULT默认false
    boolean useExternal = conf.getBoolean(EXTERNAL_BLOCKCACHE_KEY, EXTERNAL_BLOCKCACHE_DEFAULT);
    if (useExternal) {...} 
    else {
      // otherwise use the bucket cache.
      // 堆外缓存
      L2_CACHE_INSTANCE = getBucketCache(conf);
      // 成功分配到堆外缓存，则通过CombinedBlockCache将两种缓存结合使用
      GLOBAL_BLOCK_CACHE_INSTANCE = L2_CACHE_INSTANCE == null ? onHeapCache
          : new CombinedBlockCache(onHeapCache, L2_CACHE_INSTANCE);
    }
    return GLOBAL_BLOCK_CACHE_INSTANCE;
  }
```
### 3.1.2 getOnHeapCacheInternal()
```java
// L1缓存
private synchronized static LruBlockCache getOnHeapCacheInternal(final Configuration c) {
    // HFILE_BLOCK_CACHE_SIZE_DEFAULT = 0.4f
    final long cacheSize = MemorySizeUtil.getOnHeapCacheSize(c);
    // DEFAULT_BLOCKSIZE = 64 * 1024 block默认大小64kB
    int blockSize = c.getInt(BLOCKCACHE_BLOCKSIZE_KEY, HConstants.DEFAULT_BLOCKSIZE);
    ONHEAP_CACHE_INSTANCE = new LruBlockCache(cacheSize, blockSize, true, c);
    return ONHEAP_CACHE_INSTANCE;
}

public LruBlockCache(long maxSize, long blockSize, boolean evictionThread, Configuration conf) {
    this(maxSize, blockSize, evictionThread,
        (int) Math.ceil(1.2 * maxSize / blockSize),
        // Backing Concurrent Map Configuration 
        DEFAULT_LOAD_FACTOR, 
        DEFAULT_CONCURRENCY_LEVEL,
        // eviction的参数: DEFAULT_MIN_FACTOR = 0.95f 
        conf.getFloat(LRU_MIN_FACTOR_CONFIG_NAME, DEFAULT_MIN_FACTOR),
        // DEFAULT_ACCEPTABLE_FACTOR = 0.99f
        conf.getFloat(LRU_ACCEPTABLE_FACTOR_CONFIG_NAME, DEFAULT_ACCEPTABLE_FACTOR),
        // 存储优先级的默认值
        /* 
         * DEFAULT_SINGLE_FACTOR = 0.25f;
         * DEFAULT_MULTI_FACTOR = 0.50f;
         * DEFAULT_MEMORY_FACTOR = 0.25f;
        */
        conf.getFloat(LRU_SINGLE_PERCENTAGE_CONFIG_NAME, DEFAULT_SINGLE_FACTOR),
        conf.getFloat(LRU_MULTI_PERCENTAGE_CONFIG_NAME, DEFAULT_MULTI_FACTOR),
        conf.getFloat(LRU_MEMORY_PERCENTAGE_CONFIG_NAME, DEFAULT_MEMORY_FACTOR),
        // DEFAULT_HARD_CAPACITY_LIMIT_FACTOR = 1.2f;
        conf.getFloat(LRU_HARD_CAPACITY_LIMIT_FACTOR_CONFIG_NAME,
                      DEFAULT_HARD_CAPACITY_LIMIT_FACTOR),
        conf.getBoolean(LRU_IN_MEMORY_FORCE_MODE_CONFIG_NAME, DEFAULT_IN_MEMORY_FORCE_MODE), // 默认false
        // DEFAULT_MAX_BLOCK_SIZE = 16L * 1024L * 1024L 16M
        conf.getLong(LRU_MAX_BLOCK_SIZE, DEFAULT_MAX_BLOCK_SIZE)
    );
  }

public LruBlockCache(long maxSize, long blockSize, boolean evictionThread,
      int mapInitialSize, float mapLoadFactor, int mapConcurrencyLevel,
      float minFactor, float acceptableFactor, float singleFactor,
      float multiFactor, float memoryFactor, float hardLimitFactor,
      boolean forceInMemory, long maxBlockSize) {
    this.maxBlockSize = maxBlockSize;
    // 判断factor取值是否合法

    // 初始化属性

    // 创建一个ConcurrentHashMap保存Block的映射关系
    map = new ConcurrentHashMap<>(mapInitialSize, mapLoadFactor, mapConcurrencyLevel);
   // 开启淘汰线程
    if (evictionThread) {
      this.evictionThread = new EvictionThread(this);
      this.evictionThread.start(); // FindBugs SC_START_IN_CTOR
    } else {
      this.evictionThread = null;
    }
    // every five minutes.
    this.scheduleThreadPool.scheduleAtFixedRate(new StatisticsThread(this), STAT_THREAD_PERIOD,STAT_THREAD_PERIOD, TimeUnit.SECONDS);
}
```