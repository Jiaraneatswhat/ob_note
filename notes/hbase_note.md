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
## 2.1 逻辑结构

![[HbaseLogic.svg]]
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
### 3.1.2 堆内内存 
#### 3.1.2.1 getOnHeapCacheInternal()
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
#### 3.1.2.2 EvictionThread 淘汰缓存
```java
// 实例化LruBlockCache时会创建并启动EvictionThread
public EvictionThread(LruBlockCache cache) {
      super(Thread.currentThread().getName() + ".LruBlockCache.EvictionThread");
      setDaemon(true);
      // 标记为弱引用，会被JVM更快回收
      this.cache = new WeakReference<>(cache);
}

public void run() {
      enteringRun = true;
      while (this.go) {
        LruBlockCache cache = this.cache.get();
        if (cache == null) break;
        cache.evict();
      }
}

void evict() {
    // Ensure only one eviction at a time
    // 确保只有一个eviction在执行
    if(!evictionLock.tryLock()) return;

    try {
      evictionInProgress = true;
      long currentSize = this.size.get();
      // 应该释放的缓冲大小
      long bytesToFree = currentSize - minSize();
      // Instantiate priority buckets
      // BlockBucket底层是LruCachedBlockQueue队列
      // 实例化三个队列
      BlockBucket bucketSingle = new BlockBucket("single", bytesToFree, blockSize, singleSize());
      BlockBucket bucketMulti = new BlockBucket("multi", bytesToFree, blockSize, multiSize());
      BlockBucket bucketMemory = new BlockBucket("memory", bytesToFree, blockSize, memorySize());

      // Scan entire map putting into appropriate buckets
      // 遍历block，根据优先级添加到不同的队列
      for (LruCachedBlock cachedBlock : map.values()) {
        switch (cachedBlock.getPriority()) {
          case SINGLE: {
            bucketSingle.add(cachedBlock);
            break;
          }
          case MULTI: {
            bucketMulti.add(cachedBlock);
            break;
          }
          case MEMORY: {
            bucketMemory.add(cachedBlock);
            break;
          }
        }
      }

      long bytesFreed = 0;
      // 如果memoryFactor或者InMemory缓存超过99.9%
      if (forceInMemory || memoryFactor > 0.999f) {
        long s = bucketSingle.totalSize();
        long m = bucketMulti.totalSize();
        // 需要回收的内存大于single和multi大小的和，全部回收
        if (bytesToFree > (s + m)) {
          // this means we need to evict blocks in memory bucket to make room,
          // so the single and multi buckets will be emptied
          bytesFreed = bucketSingle.free(s);
          bytesFreed += bucketMulti.free(m);
          if (LOG.isTraceEnabled()) {
            LOG.trace("freed " + StringUtils.byteDesc(bytesFreed) +
              " from single and multi buckets");
          }
          // 剩余的从mem中释放
          bytesFreed += bucketMemory.free(bytesToFree - bytesFreed);
          if (LOG.isTraceEnabled()) {
            LOG.trace("freed " + StringUtils.byteDesc(bytesFreed) +
              " total from all three buckets ");
          }
        } else {
          // this means no need to evict block in memory bucket,
          // and we try best to make the ratio between single-bucket and
          // multi-bucket is 1:2
          // s : m = 1 : 2
          long bytesRemain = s + m - bytesToFree;
          // s或m的大小不够，就去对方的队列释放内存
          if (3 * s <= bytesRemain) {
            // single-bucket is small enough that no eviction happens for it
            // hence all eviction goes from multi-bucket
            bytesFreed = bucketMulti.free(bytesToFree);
          } else if (3 * m <= 2 * bytesRemain) {
            // multi-bucket is small enough that no eviction happens for it
            // hence all eviction goes from single-bucket
            bytesFreed = bucketSingle.free(bytesToFree);
          } else {
            // both buckets need to evict some blocks
            bytesFreed = bucketSingle.free(s - bytesRemain / 3);
            if (bytesFreed < bytesToFree) {
              bytesFreed += bucketMulti.free(bytesToFree - bytesFreed);
            }
          }
        }
        // 未超过0.99, 从三个队列循环释放
      } else {
        PriorityQueue<BlockBucket> bucketQueue = new PriorityQueue<>(3);

        bucketQueue.add(bucketSingle);
        bucketQueue.add(bucketMulti);
        bucketQueue.add(bucketMemory);

        int remainingBuckets = 3;

        BlockBucket bucket;
        while ((bucket = bucketQueue.poll()) != null) {
          long overflow = bucket.overflow();
          if (overflow > 0) {
            long bucketBytesToFree =
                Math.min(overflow, (bytesToFree - bytesFreed) / remainingBuckets);
            bytesFreed += bucket.free(bucketBytesToFree);
          }
          remainingBuckets--;
        }
      }
    }
}

public long free(long toFree) {
      LruCachedBlock cb;
      long freedBytes = 0;
      // 从LruCachedBlockQueue中取出block进行释放
      while ((cb = queue.pollLast()) != null) {
        freedBytes += evictBlock(cb, true);
        if (freedBytes >= toFree) {
          return freedBytes;
        }
      }
      return freedBytes;
}
```
#### 3.1.2.3 LruBlockCache 缓存块
```java

/* LruBlockCache通过三种Factor，将缓存分为三个区域：single-access, multi-access以及in-memory
 * single占 0.25，表示单次读取区，block被读取后会先放在这个区域，当多次被读取后会升级到下一个区域
 * multi占 0.5，表示多次读取区
 * in-memory占 0.25，只存放设置了IN-MEMORY=true的列族中读取的block
 */

public void cacheBlock(BlockCacheKey cacheKey, Cacheable buf, boolean inMemory) {
    // 需要缓存的block的大小超过了最大块大小
    if (buf.heapSize() > maxBlockSize) {
      if (stats.failInsert() % 50 == 0) {
      return;
    }
	// 从缓存map中根据cacheKey尝试获取已缓存数据块
    LruCachedBlock cb = map.get(cacheKey);
    // 缓存中有，比较缓存的内容
    if (cb != null && !BlockCacheUtil.shouldReplaceExistingCacheBlock(this, cacheKey, buf)) {
      return;
    }
    // 当前缓存大小
    long currentSize = size.get();
    // 可接受缓存大小
    // (long)Math.floor(this.maxSize * this.acceptableFactor);
    long currentAcceptableSize = acceptableSize();
    // hardLimit
    long hardLimitSize = (long) (hardCapacityLimitFactor * currentAcceptableSize);
    // 通过cacheKry创建缓存块
    cb = new LruCachedBlock(cacheKey, buf, count.incrementAndGet(), inMemory);
    long newSize = updateSizeMetrics(cb, false);
    // 将映射信息添加到map中
    map.put(cacheKey, cb);
    // 如果新大小超过当前可以接受的大小，且未执行回收过程中
    if (newSize > currentAcceptableSize && !evictionInProgress) {
      // 回收内存
      runEviction();
    }
  }
```
### 3.1.3 堆外内存
#### 3.1.3.1 getBucketCache()
```java
// L2 Cache
// 将缓存划分为一个个的Bucket，每个Bucket都贴上一个size标签，将Block缓存在最接近且小于size的bucket中
// size的分类在启动时确定，默认有(8+1)K、(16+1)K、(32+1)K、(40+1)K、(48+1)K、(56+1)K、(64+1)K、(96+1)K … (512+1)K，相同size标签的Bucket由同一个BucketSizeInfo管理
// 每个Bucket默认大小为2M
static BucketCache getBucketCache(Configuration c) {
    // Check for L2.  ioengine name must be non-null.
    // 通过hbase.bucketcache.ioengine进行设置：heap, offheap 或者 file 
    String bucketCacheIOEngineName = c.get(BUCKET_CACHE_IOENGINE_KEY, null);
    
    // DEFAULT_BUCKET_CACHE_WRITER_THREADS = 3;
    // DEFAULT_BUCKET_CACHE_WRITER_QUEUE = 64;
    int writerThreads = c.getInt(BUCKET_CACHE_WRITER_THREADS_KEY,
      DEFAULT_BUCKET_CACHE_WRITER_THREADS);
    int writerQueueLen = c.getInt(BUCKET_CACHE_WRITER_QUEUE_KEY,
      DEFAULT_BUCKET_CACHE_WRITER_QUEUE);
    
    // "hbase.bucketcache.bucket.sizes"
    String[] configuredBucketSizes = c.getStrings(BUCKET_CACHE_BUCKETS_KEY);
    int [] bucketSizes = null;
    if (configuredBucketSizes != null) {
      bucketSizes = new int[configuredBucketSizes.length];
      for (int i = 0; i < configuredBucketSizes.length; i++) {
        int bucketSize = Integer.parseInt(configuredBucketSizes[i].trim());
        // 确保bucketSize是256的倍数
        if (bucketSize % 256 != 0) {...}
        bucketSizes[i] = bucketSize;
      }
    }
    BucketCache bucketCache = null;
    try {
      int ioErrorsTolerationDuration = c.getInt(
      // 创建BucketCache
      bucketCache = new BucketCache(bucketCacheIOEngineName,
        bucketCacheSize, blockSize, bucketSizes, writerThreads, writerQueueLen, persistentPath,
        ioErrorsTolerationDuration, c);
    }
    return bucketCache;
}
```
#### 3.1.3.2 new BucketCache()
```java
public BucketCache(...)
      throws FileNotFoundException, IOException {
    this.ioEngine = getIOEngineFromName(ioEngineName, capacity, persistencePath);
    this.writerThreads = new WriterThread[writerThreadNum];
    long blockNumCapacity = capacity / blockSize;
    
	// DEFAULT_ACCEPT_FACTOR = 0.95f;
    this.acceptableFactor = conf.getFloat(ACCEPT_FACTOR_CONFIG_NAME, DEFAULT_ACCEPT_FACTOR);
    // DEFAULT_MIN_FACTOR = 0.85f;
    this.minFactor = conf.getFloat(MIN_FACTOR_CONFIG_NAME, DEFAULT_MIN_FACTOR);
    // DEFAULT_EXTRA_FREE_FACTOR = 0.10f;
    this.extraFreeFactor = conf.getFloat(EXTRA_FREE_FACTOR_CONFIG_NAME, DEFAULT_EXTRA_FREE_FACTOR);
    this.singleFactor = conf.getFloat(SINGLE_FACTOR_CONFIG_NAME, DEFAULT_SINGLE_FACTOR);
    this.multiFactor = conf.getFloat(MULTI_FACTOR_CONFIG_NAME, DEFAULT_MULTI_FACTOR);
    this.memoryFactor = conf.getFloat(MEMORY_FACTOR_CONFIG_NAME, DEFAULT_MEMORY_FACTOR);
    
    this.cacheCapacity = capacity;
    this.persistencePath = persistencePath;
    this.blockSize = blockSize;
    this.ioErrorsTolerationDuration = ioErrorsTolerationDuration;
	// 对 Bucket 的组织管理，为 Block 分配内存空间
    bucketAllocator = new BucketAllocator(capacity, bucketSizes);
    for (int i = 0; i < writerThreads.length; ++i) {
      writerQueues.add(new ArrayBlockingQueue<>(writerQLen));
    }

    assert writerQueues.size() == writerThreads.length;
    // 创建一个ramCache的map，存储 blockKey 和 Block 对应关系
    // ConcurrentMap<BlockCacheKey, RAMQueueEntry>
    this.ramCache = new ConcurrentHashMap<>();
    
	// ConcurrentMap<BlockCacheKey, BucketEntry> backingMap
    this.backingMap = new ConcurrentHashMap<>((int) blockNumCapacity);

    final String threadName = Thread.currentThread().getName();
    this.cacheEnabled = true;
    for (int i = 0; i < writerThreads.length; ++i) {
      writerThreads[i] = new WriterThread(writerQueues.get(i));
      writerThreads[i].setName(threadName + "-BucketCacheWriter-" + i);
      writerThreads[i].setDaemon(true);
    }
    // 开启writer线程，异步地将 Block 写入到内存空间
    startWriterThreads();
}
```
#### 3.1.3.3 Bucket 和 BucketSizeInfo 类
```java
// Bucket是BucketAllocator的静态内部类
// 默认大小:  DEFAULT_BUCKET_SIZES[] = {4 * 1024 + 1024, 8 * 1024 + 1024, ..., 512 * 1024 + 1024 };

public final static class Bucket {
    // Bucket在实际物理空间中的起始地址，Block的地址通过baseoffset和该Block在Bucket的偏移量唯一确定
    private long baseOffset;

    public Bucket(long offset) {
      baseOffset = offset;
      sizeIndex = -1;
    }
}

// BucketSizeInfo也是BucketAllocator的静态内部类
final class BucketSizeInfo {
    // Free bucket means it has space to allocate a block;
    // Completely free bucket means it has no block.
    private LinkedMap bucketList, freeBuckets, completelyFreeBuckets;
    private int sizeIndex;

    BucketSizeInfo(int sizeIndex) {
      bucketList = new LinkedMap();
      freeBuckets = new LinkedMap();
      completelyFreeBuckets = new LinkedMap();
      this.sizeIndex = sizeIndex;
    }
}
```
#### 3.1.3.4 new BucketAllocator()
```java
BucketAllocator(long availableSpace, int[] bucketSizes)
      throws BucketAllocatorException {
    this.bucketSizes = bucketSizes == null ? DEFAULT_BUCKET_SIZES : bucketSizes;
    Arrays.sort(this.bucketSizes);
    this.bigItemSize = Ints.max(this.bucketSizes);
    //  FEWEST_ITEMS_IN_BUCKET = 4
    this.bucketCapacity = FEWEST_ITEMS_IN_BUCKET * (long) bigItemSize;
    // 创建Bucket数组
    buckets = new Bucket[(int) (availableSpace / bucketCapacity)];
    bucketSizeInfos = new BucketSizeInfo[this.bucketSizes.length];
    // 创建并初始化BucketSizeInfo数组
    for (int i = 0; i < this.bucketSizes.length; ++i) {
      bucketSizeInfos[i] = new BucketSizeInfo(i);
    }
    for (int i = 0; i < buckets.length; ++i) {
      // 初始化Bucket数组
      buckets[i] = new Bucket(bucketCapacity * i);
      bucketSizeInfos[i < this.bucketSizes.length ? i : this.bucketSizes.length - 1]
          .instantiateBucket(buckets[i]);
    }
    this.totalSize = ((long) buckets.length) * bucketCapacity;
  }
```
### 3.1.4 BucketCache 读写 Block 流程

![[BucketCache.svg]]

- 写流程
	- 将 `Block` 写入 `RamCache`，根据 `blockKey` 的 `hash` 值存储在不同的 `RamCache` 中
	- `WriteThread` 从 `RAMCache` 中取出所有的 `Block`，每个 `WriteTheead` 对应一个 `RAMCache`
	- 每个 `WriteTheead` 会遍历 `RAMCache` 中所有 `Block`，分别调用 `bucketAllocator` 为这些 `Block` 分配内存空间
	- `BucketAllocator` 会选择与 `Block` 大小对应的 `Bucket` 进行存放，并且返回对应的物理地址偏移量 `offset`
	- `WriteThread` 将 `Block` 以及分配好的 offset 传给 `IOEngine` 模块，执行具体的内存写入操作
	- 写入成功后，将 `blockKey` 和 `offset` 的对应关系存在 `BackingMap` 中，便于之后查找
- 读流程
	- 首先从 `RAMCache` 中查找，对于还未写入 `Bucket` 的缓存 `Block`，一定存储在 `RAMCache` 中
	- 在 `RAMCache` 中没有找到，根据 `blockKey` 去 `BackingMap` 中找 `offset`
	- 根据 `offset` 去内存中查找对应的 `block` 的数据
#### 3.1.4.1 cacheBlock()
```java
public void cacheBlock(BlockCacheKey cacheKey, Cacheable buf) {
    cacheBlock(cacheKey, buf, false);
}

public void cacheBlock(BlockCacheKey cacheKey, Cacheable cachedItem, boolean inMemory) {
    // wait_when_cache默认false
    cacheBlockWithWait(cacheKey, cachedItem, inMemory, wait_when_cache);
}

// Cache the block to ramCache
private void cacheBlockWithWait(BlockCacheKey cacheKey, Cacheable cachedItem, boolean inMemory,
      boolean wait) {
    if (cacheEnabled) {
      if (backingMap.containsKey(cacheKey) || ramCache.containsKey(cacheKey)) {
        // 缓存已存在，同lruCache判断是否需要替换
        if (BlockCacheUtil.shouldReplaceExistingCacheBlock(this, cacheKey, cachedItem)) {
          cacheBlockWithWaitInternal(cacheKey, cachedItem, inMemory, wait);
        }
      } else {
        cacheBlockWithWaitInternal(cacheKey, cachedItem, inMemory, wait);
      }
    }
}

private void cacheBlockWithWaitInternal(BlockCacheKey cacheKey, Cacheable cachedItem, boolean inMemory, boolean wait) {
    if (!cacheEnabled) {
      return;
    }
    LOG.trace("Caching key={}, item={}", cacheKey, cachedItem);
    // Stuff the entry into the RAM cache so it can get drained to the persistent store
    // 将需要写入的块封装成RAMQueueEntry
    RAMQueueEntry re =
        new RAMQueueEntry(cacheKey, cachedItem, accessCount.incrementAndGet(), inMemory);
    int queueNum = (cacheKey.hashCode() & 0x7FFFFFFF) % writerQueues.size();
    BlockingQueue<RAMQueueEntry> bq = writerQueues.get(queueNum);
    boolean successfulAddition = false;
    if (wait) {...}
     else {
      // 将Entry放在队列中，由BucketCache实例化时创建的writeThread处理
      successfulAddition = bq.offer(re);
    }
    ...
  }
```
#### 3.1.4.2 WriterThread.run()
```java
// WriterThread是BucketCache的内部类
public void run() {  
  List<RAMQueueEntry> entries = new ArrayList<>();  
  try {  
    while (cacheEnabled && writerEnabled) {  
      try {  
        try {  
          // 从队列中获取Blocks  
          entries = getRAMQueueEntries(inputQueue, entries);  
        }   
        doDrain(entries);  
      }
    }  
  }
}
    
static List<RAMQueueEntry> getRAMQueueEntries(final BlockingQueue<RAMQueueEntry> q,
      final List<RAMQueueEntry> receptacle)
  throws InterruptedException {
    // Clear sets all entries to null and sets size to 0. We retain allocations. Presume it
    // ok even if list grew to accommodate thousands.
    // 从q中取出block添加到receptacle(容器)
    receptacle.clear();
    receptacle.add(q.take());
    q.drainTo(receptacle);
    return receptacle;
}

void doDrain(final List<RAMQueueEntry> entries) throws InterruptedException {
      final int size = entries.size();
      BucketEntry[] bucketEntries = new BucketEntry[size];
      int index = 0;
      while (cacheEnabled && index < size) {
        RAMQueueEntry re = null;
        try {
          // 取到一个Entry
          re = entries.get(index);
          BucketEntry bucketEntry =
            // 将Entry写入缓存中
            re.writeToCache(ioEngine, bucketAllocator, deserialiserMap, realCacheSize);
          // Successfully added.  Up index and add bucketEntry. Clear io exceptions.
          bucketEntries[index] = bucketEntry;
        }
        
      for (int i = 0; i < size; ++i) {
        BlockCacheKey key = entries.get(i).getKey();
        // Only add if non-null entry.
        if (bucketEntries[i] != null) {
          putIntoBackingMap(key, bucketEntries[i]);
        }
}
          
public BucketEntry writeToCache(final IOEngine ioEngine,
        final BucketAllocator bucketAllocator,
        final UniqueIndexMap<Integer> deserialiserMap,
        final LongAdder realCacheSize) throws CacheFullException, IOException,
        BucketAllocatorException {
      // bucketAllocator给block在bucket中分配空间
      long offset = bucketAllocator.allocateBlock(len);
      BucketEntry bucketEntry = ioEngine.usesSharedMemory()
          ? UnsafeAvailChecker.isAvailable()
              ? new UnsafeSharedMemoryBucketEntry(offset, len, accessCounter, inMemory)
              : new SharedMemoryBucketEntry(offset, len, accessCounter, inMemory)
          : new BucketEntry(offset, len, accessCounter, inMemory);
      bucketEntry.setDeserialiserReference(data.getDeserializer(), deserialiserMap);
      try {
        if (data instanceof HFileBlock) {
          // If an instance of HFileBlock, save on some allocations.
          HFileBlock block = (HFileBlock)data;
          ByteBuff sliceBuf = block.getBufferReadOnly();
          ByteBuffer metadata = block.getMetaData();
          if (LOG.isTraceEnabled()) {
            LOG.trace("Write offset=" + offset + ", len=" + len);
          }
           // 通过ioEngine向内存写入数据
          ioEngine.write(sliceBuf, offset);
          ioEngine.write(metadata, offset + len - metadata.limit());
        } else {
          ByteBuffer bb = ByteBuffer.allocate(len);
          data.serialize(bb, true);
          ioEngine.write(bb, offset);
        }
      }
      return bucketEntry;
}      
```
#### 3.1.4.3 allocateBlock()
```java
public synchronized long allocateBlock(int blockSize) throws CacheFullException,Buc ketAllocatorException {
    BucketSizeInfo bsi = roundUpToBucketSizeInfo(blockSize);
    // 通过BucketSizeInfo分配空间返回offset
    long offset = bsi.allocateBlock();
    return offset;
}

public long allocateBlock() {
      Bucket b = null;
      // 存在空闲的bucket
      if (freeBuckets.size() > 0) {
        // Use up an existing one first...
        b = (Bucket) freeBuckets.lastKey();
      }
      if (b == null) {
        // 遍历BSI找到一个空闲的bucket
        b = grabGlobalCompletelyFreeBucket();
        // 找到后重新添加到free Map中
        if (b != null) instantiateBucket(b);
      }
      if (b == null) return -1;
      // 再通过bucket对象来分配空间
      long result = b.allocate();
      blockAllocated(b);
      return result;
}

public long allocate() {
      assert freeCount > 0; // Else should not have been called
      assert sizeIndex != -1;
      ++usedCount;
      // 更新offset
      long offset = baseOffset + (freeList[--freeCount] * itemAllocationSize);
      assert offset >= 0;
      return offset;
}
```
#### 3.1.4.4 读 Block 流程
```java
public Cacheable getBlock(BlockCacheKey key, boolean caching, boolean repeat,
      boolean updateCacheMetrics) {
    if (!cacheEnabled) {
      return null;
    }
    RAMQueueEntry re = ramCache.get(key);
    // 在cache中找到了，计算命中率后返回数据
    if (re != null) {
      if (updateCacheMetrics) {
        cacheStats.hit(caching, key.isPrimary(), key.getBlockType());
      }
      re.access(accessCount.incrementAndGet());
      return re.getData();
    }
    // 否则去backingMap中
    BucketEntry bucketEntry = backingMap.get(key);
    if (bucketEntry != null) {
      long start = System.nanoTime();
      ReentrantReadWriteLock lock = offsetLock.getLock(bucketEntry.offset());
      try {
        lock.readLock().lock();
        if (bucketEntry.equals(backingMap.get(key))) {
          // 获取偏移量
          int len = bucketEntry.getLength();
          if (LOG.isTraceEnabled()) {
            LOG.trace("Read offset=" + bucketEntry.offset() + ", len=" + len);
          }
          // 通过offset去内存中读取
          Cacheable cachedBlock = ioEngine.read(bucketEntry.offset(), len,
              bucketEntry.deserializerReference(this.deserialiserMap));
          ...
    // 未在BackingMap中找到返回null
    return null;
}
```
### 3.1.5 混合内存
```java
public class CombinedBlockCache implements ResizableBlockCache, HeapSize {
    protected final LruBlockCache onHeapCache;
    protected final BlockCache l2Cache;
    protected final CombinedCacheStats combinedCacheStats;

    public CombinedBlockCache(LruBlockCache onHeapCache, BlockCache l2Cache) {
        this.onHeapCache = onHeapCache;
        this.l2Cache = l2Cache;
        this.combinedCacheStats = new CombinedCacheStats(onHeapCache.getStats(),
                                                         l2Cache.getStats());
    }

    public void cacheBlock(BlockCacheKey cacheKey, Cacheable buf, boolean inMemory) {
        boolean metaBlock = buf.getBlockType().getCategory() != BlockCategory.DATA;
        if (metaBlock) {
            // 元数据的block存放在L1缓存中
            onHeapCache.cacheBlock(cacheKey, buf, inMemory);
        } else {
            l2Cache.cacheBlock(cacheKey, buf, inMemory);
        }
    }

    public Cacheable getBlock(BlockCacheKey cacheKey, boolean caching,
                              boolean repeat, boolean updateCacheMetrics) {
        // TODO: is there a hole here, or just awkwardness since in the lruCache getBlock
        // we end up calling l2Cache.getBlock.
        // We are not in a position to exactly look at LRU cache or BC as BlockType may not be getting
        // passed always.
        // 首先去L1缓存中找，找不到去L2缓存找
        return onHeapCache.containsBlock(cacheKey)?
            onHeapCache.getBlock(cacheKey, caching, repeat, updateCacheMetrics):
        l2Cache.getBlock(cacheKey, caching, repeat, updateCacheMetrics);
    }

    public boolean evictBlock(BlockCacheKey cacheKey) {
        // L1或L2缓存进行evict
        return onHeapCache.evictBlock(cacheKey) || l2Cache.evictBlock(cacheKey);
    }
```
## 3.2 WAL
- `AbstractFSWAL` 是对文件系统中 `WAL` 的实现，有两个子类：`FSHLog` 和 `AsyncFSWAL`
- 和 `FSLog` 不同，`AsyncFSWAL` 采用异步方式来写日志，效率更高
### 3.2.1 WALFactory 类
```java
public class WALFactory {
	
  // providers的类型，默认是AsyncFSWALProvider
  static enum Providers {
    defaultProvider(AsyncFSWALProvider.class),
    filesystem(FSHLogProvider.class),
    multiwal(RegionGroupingProvider.class),
    asyncfs(AsyncFSWALProvider.class);

  public static final String WAL_PROVIDER = "hbase.wal.provider";
  static final String DEFAULT_WAL_PROVIDER = Providers.defaultProvider.name();

  public static final String META_WAL_PROVIDER = "hbase.wal.meta_provider";

  final String factoryId;
  private final WALProvider provider;
  private final AtomicReference<WALProvider> metaProvider = new AtomicReference<>();
  private final Class<? extends AbstractFSWALProvider.Reader> logReaderClass;
  
 // provider相关的方法
  WALProvider createProvider(Class<? extends WALProvider> clazz, String providerId)
      throws IOException {...}
  WALProvider getProvider(String key, String defaultValue, String providerId) throws IOException {...}
  
  // 初始化
  public WALFactory(Configuration conf, String factoryId) throws IOException {
    // until we've moved reader/writer construction down into providers, this initialization must
    // happen prior to provider initialization, in case they need to instantiate a reader/writer.
    timeoutMillis = conf.getInt("hbase.hlog.open.timeout", 300000);
    /* TODO Both of these are probably specific to the fs wal provider */
    logReaderClass = conf.getClass("hbase.regionserver.hlog.reader.impl", ProtobufLogReader.class,
      AbstractFSWALProvider.Reader.class);
    this.conf = conf;
    this.factoryId = factoryId;
    // end required early initialization
    if (conf.getBoolean("hbase.regionserver.hlog.enabled", true)) {
      // AsyncFSWALProvider会创建AsyncFSWAL对象, 向日志中写数据
      provider = getProvider(WAL_PROVIDER, DEFAULT_WAL_PROVIDER, null);
    } else {...}
  }
  
  // WAL相关方法
  public List<WAL> getWALs() {...}
  public WAL getWAL(RegionInfo region) throws IOException {...}
  
  // Reader/Writer相关, Reader/Writer对HLog进行读写
  public Reader createReader(final FileSystem fs, final Path path) throws IOException {...}
  public Writer createWALWriter(final FileSystem fs, final Path path) throws IOException {...}
}
```
### 3.2.2 WAL 类
```java
// Write Ahead Log, provides service for reading, writing waledits, provides APIs for WAL users (such as RegionServer) to use the WAL (do append, sync, etc)
// 顶层接口，定义了操作WAL的api
public interface WAL extends Closeable, WALFileLengthProvider {

  byte[][] rollWriter() throws FailedLogCloseException, IOException;

  long append(RegionInfo info, WALKeyImpl key, WALEdit edits, boolean inMemstore) throws IOException;

  void sync() throws IOException;

  Long startCacheFlush(final byte[] encodedRegionName, Map<byte[], Long> familyToSeq);

  void completeCacheFlush(final byte[] encodedRegionName);
  
  // Reader的父接口 
  interface Reader extends Closeable {
    Entry next() throws IOException;
    Entry next(Entry reuse) throws IOException;
    void seek(long pos) throws IOException;
    long getPosition() throws IOException;
    void reset() throws IOException;
  }

  /**
   * Utility class that lets us keep track of the edit with it's key.
   */
  // Entry的父类
  class Entry {
    private final WALEdit edit;
    private final WALKeyImpl key;

    public Entry(WALKeyImpl key, WALEdit edit) {...}
	...
  }
}
```
## 3.3 HRegion
```java
public class HRegion implements HeapSize, PropagatingConfigurationObserver, Region {
    // 默认cell的最大大小10m
    public static final int DEFAULT_MAX_CELL_SIZE = 10485760;
    // 默认的Durability
    private static final Durability DEFAULT_DURABILITY = Durability.SYNC_WAL;
    // 存放store
    protected final Map<byte[], HStore> stores =
      new ConcurrentSkipListMap<>(Bytes.BYTES_RAWCOMPARATOR);
    private final WAL wal;
    ...
    
    // 判断region是否需要split
    public byte[] checkSplit() {...}
    // compact的操作的逻辑
    public void compact() {...} 
    // 操作WAL
    private WriteEntry doWALAppend(){...}
    // 写流程的一些方法
    private void doBatchMutate(Mutation mutation) throws IOException {...}
    private void doMiniBatchMutate(BatchOperation<?> batchOp) throws IOException {...}
    ...
}
```
## 3.4 HStore
```java
// HStore是HRegion的属性
//A Store holds a column family in a Region.  Its a memstore and a set of zero or more StoreFiles, which stretch backwards over time
public class HStore implements Store, HeapSize, StoreConfigInformation, PropagatingConfigurationObserver {
    public static final String DEFAULT_BLOCK_STORAGE_POLICY = "HOT";
    public static final int DEFAULT_COMPACTCHECKER_INTERVAL_MULTIPLIER = 1000;
    public static final int DEFAULT_BLOCKING_STOREFILE_COUNT = 16;
    // MemStore
    protected final MemStore memstore;
    
    protected HStore(final HRegion region, final ColumnFamilyDescriptor family,
      final Configuration confParam) throws IOException {
		
        // 实例化该store中使用的MemSrtore
        this.memstore = getMemstore();
    	this.storeEngine = createStoreEngine(this, this.conf, this.comparator);
    	// 根据列族名加载HStoreFile
    	List<HStoreFile> hStoreFiles = loadStoreFiles();
}
```
# 4. 写流程
## 4.1 客户端发起 put 请求
### 4.1.1 HTable.put()
```java
// HTable
public void put(final Put put) throws IOException {  
  // 检查写入的column是否存在，cell大小是否超过最大配置
  validatePut(put);  
  ClientServiceCallable<Void> callable =  
      new ClientServiceCallable<Void>(this.connection, getName(), put.getRow(), this.rpcControllerFactory.newController(), put.getPriority()) {  
    @Override  
    protected Void rpcCall() throws Exception {  
      MutateRequest request =            RequestConverter.buildMutateRequest(getLocation().getRegionInfo().getRegionName(), put);  
      // 使用protobuf协议序列化，提交RPC请求
      doMutate(request);  
      return null;  
    }  
  }; 
  // 调用callable的rpcCall方法
  rpcCallerFactory.<Void> newCaller(this.writeRpcTimeoutMs).callWithRetries(callable,  
      this.operationTimeoutMs);  
}

public T callWithRetries(RetryingCallable<T> callable, int callTimeout)  
throws IOException, RuntimeException {  
  List<RetriesExhaustedException.ThrowableWithExtraContext> exceptions = new ArrayList<>();  
  for (int tries = 0;; tries++) {  
    long expectedSleep;  
    try {
	  // 查找meta表位置  
      callable.prepare(tries != 0);  
      interceptor.intercept(context.prepare(callable, tries));  
      return callable.call(getTimeout(callTimeout));  
    }
```
### 4.1.2 定位 meta 表位置
```java
// RegionServerCallable.java
public void prepare(final boolean reload) throws IOException {  
  // tableNmae是要put数据的表
  // row是存储rowkey值的字节数组
  try (RegionLocator regionLocator = connection.getRegionLocator(tableName)) {  
    this.location = regionLocator.getRegionLocation(row);  
  }  
}
```
#### 4.1.2.1 getRegionLocator()
```java
public RegionLocator getRegionLocator(TableName tableName) throws IOException {  
  return new HRegionLocator(tableName, this);  
}

// private final ClusterConnection connection
public HRegionLocator(TableName tableName, ClusterConnection connection) {  
  this.connection = connection;  
  this.tableName = tableName;  
}
```
#### 4.1.2.2 getRegionLocation()
```java
public HRegionLocation getRegionLocation(final byte [] row)  
throws IOException {  
  return connection.getRegionLocation(tableName, row, false);  
}

public HRegionLocation getRegionLocation(final TableName tableName, final byte[] row,  
    boolean reload) throws IOException {  
  return reload ? relocateRegion(tableName, row) : locateRegion(tableName, row);  
}

public HRegionLocation locateRegion(final TableName tableName, final byte[] row)  
    throws IOException {  
  RegionLocations locations = locateRegion(tableName, row, true, true);  
  return locations == null ? null : locations.getRegionLocation();  
}

public RegionLocations locateRegion(final TableName tableName, final byte[] row, boolean useCache,  
    boolean retry) throws IOException {  
  return locateRegion(tableName, row, useCache, retry, RegionReplicaUtil.DEFAULT_REPLICA_ID);  
}

public RegionLocations locateRegion(final TableName tableName, final byte[] row, boolean useCache,  
    boolean retry, int replicaId) throws IOException {  
  if (tableName.equals(TableName.META_TABLE_NAME)) {  
    return locateMeta(tableName, useCache, replicaId);  
  } else { 
    // Region not in the cache - have to go to the meta RS  
    return locateRegionInMeta(tableName, row, useCache, retry, replicaId);  
  }  
}
```
#### 4.1.2.3 locateRegionInMeta()
```java
private RegionLocations locateRegionInMeta(TableName tableName, byte[] row, boolean useCache,  
    boolean retry, int replicaId) throws IOException {  
  // true
  if (useCache) {  
    RegionLocations locations = getCachedLocation(tableName, row);  
    if (locations != null && locations.getRegionLocation(replicaId) != null) {  
      return locations;  
    }  
  }  
  // build the key of the meta region we should be looking for.  
  // the extra 9's on the end are necessary to allow "exact" matches  // without knowing the precise region names.  byte[] metaStartKey = RegionInfo.createRegionName(tableName, row, HConstants.NINES, false);  
  byte[] metaStopKey =  
    RegionInfo.createRegionName(tableName, HConstants.EMPTY_START_ROW, "", false);  
  Scan s = new Scan().withStartRow(metaStartKey).withStopRow(metaStopKey, true)  
    .addFamily(HConstants.CATALOG_FAMILY).setReversed(true).setCaching(5)  
    .setReadType(ReadType.PREAD);  
  if (this.useMetaReplicas) {  
    s.setConsistency(Consistency.TIMELINE);  
  }  
  int maxAttempts = (retry ? numTries : 1);  
  for (int tries = 0; ; tries++) {  
    if (tries >= maxAttempts) {  
      throw new NoServerForRegionException("Unable to find region for "  
          + Bytes.toStringBinary(row) + " in " + tableName + " after " + tries + " tries.");  
    }  
    if (useCache) {  
      RegionLocations locations = getCachedLocation(tableName, row);  
      if (locations != null && locations.getRegionLocation(replicaId) != null) {  
        return locations;  
      }  
    } else {  
      // If we are not supposed to be using the cache, delete any existing cached location  
      // so it won't interfere.      // We are only supposed to clean the cache for the specific replicaId      metaCache.clearCache(tableName, row, replicaId);  
    }  
    // Query the meta region  
    long pauseBase = this.pause;  
    userRegionLock.lock();  
    try {  
      if (useCache) {// re-check cache after get lock  
        RegionLocations locations = getCachedLocation(tableName, row);  
        if (locations != null && locations.getRegionLocation(replicaId) != null) {  
          return locations;  
        }  
      }  
      s.resetMvccReadPoint();  
      try (ReversedClientScanner rcs =  
        new ReversedClientScanner(conf, s, TableName.META_TABLE_NAME, this, rpcCallerFactory,  
          rpcControllerFactory, getMetaLookupPool(), metaReplicaCallTimeoutScanInMicroSecond)) {  
        boolean tableNotFound = true;  
        for (;;) {  
          Result regionInfoRow = rcs.next();  
          if (regionInfoRow == null) {  
            if (tableNotFound) {  
              throw new TableNotFoundException(tableName);  
            } else {  
              throw new IOException(  
                "Unable to find region for " + Bytes.toStringBinary(row) + " in " + tableName);  
            }  
          }  
          tableNotFound = false;  
          // convert the row result into the HRegionLocation we need!  
          RegionLocations locations = MetaTableAccessor.getRegionLocations(regionInfoRow);  
          if (locations == null || locations.getRegionLocation(replicaId) == null) {  
            throw new IOException("RegionInfo null in " + tableName + ", row=" + regionInfoRow);  
          }  
          RegionInfo regionInfo = locations.getRegionLocation(replicaId).getRegion();  
          if (regionInfo == null) {  
            throw new IOException("RegionInfo null or empty in " + TableName.META_TABLE_NAME +  
              ", row=" + regionInfoRow);  
          }  
          // See HBASE-20182. It is possible that we locate to a split parent even after the  
          // children are online, so here we need to skip this region and go to the next one.          if (regionInfo.isSplitParent()) {  
            continue;  
          }  
          if (regionInfo.isOffline()) {  
            throw new RegionOfflineException("Region offline; disable table call? " +  
                regionInfo.getRegionNameAsString());  
          }  
          // It is possible that the split children have not been online yet and we have skipped  
          // the parent in the above condition, so we may have already reached a region which does          // not contains us.          if (!regionInfo.containsRow(row)) {  
            throw new IOException(  
              "Unable to find region for " + Bytes.toStringBinary(row) + " in " + tableName);  
          }  
          ServerName serverName = locations.getRegionLocation(replicaId).getServerName();  
          if (serverName == null) {  
            throw new NoServerForRegionException("No server address listed in " +  
              TableName.META_TABLE_NAME + " for region " + regionInfo.getRegionNameAsString() +  
              " containing row " + Bytes.toStringBinary(row));  
          }  
          if (isDeadServer(serverName)) {  
            throw new RegionServerStoppedException(  
              "hbase:meta says the region " + regionInfo.getRegionNameAsString() +  
                " is managed by the server " + serverName + ", but it is dead.");  
          }  
          // Instantiate the location  
          cacheLocation(tableName, locations);  
          return locations;  
        }  
      }  
    } catch (TableNotFoundException e) {  
      // if we got this error, probably means the table just plain doesn't  
      // exist. rethrow the error immediately. this should always be coming      // from the HTable constructor.      throw e;  
    } catch (IOException e) {  
      ExceptionUtil.rethrowIfInterrupt(e);  
      if (e instanceof RemoteException) {  
        e = ((RemoteException)e).unwrapRemoteException();  
      }  
      if (e instanceof CallQueueTooBigException) {  
        // Give a special check on CallQueueTooBigException, see #HBASE-17114  
        pauseBase = this.pauseForCQTBE;  
      }  
      if (tries < maxAttempts - 1) {  
        LOG.debug("locateRegionInMeta parentTable='{}', attempt={} of {} failed; retrying " +  
          "after sleep of {}", TableName.META_TABLE_NAME, tries, maxAttempts, maxAttempts, e);  
      } else {  
        throw e;  
      }  
      // Only relocate the parent region if necessary  
      if(!(e instanceof RegionOfflineException ||  
          e instanceof NoServerForRegionException)) {  
        relocateRegion(TableName.META_TABLE_NAME, metaStartKey, replicaId);  
      }  
    } finally {  
      userRegionLock.unlock();  
    }  
    try{  
      Thread.sleep(ConnectionUtils.getPauseTime(pauseBase, tries));  
    } catch (InterruptedException e) {  
      throw new InterruptedIOException("Giving up trying to location region in " +  
        "meta: thread is interrupted.");  
    }  
  }  
}
```
### 4.1.3 HRegion.put()
```java
public void put(Put put) throws IOException {  
  checkReadOnly();  
 
  startRegionOperation(Operation.PUT);  
  try {  
    doBatchMutate(put);  
  } finally {  
    closeRegionOperation(Operation.PUT);  
  }  
}
```