# 1 启动
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
# 2 存储结构
## 2.1 逻辑结构

![[HbaseLogic.svg]]
## 2.2 术语
- namespace
	- 类似于 `SQL` 中的 `database`
	- `HBase` 自带两个 `namespace`，`hbase` 和 `default`，`hbase` 存放的是 `HBase` 的内置表，`default` 是用户默认使用的 `namespace`
- table
- row：每行数据由一个 `RowKey` 和多个 `Column` 组成
- rowkey：`rowkey` 是行的唯一标识，读写都需要通过 `rowkey` 来指定
- column family：别名 `store`，太多的列族会降低性能
- cell
	- 一个列的一个版本
	- 由 `rowkey, column Family：column Qualifier, time Stamp` 唯一确定的单元
	- `cell` 中的数据全部是字节码形式存储
- region
	- 一张表若干连续的行形成的区域
	- `Region` 中行的排序按照 `rowkey` 字典排序
	- `Region` 不能跨 `RegionSever`，且当数据量大的时候，`HBase` 会拆分 `Region`
- 在 HBase 中一个列族和一个列修饰符组合起来(列族：列修饰符)称为一个列
## 2.3 HFlie 结构

![[HFile.svg]]
# 3 RegionServer 架构

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
	- 每个 `WriteTheead` 会遍历 `RAMCache` 中所有 `Block`，分别调用 `BucketAllocator` 为这些 `Block` 分配内存空间
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
    
    protected HStore(final HRegion region, final ColumnFamilyDescriptor family, final Configuration confParam) throws IOException {
		
        // 实例化该store中使用的MemSrtore
        this.memstore = getMemstore();
    	this.storeEngine = createStoreEngine(this, this.conf, this.comparator);
    	// 根据列族名加载HStoreFile
    	List<HStoreFile> hStoreFiles = loadStoreFiles();
}
```
# 4 写流程
## 4.1 客户端发起 put 请求

![[write1.svg]]
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
      // Mutation是Put, Delete等对象的父类
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
	  // ClientServiceCallable 继承了 RegionServerCallable，会调用父类的 prepare 方法
      callable.prepare(tries != 0);  
      interceptor.intercept(context.prepare(callable, tries));  
      return callable.call(getTimeout(callTimeout));  
    }
```
## 4.2 定位 meta 表位置
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
### 4.2.1 getRegionLocator()
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
### 4.2.2 getRegionLocation()
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
### 4.2.3 locateRegionInMeta()
```java
private RegionLocations locateRegionInMeta(TableName tableName, byte[] row, boolean useCache,  
    boolean retry, int replicaId) throws IOException {  
  // true
  if (useCache) {  
    RegionLocations locations = getCachedLocation(tableName, row);
    // 从缓存中读取到直接返回
    if (locations != null && locations.getRegionLocation(replicaId) != null) {  
      return locations;  
    }  
  }  
  // 通过表名，row_key 和 HConstants.NINES 创建 RegionName，作为meta的startKey
   byte[] metaStartKey = RegionInfo.createRegionName(tableName, row, HConstants.NINES, false);  
  byte[] metaStopKey =  
    RegionInfo.createRegionName(tableName, HConstants.EMPTY_START_ROW, "", false);  
  Scan s = new Scan().withStartRow(metaStartKey).withStopRow(metaStopKey, true)  
    .addFamily(HConstants.CATALOG_FAMILY).setReversed(true).setCaching(5) 
    .setReadType(ReadType.PREAD);
  // numTries = 16
  int maxAttempts = (retry ? numTries : 1);  
  for (int tries = 0; ; tries++) {  
    if (useCache) {  
      // 再次去缓存里读
      RegionLocations locations = getCachedLocation(tableName, row);  
      if (locations != null && locations.getRegionLocation(replicaId) != null) {  
        return locations;  
      }  
    }

	// 仍未找到，去meta表中找
	// 创建一个 ReversedClientScanner 对象，传入hbase:meta表名
    try (ReversedClientScanner rcs =  
        new ReversedClientScanner(conf, s, TableName.META_TABLE_NAME, this, rpcCallerFactory,  
          rpcControllerFactory, getMetaLookupPool(), metaReplicaCallTimeoutScanInMicroSecond)) {  
        boolean tableNotFound = true;  
        for (;;) {  
          Result regionInfoRow = rcs.next();   
          tableNotFound = false;  
          // convert the row result into the HRegionLocation we need!  
          RegionLocations locations = MetaTableAccessor.getRegionLocations(regionInfoRow);  
          RegionInfo regionInfo = locations.getRegionLocation(replicaId).getRegion();    
          ServerName serverName = locations.getRegionLocation(replicaId).getServerName();  
          // Instantiate the location  
          cacheLocation(tableName, locations);  
          return locations;  
        }  
      }  
    }  
  }  
}
```
### 4.2.4 getCachedLocation()
```java
// 在 cache 中寻找 TableName 和 row 的位置
RegionLocations getCachedLocation(final TableName tableName,  
    final byte [] row) {  
  return metaCache.getCachedLocation(tableName, row);  
}

// MetaCache.java
public RegionLocations getCachedLocation(final TableName tableName, final byte [] row) {  
  ConcurrentNavigableMap<byte[], RegionLocations> tableLocations =  
    getTableLocations(tableName);  
  
  Entry<byte[], RegionLocations> e = tableLocations.floorEntry(row);  
  if (e == null) {  
    if (metrics != null) metrics.incrMetaCacheMiss();  
    return null;  
  }  
  RegionLocations possibleRegion = e.getValue();  

  if (Bytes.equals(endKey, HConstants.EMPTY_END_ROW) ||  
      Bytes.compareTo(endKey, 0, endKey.length, row, 0, row.length) > 0) {  
    if (metrics != null) metrics.incrMetaCacheHit();  
    return possibleRegion;  
  }  
}

// 返回null说明没有缓存
private ConcurrentNavigableMap<byte[], RegionLocations> getTableLocations(  
    final TableName tableName) {  
  // find the map of cached locations for this table  
  return computeIfAbsent(cachedRegionLocations, tableName,  
    () -> new CopyOnWriteArrayMap<>(Bytes.BYTES_COMPARATOR));  
}
```
### 4.2.5 循环遍历 ReversedClientScanner
```java
public Result next() throws IOException {  
  return nextWithSyncCache();  
}

protected Result nextWithSyncCache() throws IOException {  
  loadCache();  
  return result;  
}

protected void loadCache() throws IOException {  
  long remainingResultSize = maxScannerResultSize;  
  for (;;) {  
    Result[] values;  
    try {      
      // ScannerCallableWithReplicas实现了callable接口，调用其call方法
      values = call(callable, caller, scannerTimeout, true);  
      }  
      retryAfterOutOfOrderException.setValue(true);  
    }
    long currentTime = System.currentTimeMillis();  
    if (this.scanMetrics != null) {  
      this.scanMetrics.sumOfMillisSecBetweenNexts.addAndGet(currentTime - lastNext);  
    }  
    lastNext = currentTime;  
    // Groom the array of Results that we received back from the server before adding that  
    // Results to the scanner's cache. If partial results are not allowed to be seen by the    // caller, all book keeping will be performed within this method.    int numberOfCompleteRowsBefore = scanResultCache.numberOfCompleteRows();  
    Result[] resultsToAddToCache =  
        scanResultCache.addAndGet(values, callable.isHeartbeatMessage());  
    int numberOfCompleteRows =  
        scanResultCache.numberOfCompleteRows() - numberOfCompleteRowsBefore;  
    for (Result rs : resultsToAddToCache) {  
      cache.add(rs);  
      this.lastResult = rs;  
    }   
  }  
}
```
### 4.2.6 call()
```java
// 父类ClientScanner的call
private Result[] call(ScannerCallableWithReplicas callable, RpcRetryingCaller<Result[]> caller,  
    int scannerTimeout, boolean updateCurrentRegion) throws IOException { 
  Result[] rrs = caller.callWithoutRetries(callable, scannerTimeout);  
  return rrs;  
}

public T callWithoutRetries(RetryingCallable<T> callable, int callTimeout)  
throws IOException, RuntimeException {  
   try {  
    callable.prepare(false);  
    return callable.call(callTimeout);  
  }
}
// 第二次进入4.1.2.2中的locateMeta方法时，此时表名为meta，会执行locateMeta方法

private RegionLocations locateMeta(final TableName tableName,  
    boolean useCache, int replicaId) throws IOException {  
  // only one thread should do the lookup.  
  synchronized (metaRegionLock) {  
    // Look up from zookeeper  
    locations = get(this.registry.getMetaRegionLocation());  
    if (locations != null) {  
	  // 查找结束后进行缓存
      cacheLocation(tableName, locations);  
    }  
  }  
  return locations;  
}
```
### 4.2.7 zk 的作用
#### 4.2.7.1 ZNodePaths
```java
/*
 * HBase在zk创建节点信息对应的类是ZNodePaths，启动Master或RegionServer节点时会创建ZKWatcher对象，初始化ZNodePaths
*/
public ZNodePaths(Configuration conf) {  
  // "/hbase"
  baseZNode = conf.get(ZOOKEEPER_ZNODE_PARENT, DEFAULT_ZOOKEEPER_ZNODE_PARENT);  
   
  // metaServerZNode(元数据服务器地址) --> /hbase/meta-region-server
  metaReplicaZNodes = builder.build();  
  // rsZNode --> /hbase/rs
  rsZNode = joinZNode(baseZNode, conf.get("zookeeper.znode.rs", "rs"));  
  // masterAddressZNode --> /hbase/master
  masterAddressZNode = joinZNode(baseZNode, conf.get("zookeeper.znode.master", "master"));  
  // 备用master --> /hbase/backup-masters-0
  backupMasterAddressesZNode =  
      joinZNode(baseZNode, conf.get("zookeeper.znode.backup.masters", "backup-masters"));  
 // clusterStateZNode --> /hbase/running
  clusterStateZNode = joinZNode(baseZNode, conf.get("zookeeper.znode.state", "running"));  
  // tableZNode --> /hbase/table
  tableZNode = joinZNode(baseZNode, conf.get("zookeeper.znode.tableEnableDisable", "table"));  
  // splitLogZNode --> /hbase/splitWAL
  splitLogZNode = joinZNode(baseZNode, conf.get("zookeeper.znode.splitlog", SPLIT_LOGDIR_NAME));  
  // namespaceZNode --> /hbase/namespace
  namespaceZNode = joinZNode(baseZNode, conf.get("zookeeper.znode.namespace", "namespace"));  
}
// 在zk上创建节点
private void createBaseZNodes() throws ZooKeeperConnectionException { 
try { 
	// Create all the necessary "directories" of znodes 
	ZKUtil.createWithParents(this, znodePaths.baseZNode); //createAndFailSilent创建Zookeeper节点，节点存在则忽略 
	ZKUtil.createAndFailSilent(this, znodePaths.rsZNode);               
	ZKUtil.createAndFailSilent(this, znodePaths.drainingZNode);  
	ZKUtil.createAndFailSilent(this, znodePaths.tableZNode); 
	ZKUtil.createAndFailSilent(this, znodePaths.splitLogZNode); 
	ZKUtil.createAndFailSilent(this, znodePaths.backupMasterAddressesZNode); 
	ZKUtil.createAndFailSilent(this, znodePaths.tableLockZNode); 
	ZKUtil.createAndFailSilent(this, znodePaths.masterMaintZNode); 
}
```
#### 4.2.7.2 节点信息
- `meta-region-server`
	- 存储 `HBase` 集群 `hbase:meta` 元数据表所在的 `RegionServer` 访问地址，客户端读写数据首先会从此节点读取 `hbase:meta` 元数据的访问地址
- `backup-masters`：HA 相关
- `table`：集群中所有表的信息，创建表后进行表的信息存储
- `master`：当前 `ative master` 节点
- `namespace`：做表空间的逻辑隔离
- `rs`：集群中所有运行的 `RegionServer`，`Master` 节点也会监听此节点信息
## 4.3 扫描 meta 表，获取数据要写到哪个 RS
### 4.3.1 从 zk 上查找
```java
// ZKAsyncRegistry.java
public CompletableFuture<RegionLocations> getMetaRegionLocation() {  
  CompletableFuture<RegionLocations> future = new CompletableFuture<>();  
  // metaReplicaZNodes = {0=/hbase/meta-region-server}
  HRegionLocation[] locs = new HRegionLocation[znodePaths.metaReplicaZNodes.size()];  
  MutableInt remaining = new MutableInt(locs.length);  
  znodePaths.metaReplicaZNodes.forEach((replicaId, path) -> {  
    if (replicaId == DEFAULT_REPLICA_ID) {  
      addListener(getAndConvert(path, ZKAsyncRegistry::getMetaProto), (proto, error) -> {  
        // 从HBaseProtos中获取到server地址和端口
        locs[DEFAULT_REPLICA_ID] = new HRegionLocation(  
          getRegionInfoForDefaultReplica(FIRST_META_REGIONINFO), stateAndServerName.getSecond());  
        tryComplete(remaining, locs, future);  
      });  
    } 
  });  
  return future;  
}
```
### 4.3.2 cacheLocation()
```java
public void cacheLocation(final TableName tableName, final RegionLocations location) {  
  metaCache.cacheLocation(tableName, location);  
}

public void cacheLocation(final TableName tableName, final RegionLocations locations) {  
  byte [] startKey = locations.getRegionLocation().getRegion().getStartKey();  
  ConcurrentMap<byte[], RegionLocations> tableLocations = getTableLocations(tableName);  
  RegionLocations oldLocation = tableLocations.putIfAbsent(startKey, locations);  
  boolean isNewCacheEntry = (oldLocation == null);  
  if (isNewCacheEntry) {    
    addToCachedServers(locations);  
    return;  
  }  
}

private void addToCachedServers(RegionLocations locations) {  
  for (HRegionLocation loc : locations.getRegionLocations()) {  
    if (loc != null) {  
      // 添加到集合中
      cachedServers.add(loc.getServerName());  
    }  
  }  
}
```
## 4.4 ClientServiceCallable.doMutate()
```java
protected ClientProtos.MutateResponse doMutate(ClientProtos.MutateRequest request)  
throws org.apache.hbase.thirdparty.com.google.protobuf.ServiceException {  
  return getStub().mutate(getRpcController(), request);  
}

// 服务端Ipc实现类是RSRpcServices
public MutateResponse mutate(final RpcController rpcc,  
    final MutateRequest request) throws ServiceException {
    type = mutation.getMutateType();
    try {
	    // 将请求反序列化为PUT对象
		case PUT:  
			Put put = ProtobufUtil.toPut(mutation, cellScanner);  
		     // 有协处理器
		    if (request.hasCondition()) {...}  
		} else {  
		    // 调用HReigon的put方法
		    region.put(put);  
		    processed = Boolean.TRUE;  
		  }  
		  break;
	    
public void put(Put put) throws IOException {  

  checkReadOnly();  
  // 检查memstore空间
  checkResources();
  startRegionOperation(Operation.PUT);  
  try {  
    doBatchMutate(put);  
  }  
}

void checkResources() throws RegionTooBusyException {  
  // If catalog region, do not impose resource constraints or block updates.  
  if (this.getRegionInfo().isMetaRegion()) return;  
  
  MemStoreSize mss = this.memStoreSizing.getMemStoreSize();  
  if (mss.getHeapSize() + mss.getOffHeapSize() > this.blockingMemStoreSize) {  
	// 超过memstore大小时flush
	/*
	 * 将FlushRequest添加到队列中，MemStoreFlusher通过FlushHandler进行flush，最后调用HRegion中的flushCache方法
	*/
    requestFlush();  
  }  
}
```
### 4.4.1 doBatchMutate()
```java
private void doBatchMutate(Mutation mutation) throws IOException {  
  // Currently this is only called for puts and deletes, so no nonces.  
  OperationStatus[] batchMutate = this.batchMutate(new Mutation[]{mutation});   
}

public OperationStatus[] batchMutate(Mutation[] mutations) throws IOException {  
  return batchMutate(mutations, HConstants.NO_NONCE, HConstants.NO_NONCE);  
}

public OperationStatus[] batchMutate(Mutation[] mutations, long nonceGroup, long nonce)  
    throws IOException {  
  return batchMutate(mutations, false, nonceGroup, nonce);  
}

public OperationStatus[] batchMutate(Mutation[] mutations, boolean atomic, long nonceGroup,  
    long nonce) throws IOException {  
  // As it stands, this is used for 3 things  
  //  * batchMutate with single mutation - put/delete, separate or from checkAndMutate.  
  //  * coprocessor calls (see ex. BulkDeleteEndpoint).  
  // So nonces are not really ever used by HBase. They could be by coprocs, and checkAnd...  
  return batchMutate(new MutationBatchOperation(this, mutations, atomic, nonceGroup, nonce));  
}

OperationStatus[] batchMutate(BatchOperation<?> batchOp) throws IOException {  
  boolean initialized = false;  
  batchOp.startRegionOperation();  
  try {  
    while (!batchOp.isDone()) {  
      doMiniBatchMutate(batchOp);  
      requestFlushIfNeeded();  
    }  
  }
  return batchOp.retCodeDetails;  
}
```
### 4.4.2 doMiniBatchMutate()
```java
private void doMiniBatchMutate(BatchOperation<?> batchOp) throws IOException {  
  boolean success = false;  
  WALEdit walEdit = null;  
  WriteEntry writeEntry = null;  
  boolean locked = false;  
  // We try to set up a batch in the range [batchOp.nextIndexToProcess,lastIndexExclusive)  
  MiniBatchOperationInProgress<Mutation> miniBatchOp = null;  
  /** Keep track of the locks we hold so we can release them in finally clause */  
  List<RowLock> acquiredRowLocks = Lists.newArrayListWithCapacity(batchOp.size());  
  try {  
    // STEP 1. Try to acquire as many locks as we can and build mini-batch of operations with  
    // locked rows    
    miniBatchOp = batchOp.lockRowsAndBuildMiniBatch(acquiredRowLocks);  
    }  
  
    lock(this.updatesLock.readLock(), miniBatchOp.getReadyToWriteCount());  
    locked = true;  
  
    // STEP 2. 更新操作的时间戳为最新
    long now = EnvironmentEdgeManager.currentTime();  
    batchOp.prepareMiniBatchOperations(miniBatchOp, now, acquiredRowLocks);  
  
    // STEP 3. Build WAL edit  
    List<Pair<NonceKey, WALEdit>> walEdits = batchOp.buildWALEdits(miniBatchOp);  
  
    // STEP 4. Append the WALEdits to WAL and sync.  
    // 遍历WALEdits，执行append
    for(Iterator<Pair<NonceKey, WALEdit>> it = walEdits.iterator(); it.hasNext();) {  
      if (walEdit != null && !walEdit.isEmpty()) {  
        writeEntry = doWALAppend(walEdit, batchOp.durability, batchOp.getClusterIds(), now, nonceKey.getNonceGroup(), nonceKey.getNonce(), batchOp.getOrigLogSeqNum());  
      }  
  
      // Complete mvcc for all but last writeEntry (for replay case)  
      if (it.hasNext() && writeEntry != null) {  
        mvcc.complete(writeEntry);  
        writeEntry = null;  
      }  
    }  
  
    // STEP 5. Write back to memStore  
    // NOTE: writeEntry can be null here    
    writeEntry = batchOp.writeMiniBatchOperationsToMemStore(miniBatchOp, writeEntry);  
  
    // STEP 6. Complete MiniBatchOperations: If required calls postBatchMutate() CP hook and  
    // complete mvcc for last writeEntry    
    batchOp.completeMiniBatchOperations(miniBatchOp, writeEntry);  
    writeEntry = null;  
    success = true;  
  } finally {  
    // Call complete rather than completeAndWait because we probably had error if walKey != null  
    if (writeEntry != null) mvcc.complete(writeEntry);  
  
    if (locked) {  
      this.updatesLock.readLock().unlock();  
    }  
    releaseRowLocks(acquiredRowLocks);  
  
    final int finalLastIndexExclusive =  
        miniBatchOp != null ? miniBatchOp.getLastIndexExclusive() : batchOp.size();  
    final boolean finalSuccess = success;  
    batchOp.visitBatchOperations(true, finalLastIndexExclusive, (int i) -> {  
      batchOp.retCodeDetails[i] =  
          finalSuccess ? OperationStatus.SUCCESS : OperationStatus.FAILURE;  
      return true;  
    });  
  
    batchOp.doPostOpCleanupForMiniBatch(miniBatchOp, walEdit, finalSuccess);  
  
    batchOp.nextIndexToProcess = finalLastIndexExclusive;  
  }  
}
```
#### 4.4.2.1 lockRowsAndBuildMiniBatch()
```java
// 对BatchOp加锁，返回MiniBatch
public MiniBatchOperationInProgress<Mutation> lockRowsAndBuildMiniBatch(  
    List<RowLock> acquiredRowLocks) throws IOException {  
  int readyToWriteCount = 0;  
  int lastIndexExclusive = 0;  
  RowLock prevRowLock = null;  
   
    Mutation mutation = getMutation(lastIndexExclusive);  
    // If we haven't got any rows in our batch, we should block to get the next one.  
    RowLock rowLock = null;  
    try {  
      // if atomic then get exclusive lock, else shared lock  
      // 返回一个RowLockImpl
      rowLock = region.getRowLockInternal(mutation.getRow(), !isAtomic(), prevRowLock);  
    }  
  }  
  return createMiniBatch(lastIndexExclusive, readyToWriteCount);  
}

protected MiniBatchOperationInProgress<Mutation> createMiniBatch(final int lastIndexExclusive,  
    final int readyToWriteCount) {  
  return new MiniBatchOperationInProgress<>(getMutationsForCoprocs(), retCodeDetails, walEditsFromCoprocessors, nextIndexToProcess, lastIndexExclusive, readyToWriteCount);  
}
```
#### 4.4.2.2 创建 WALEdits
```java
public List<Pair<NonceKey, WALEdit>> buildWALEdits(  
    final MiniBatchOperationInProgress<Mutation> miniBatchOp) throws IOException {  
  List<Pair<NonceKey, WALEdit>> walEdits = new ArrayList<>();  
  // 调用 Visitor 的 visit 方法
  visitBatchOperations(true, nextIndexToProcess + miniBatchOp.size(), new Visitor() {  
    private Pair<NonceKey, WALEdit> curWALEditForNonce;  
    @Override  
    public boolean visit(int index) throws IOException {  
      Mutation m = getMutation(index);  
      
  
      // the batch may contain multiple nonce keys (replay case). If so, write WALEdit for each.  
      // Given how nonce keys are originally written, these should be contiguous.      // They don't have to be, it will still work, just write more WALEdits than needed.      long nonceGroup = getNonceGroup(index);  
      long nonce = getNonce(index);  
      if (curWALEditForNonce == null ||  
          curWALEditForNonce.getFirst().getNonceGroup() != nonceGroup ||  
          curWALEditForNonce.getFirst().getNonce() != nonce) {  
        // 创建WALEdit对象添加到集合中
        curWALEditForNonce = new Pair<>(new NonceKey(nonceGroup, nonce),  
            new WALEdit(miniBatchOp.getCellCount(), isInReplay()));  
        walEdits.add(curWALEditForNonce);  
      }  
      WALEdit walEdit = curWALEditForNonce.getSecond();  
     
      // Add WAL edits from CPs.  
      // 添加来自协处理器的WALEdits
      WALEdit fromCP = walEditsFromCoprocessors[index];  
      if (fromCP != null) {  
        for (Cell cell : fromCP.getCells()) {  
          walEdit.add(cell);  
        }  
      }  
      walEdit.add(familyCellMaps[index]);  
  
      return true;  
    }  
  });  
  return walEdits;  
}
```
#### 4.4.2.3 写入 WAL: doWALAppend()
- HLog 的写入分为三个阶段
	- 将数据写入本地缓存
	- 将本地缓存写入文件系统
	- 执行 sync 操作同步到磁盘

![[write2.svg]]

```java
private WriteEntry doWALAppend(WALEdit walEdit, Durability durability, List<UUID> clusterIds,  
    long now, long nonceGroup, long nonce, long origLogSeqNum) throws IOException {  
	// WAL由WALKey和WALValue组成 
  WALKeyImpl walKey = walEdit.isReplay()?  // 是否通过replay产生的Edit
      new WALKeyImpl(...);  
  WriteEntry writeEntry = null;  
  try {  
    // append到HLog
    long txid = this.wal.append(this.getRegionInfo(), walKey, walEdit, true);  
    // Call sync on our edit.  
    if (txid != 0) {  
      sync(txid, durability);  
    }  
    writeEntry = walKey.getWriteEntry();  
  } 
  return writeEntry;  
}

// WAL的实现类AbstractFSWAL有两个子类FSHLog和AsyncFSWAL
// 都会调用父类的stampSequenceIdAndPublishToRingBuffer方法
protected final long stampSequenceIdAndPublishToRingBuffer(RegionInfo hri, WALKeyImpl key,  
    WALEdit edits, boolean inMemstore, RingBuffer<RingBufferTruck> ringBuffer)  
    throws IOException {   
  long txid = txidHolder.longValue();  
    FSWALEntry entry = new FSWALEntry(txid, key, edits, hri, inMemstore);  
    entry.stampRegionSequenceId(we);  
    ringBuffer.get(txid).load(entry);  
  } finally {  
    ringBuffer.publish(txid);  
  }  
  return txid;  
}
```
### 4.4.3 FSHLog 的 append
```java
// 将entry对象load进入DisruptorTruck中
void load(FSWALEntry entry) {  
  this.entry = entry;  
  this.type = Type.APPEND;  
}

// FSHLog中的RingBufferEventHandler会通过onEvent方法进行处理:
public void onEvent(final RingBufferTruck truck, final long sequence, boolean endOfBatch)  
    throws Exception {  
  
  try {  
    else if (truck.type() == RingBufferTruck.Type.APPEND) {  
      // 从truck中unload一个entry
      FSWALEntry entry = truck.unloadAppend();     
      try {  
        append(entry);  
		} 
	}
}

void append(final FSWALEntry entry) throws Exception {  
  try {  
    FSHLog.this.append(writer, entry);  
	  }   
}

protected final boolean append(W writer, FSWALEntry entry) throws IOException {  
  doAppend(writer, entry);  
  return true;  
}

protected void doAppend(Writer writer, FSWALEntry entry) throws IOException {  
  writer.append(entry);  
}

// ProtobufLogWriter.java
public void append(Entry entry) throws IOException {  
  entry.setCompressionContext(compressionContext);  
  entry.getKey().getBuilder(compressor).  
      setFollowingKvCount(entry.getEdit().size()).build().writeDelimitedTo(output);  
  for (Cell cell : entry.getEdit().getCells()) {  
    // cellEncoder must assume little about the stream, since we write PB and cells in turn.  
    cellEncoder.write(cell);  
  }  
  length.set(output.getPos());  
}

// 通过CellCodec将WALEntry写入Hadoop中
public void write(Cell cell) throws IOException {  
  checkFlushed();  
  // Row  
  write(cell.getRowArray(), cell.getRowOffset(), cell.getRowLength());  
  // Column family  
  write(cell.getFamilyArray(), cell.getFamilyOffset(), cell.getFamilyLength());  
  // Qualifier  
  write(cell.getQualifierArray(), cell.getQualifierOffset(), cell.getQualifierLength());  
  // Version  
  this.out.write(Bytes.toBytes(cell.getTimestamp()));  
  // Type  
  this.out.write(cell.getTypeByte());  
  // Value  
  write(cell.getValueArray(), cell.getValueOffset(), cell.getValueLength());  
  // MvccVersion  
  this.out.write(Bytes.toBytes(cell.getSequenceId()));  
}
```
### 4.4.4 AsyncFSWAL 的 sync
```java
// 通过append方法写入到文件后，通过sync方法将操作同步到磁盘
// 4.1.5.3
/*
 * HLog持久化等级：
 *     SKIP_WAL：只写缓存，不写HLog
 *     ASYNC_WAL, SYNC_WAL：异(同)步写HLog
 *     FSYNC_WAL：同步将数据写入日志文件并强制落盘
*/

private void sync(long txid, Durability durability) throws IOException {  
  if (this.getRegionInfo().isMetaRegion()) {  
    this.wal.sync(txid);  
  } else {  
    switch(durability) {  
    case USE_DEFAULT:  
      // do what table defaults to  
      if (shouldSyncWAL()) {  
        this.wal.sync(txid);  
      }  
      break;  
    case SKIP_WAL:  
      // nothing do to  
      break;  
    case ASYNC_WAL:  
      // nothing do to  
      break;  
    case SYNC_WAL:  
    case FSYNC_WAL:  
      // sync the WAL edit (SYNC and FSYNC treated the same for now)  
      this.wal.sync(txid);  
      break; 
    }  
  }  
}

public void sync(long txid) throws IOException {  
  try (TraceScope scope = TraceUtil.createTrace("AsyncFSWAL.sync")) {  
    // here we do not use ring buffer sequence as txid  
    SyncFuture future;  
    try {  
      // 获取Future对象
      future = getSyncFuture(txid);  
      RingBufferTruck truck = waitingConsumePayloads.get(sequence); 
      // 将future对象load至truck中 
      truck.load(future);  
    } finally {  
      waitingConsumePayloads.publish(sequence);  
    }  
    blockOnSync(future);  
  }  
}

// Handler进行处理
public void onEvent(final RingBufferTruck truck, final long sequence, boolean endOfBatch)    
  try {  
    if (truck.type() == RingBufferTruck.Type.SYNC) {
	  // 将Future对象添加到SyncFuture中  
      this.syncFutures[this.syncFuturesCount.getAndIncrement()] = truck.unloadSync();  
      if (this.syncFuturesCount.get() == this.syncFutures.length) {  
        endOfBatch = true;  
      }
   }
}

// 创建RingBufferEventHandler对象时，会创建SyncRunner和SyncFuture 
// SyncRunner的run方法会去SyncFuture的阻塞队列中取Future对象
public void run() {  
  long currentSequence;  
  while (!isInterrupted()) {  
    int syncCount = 0;  
    try {  
      while (true) {  
        takeSyncFuture = null;  
		// 获取到Future
        takeSyncFuture = this.syncFutures.take();  
      try {   
        writer.sync();    
      }
  }  
}

public void sync() throws IOException {  
  FSDataOutputStream fsdos = this.output;  
  if (fsdos == null) {  
    return; // Presume closed  
  }  
  // 刷写到磁盘
  fsdos.flush();  
  fsdos.hflush();  
}
```
### 4.4.5 写入 MemStore
```java
public WriteEntry writeMiniBatchOperationsToMemStore(  
    final MiniBatchOperationInProgress<Mutation> miniBatchOp, @Nullable WriteEntry writeEntry)  
    throws IOException {   
  super.writeMiniBatchOperationsToMemStore(miniBatchOp, writeEntry.getWriteNumber());  
  return writeEntry;  
}

protected void writeMiniBatchOperationsToMemStore(  
    final MiniBatchOperationInProgress<Mutation> miniBatchOp, final long writeNumber)  
    throws IOException {  
  MemStoreSizing memStoreAccounting = new NonThreadSafeMemStoreSizing();  
  visitBatchOperations(true, miniBatchOp.getLastIndexExclusive(), (int index) -> {  
    applyFamilyMapToMemStore(familyCellMaps[index], memStoreAccounting);  
    return true;  
  });  
  // update memStore size  
  region.incMemStoreSize(memStoreAccounting.getDataSize(), memStoreAccounting.getHeapSize(),  
    memStoreAccounting.getOffHeapSize(), memStoreAccounting.getCellsCount());  
}

protected void applyFamilyMapToMemStore(Map<byte[], List<Cell>> familyMap,  
    MemStoreSizing memstoreAccounting) throws IOException {  
  for (Map.Entry<byte[], List<Cell>> e : familyMap.entrySet()) {  
    byte[] family = e.getKey();  
    List<Cell> cells = e.getValue();  
    region.applyToMemStore(region.getStore(family), cells, false, memstoreAccounting);  
  }  
}

private void applyToMemStore(HStore store, List<Cell> cells, boolean delta,  
    MemStoreSizing memstoreAccounting) throws IOException {  
  // 传入的delta是false 
  boolean upsert = delta && store.getColumnFamilyDescriptor().getMaxVersions() == 1;  
  if (upsert) {  
  } else {  
    store.add(cells, memstoreAccounting);  
  }  
}

public void add(final Iterable<Cell> cells, MemStoreSizing memstoreSizing) {  
  lock.readLock().lock();  
  try {  
    memstore.add(cells, memstoreSizing);  
  } 
}

public void add(Iterable<Cell> cells, MemStoreSizing memstoreSizing) {  
  for (Cell cell : cells) {  
    add(cell, memstoreSizing);  
  }  
}

public void add(Cell cell, MemStoreSizing memstoreSizing) {  
  Cell toAdd = maybeCloneWithAllocator(cell, false);  
  internalAdd(toAdd, mslabUsed, memstoreSizing);  
}

private void internalAdd(final Cell toAdd, final boolean mslabUsed, MemStoreSizing memstoreSizing) {  
  // MutableSegment active
  // active的Segment处理write操作
  active.add(toAdd, mslabUsed, memstoreSizing);  
  setOldestEditTimeToNow();  
  checkActiveSize();  
}

public void add(Cell cell, boolean mslabUsed, MemStoreSizing memStoreSizing) {  
  internalAdd(cell, mslabUsed, memStoreSizing);  
}

protected void internalAdd(Cell cell, boolean mslabUsed, MemStoreSizing memstoreSizing) {  
  boolean succ = getCellSet().add(cell);  
  // 更新metaInfo
  updateMetaInfo(cell, succ, mslabUsed, memstoreSizing);  
}
```
# 5 MemStore 的 flush
## 5.1 MemStore 级别
### 5.1.1 checkResources()
```java
// 4.4 doMutate()
void checkResources() throws RegionTooBusyException {   
// 检查MemStore大小是否超过 hbase.hregion.memstore.flush.size 128M
    requestFlush();  
}

private void requestFlush() {  
  requestFlush0(FlushLifeCycleTracker.DUMMY);  
}

private void requestFlush0(FlushLifeCycleTracker tracker) {  
  if (shouldFlush) {  
    // getFlushRequester()返回HRegionServer中的MemStoreFlusher
    this.rsServices.getFlushRequester().requestFlush(this, false, tracker);   
  }  
}

public void requestFlush(HRegion r, boolean forceFlushAllStores, FlushLifeCycleTracker tracker) {  
  r.incrementFlushesQueuedCount();  
  synchronized (regionsInQueue) {  
    if (!regionsInQueue.containsKey(r)) { 
      // 封装成 FlushRegionEntry 后加入阻塞队列中
      FlushRegionEntry fqe = new FlushRegionEntry(r, forceFlushAllStores, tracker);  
      this.regionsInQueue.put(r, fqe);  
      this.flushQueue.add(fqe);  
    }  
  }  
}
```
### 5.1.2 HRegionServer 创建 MemStoreFlusher
```java
// handleReportForDutyResponse()方法调用startServices()：
protected void handleReportForDutyResponse(final RegionServerStartupResponse c)  
throws IOException {
	try{
		if (getConfiguration().getBoolean("hbase.regionserver.workers", true)) {  
		  startServices();  
		}
	}
}

private void startServices() throws IOException {  
  if (!isStopped() && !isAborted()) {  
    initializeThreads();  
  }
}

private void initializeThreads() throws IOException {  
  // Cache flushing thread.  
  this.cacheFlusher = new MemStoreFlusher(conf, this);  
  
  // Compaction thread  
  this.compactSplitThread = new CompactSplit(this);  
  
  // Background thread to check for compactions; needed if region has not gotten updates  
  // in a while. It will take care of not checking too frequently on store-by-store basis.  
  this.compactionChecker = new CompactionChecker(this, this.threadWakeFrequency, this);  
  this.periodicFlusher = new PeriodicMemStoreFlusher(this.threadWakeFrequency, this);  
}

public MemStoreFlusher(final Configuration conf,  
    final HRegionServer server) {  
  // 创建FlushHandler
  this.flushHandlers = new FlushHandler[handlerCount];
  this.cacheFlusher.start(uncaughtExceptionHandler);  
}

// FlushHandler的run()方法
public void run() {  
  while (!server.isStopped()) {  
    FlushQueueEntry fqe = null;  
    try {  
      wakeupPending.set(false);
	  // 从队列中取出一个fqe
      fqe = flushQueue.poll(threadWakeFrequency, TimeUnit.MILLISECONDS);  
      if (fqe == null || fqe == WAKEUPFLUSH_INSTANCE) {
	    // 判断全局memstore的大小   
        FlushType type = isAboveLowWaterMark();  
        if (type != FlushType.NORMAL) {           
          if (!flushOneForGlobalPressure()) {       
            Thread.sleep(1000);  
            wakeUpIfBlocking();  
          }  
          // Enqueue another one of these tokens so we'll wake up again  
          wakeupFlushThread();  
        }  
        continue;  
      }  
      FlushRegionEntry fre = (FlushRegionEntry) fqe;  
      if (!flushRegion(fre)) {  
        break;  
      }  
    } 
  }  
}
```
### 5.1.3 isAboveLowWaterMark()
```java
private FlushType isAboveLowWaterMark() {  
  return server.getRegionServerAccounting().isAboveLowWaterMark();  
}

// 不是NORMAL的情况下，计算大小后flush Region，否则直接flushRegion
public FlushType isAboveLowWaterMark() {  
  // for onheap memstore we check if the global memstore size and the  
  // global heap overhead is greater than the global memstore lower mark limit  if (memType == MemoryType.HEAP) {  
    if (getGlobalMemStoreHeapSize() >= globalMemStoreLimitLowMark) {  
      return FlushType.ABOVE_ONHEAP_LOWER_MARK;  
    }  
  } else {  
    if (getGlobalMemStoreOffHeapSize() >= globalMemStoreLimitLowMark) {  
      // Indicates that the offheap memstore's size is greater than the global memstore  
      // lower limit      return FlushType.ABOVE_OFFHEAP_LOWER_MARK;  
    } else if (getGlobalMemStoreHeapSize() >= globalOnHeapMemstoreLimitLowMark) {  
      // Indicates that the offheap memstore's heap overhead is greater than the global memstore  
      // onheap lower limit      return FlushType.ABOVE_ONHEAP_LOWER_MARK;  
    }  
  }  
  return FlushType.NORMAL;  
}
```
### 5.1.4 flushOneForGlobalPressure()
```java
private boolean flushOneForGlobalPressure() {
	// 计算出bestFlushableRegion，bestAnyRegion，bestRegionReplica以及对应的size
	flushedOne = flushRegion(regionToFlush, true, false, FlushLifeCycleTracker.DUMMY);
}

private boolean flushRegion(HRegion region, boolean emergencyFlush, boolean forceFlushAllStores,  
    FlushLifeCycleTracker tracker) {   
  try {  
    notifyFlushRequest(region, emergencyFlush);  
    FlushResult flushResult = region.flushcache(forceFlushAllStores, false, tracker);  
    boolean shouldCompact = flushResult.isCompactionNeeded();  
    // We just want to check the size  
    boolean shouldSplit = region.checkSplit() != null;  
    if (shouldSplit) {  
      this.server.compactSplitThread.requestSplit(region);  
    } else if (shouldCompact) {  
      server.compactSplitThread.requestSystemCompaction(region, Thread.currentThread().getName());  
    }   
  return true;  
}

public FlushResultImpl flushcache(boolean forceFlushAllStores, boolean writeFlushRequestWalMarker,  
    FlushLifeCycleTracker tracker) throws IOException {  
  lock.readLock().lock();  
    try {  
      Collection<HStore> specificStoresToFlush =  
          forceFlushAllStores ? stores.values() : flushPolicy.selectStoresToFlush();  
      FlushResultImpl fs =  
          internalFlushcache(specificStoresToFlush, status, writeFlushRequestWalMarker, tracker);  
    } 
}

private FlushResultImpl internalFlushcache(Collection<HStore> storesToFlush, MonitoredTask status,  
    boolean writeFlushWalMarker, FlushLifeCycleTracker tracker) throws IOException {  
  return internalFlushcache(this.wal, HConstants.NO_SEQNUM, storesToFlush, status,  
    writeFlushWalMarker, tracker);  
}

protected FlushResultImpl internalFlushcache(WAL wal, long myseqid,  
    Collection<HStore> storesToFlush, MonitoredTask status, boolean writeFlushWalMarker,  
    FlushLifeCycleTracker tracker) throws IOException {  
  // 创建一个PrepareFlushResult
  PrepareFlushResult result =  
      internalPrepareFlushCache(wal, myseqid, storesToFlush, status, writeFlushWalMarker, tracker);  
  if (result.result == null) {  
    // 执行flush
    return internalFlushCacheAndCommit(wal, status, result, storesToFlush);  
  }  
}   
```
## 5.2 Region 级别
- 当一个 `Region` 中所有的 `memstore` 的大小达到了 `hbase.hregion.memstore.flush.size(默认128M) *  hbase.hregion.memstore.block.multiplier(默认4)` 时，会阻止继续往该 `Region` 写数据，进行所有 `memstore` 的刷写
- 在进行 `Region` 级别的操作(`split, merge, compact`)前，都会执行 `requestFlush()`
## 5.3 RegionServer 级别
- 当一个 `RegionServer` 中所有 `Memstore` 的大小总和达到了上限(`hbase.regionserver.global.memstore.upperLimit ＊ hbase_heapsize，默认 40%的 JVM 内存使用量`)，会触发部分 `Memstore` 刷新
- flush 顺序是按照 `Memstore` 由大到小执行，直至总体内存使用量低于阈值（`hbase.regionserver.global.memstore.lowerLimit ＊ hbase_heapsize，默认 38%` ) 的 JVM 内存使用量）
![[RegionServerFlush.svg]]


## 5.4 定时 flush
```java
// 创建HRegion对象时，初始化flushCheckInterval
// DEFAULT_CACHE_FLUSH_INTERVAL = 3600000;
this.flushCheckInterval = conf.getInt(MEMSTORE_PERIODIC_FLUSH_INTERVAL,  
    DEFAULT_CACHE_FLUSH_INTERVAL);
    
// HRegionServer中的内部类
// initializeThreads中会创建PeriodicMemStoreFlusher对象
static class PeriodicMemStoreFlusher extends ScheduledChore {  
  final HRegionServer server;  

  @Override  
  protected void chore() {  
    final StringBuilder whyFlush = new StringBuilder();  
    for (HRegion r : this.server.onlineRegions.values()) {  
      if (r == null) continue; 
      // 判断需不需要flush：上次modify的时间距现在有没有超过1h
      if (r.shouldFlush(whyFlush)) {  
        FlushRequester requester = server.getFlushRequester();  
        if (requester != null) {     
          requester.requestDelayedFlush(r, randomDelay, false);  
        }  
      }  
    }  
  }  
}
```
## 5.5 HLog 上限
- `LogRoller` 的 `run()` 方法进行 HLog 的滚动
```java
// 1h进行一次roll
this.rollperiod = this.server.getConfiguration().  
  getLong("hbase.regionserver.logroll.period", 3600000);

public void run() {  
  while (running) {  
    rollLock.lock(); // FindBugs UL_UNRELEASED_LOCK_EXCEPTION_PATH  
    try {  
      this.lastrolltime = now;  
      for (Entry<WAL, Boolean> entry : walNeedsRoll.entrySet()) {  
        final WAL wal = entry.getKey();  
        // Force the roll if the logroll.period is elapsed or if a roll was requested.  
        // The returned value is an array of actual region names.        
        final byte [][] regionsToFlush = wal.rollWriter(periodic ||  
            entry.getValue().booleanValue());  
        walNeedsRoll.put(wal, Boolean.FALSE);  
        if (regionsToFlush != null) {  
          for (byte[] r : regionsToFlush) {  
	        // 获取到regionsToFlush后，调用requestFlush
            scheduleFlush(r);  
          }  
        }  
      }  
    } 
  }  
}

public byte[][] rollWriter(boolean force) throws FailedLogCloseException, IOException {  
  rollWriterLock.lock();  
  try {  
    byte[][] regionsToFlush = null;    
      if (getNumRolledLogFiles() > 0) {  
        cleanOldLogs();  
        regionsToFlush = findRegionsToForceFlush();  
      }  
    } 
    return regionsToFlush;  
  } 
}

byte[][] findRegionsToForceFlush() throws IOException {  
  byte[][] regions = null;  
  int logCount = getNumRolledLogFiles();
  // this.maxLogs = conf.getInt("hbase.regionserver.maxlogs",  Math.max(32, calculateMaxLogFiles(conf, logrollsize)));  
  if (logCount > this.maxLogs && logCount > 0) {  
	   // 给regions赋值后返回
   } 
  return regions;  
}
```
## 5.6 Flush 写出 HFile

# 6 读流程
## 6.1 RPC 发送 get 请求
```java
private Result get(Get get, final boolean checkExistenceOnly) throws IOException {  

	if (get.getConsistency() == Consistency.STRONG) {  
	  final Get configuredGet = get;  
	  ClientServiceCallable<Result> callable = new ClientServiceCallable<Result>() {  
    @Override  
    protected Result rpcCall() throws Exception {  
      ClientProtos.GetRequest request = RequestConverter.buildGetRequest();  
      ClientProtos.GetResponse response = doGet(request);  
      return response == null? null:  
        ProtobufUtil.toResult(response.getResult(), getRpcControllerCellScanner());  
    }  
  };  
	  return rpcCallerFactory.<Result>newCaller(readRpcTimeoutMs).callWithRetries(callable,  
      this.operationTimeoutMs);  
	}
}
```
## 6.2 RS 处理 GET 请求
```java
public GetResponse get(final RpcController controller,  
    final GetRequest request) throws ServiceException {  
  long before = EnvironmentEdgeManager.currentTime();  
  OperationQuota quota = null;  
  HRegion region = null;  
  try {  
    region = getRegion(request.getRegion());  
    Boolean existence = null;  
    Result r = null;  
    RpcCallContext context = RpcServer.getCurrentCall().orElse(null);  
    quota = getRpcQuotaManager().checkQuota(region, OperationQuota.OperationType.GET);  
  
    Get clientGet = ProtobufUtil.toGet(get);  
    if (get.getExistenceOnly() && region.getCoprocessorHost() != null) {  
      existence = region.getCoprocessorHost().preExists(clientGet);  
    }  
    if (existence == null) {  
      if (context != null) {
        // 获取数据  
        r = get(clientGet, (region), null, context);  
      }  
      if (get.getExistenceOnly()) {...}  
    }    
    return builder.build();  
  }
}

private Result get(Get get, HRegion region, RegionScannersCloseCallBack closeCallBack,  
    RpcCallContext context) throws IOException {  
  region.prepareGet(get);  
  
  // This method is almost the same as HRegion#get.  
  List<Cell> results = new ArrayList<>();  
  long before = EnvironmentEdgeManager.currentTime();  
  // 将 get 封装为 Scan 对象
  Scan scan = new Scan(get);  
  RegionScannerImpl scanner = null;  
  try {  
	// RegionScanner
    scanner = region.getScanner(scan);  
    scanner.next(results);  
  } 

  // 创建 Result 对象
  return Result.create(results, get.isCheckExistenceOnly() ? !results.isEmpty() : null, stale);  
}
```
## 6.3 创建 RegionScannerImpl
- Scanner 的构建流程

![[scanner.svg]]

- `RegionScanner` 由多个 `StoreScanner` 组成，一张表有多少个列族就有多少个 `StoreScanner`，每个 `StoreScanner` 负责对应 `Store` 的数据查找
- 一个 `StoreScanner` 由 `MemStoreScanner` 和 `StoreFileScanner` 组成
- `StoreScanner` 会为当前 `Store` 中每个 `HFile` 创建一个 `StoreFileScanner`，用于执行对应文件的检索，也会为 `MemStore` 生成一个 `MemStoreScanner` 用于执行该 `MemStore` 的数据检索
- 当创建完所有的 `Scanner` 后，需要通过 `key` 来过滤掉一些不满足查询条件的 `Scanner`
- 每个 `Scanner` 去寻找 `startKey`
- 最后将该 `Store` 中的所有 `Scanner` 合并成一个最小堆
```java
public RegionScannerImpl getScanner(Scan scan) throws IOException {  
 return getScanner(scan, null);  
}

public RegionScannerImpl getScanner(Scan scan, List<KeyValueScanner> additionalScanners)  
    throws IOException {  
  return getScanner(scan, additionalScanners, HConstants.NO_NONCE, HConstants.NO_NONCE);  
}

private RegionScannerImpl getScanner(Scan scan, List<KeyValueScanner> additionalScanners,  
    long nonceGroup, long nonce) throws IOException {
  // 判断操作类型，GET 或 SCAN 时是否支持读，添加读锁  
  startRegionOperation(Operation.SCAN);  
  try {  
    // Verify families are all valid
    // 没有待扫描的列族时，去 htableDescriptor 中获取后添加
    if (!scan.hasFamilies()) {  
      // Adding all families to scanner  
      for (byte[] family : this.htableDescriptor.getColumnFamilyNames()) {  
        scan.addFamily(family);  
      }  
    } else {  
      for (byte[] family : scan.getFamilyMap().keySet()) {  
        // 否则检查列族是否存在
        checkFamily(family);  
      }  
    }  
    return instantiateRegionScanner(scan, additionalScanners, nonceGroup, nonce);  
  } finally {  
    closeRegionOperation(Operation.SCAN);  
  }  
}
// Scan 可以正向也可以逆向进行，设置错误时可能出现找不到数据的情况
protected RegionScannerImpl instantiateRegionScanner(Scan scan,  
    List<KeyValueScanner> additionalScanners, long nonceGroup, long nonce) throws IOException {  
  if (scan.isReversed()) {  
    if (scan.getFilter() != null) {  
      scan.getFilter().setReversed(true);  
    }  
    return new ReversedRegionScannerImpl(scan, additionalScanners, this);  
  }  
  return new RegionScannerImpl(scan, additionalScanners, this, nonceGroup, nonce);  
}

RegionScannerImpl(Scan scan, List<KeyValueScanner> additionalScanners, HRegion region,  
    long nonceGroup, long nonce) throws IOException {    
  initializeScanners(scan, additionalScanners);  
}

protected void initializeScanners(Scan scan, List<KeyValueScanner> additionalScanners)  
    throws IOException {  
  try {
    // 遍历列族，创建对应数量的 Scanner  
    for (Map.Entry<byte[], NavigableSet<byte[]>> entry : scan.getFamilyMap().entrySet()) { 
      // 创建 Store 的 Scanner
      HStore store = stores.get(entry.getKey());  
      KeyValueScanner scanner = store.getScanner(scan, entry.getValue(), this.readPt);  
      instantiatedScanners.add(scanner);  
      if (this.filter == null || !scan.doLoadColumnFamiliesOnDemand()  
          || this.filter.isFamilyEssential(entry.getKey())) {  
        scanners.add(scanner);  
      } else {  
        joinedScanners.add(scanner);  
      }  
    }  
    initializeKVHeap(scanners, joinedScanners, region);  
  }
}
```
### 6.3.1 创建 StoreScanner
```java
public KeyValueScanner getScanner(Scan scan, final NavigableSet<byte[]> targetCols, long readPt)  
    throws IOException {  
  lock.readLock().lock();  
  try {
    // scanInfo 中有扫描列族的信息   
    ScanInfo scanInfo;  
    return createScanner(scan, scanInfo, targetCols, readPt);  
  } finally {  
    lock.readLock().unlock();  
  }  
}

protected KeyValueScanner createScanner(Scan scan, ScanInfo scanInfo,  
    NavigableSet<byte[]> targetCols, long readPt) throws IOException {  
  return scan.isReversed() ? new ReversedStoreScanner(this, scanInfo, scan, targetCols, readPt)  
      : new StoreScanner(this, scanInfo, scan, targetCols, readPt);  
}

// StoreScanner.java
public StoreScanner(HStore store, ScanInfo scanInfo, Scan scan, NavigableSet<byte[]> columns,  
    long readPt) throws IOException {  
  // 构造器创建 StoreScanner
  this(store, scan, scanInfo, columns != null ? columns.size() : 0, readPt, scan.getCacheBlocks(), ScanType.USER_SCAN);  
  
  try {  
    // Pass columns to try to filter out unnecessary StoreFiles.  
    // 创建 MemStore 和 StoreFile 的 Scanner后，通过 KV 过滤
    List<KeyValueScanner> scanners = selectScannersFrom(store,  
      store.getScanners(cacheBlocks, scanUsePread, false, matcher, scan.getStartRow(), scan.includeStartRow(), scan.getStopRow(), scan.includeStopRow(), this.readPt));  
  
    // 每个 Scanner 遍历寻找 start key
    seekScanners(scanners, matcher.getStartKey(), explicitColumnQuery && lazySeekEnabledGlobally, parallelSeekEnabled);  
    // 构建 heap
	resetKVHeap(scanners, comparator);
  }   
}
```
### 6.3.2 创建 StoreFileScanner 和 KeyValueScanner(扫描 MemStore)
```java
public List<KeyValueScanner> getScanners(boolean cacheBlocks, boolean usePread,  
    boolean isCompaction, ScanQueryMatcher matcher, byte[] startRow, boolean includeStartRow,  
    byte[] stopRow, boolean includeStopRow, long readPt) throws IOException {  
  Collection<HStoreFile> storeFilesToScan;  
  List<KeyValueScanner> memStoreScanners;  
  this.lock.readLock().lock();  
  try {
    // 获取要 scan 的StoreFile  
    storeFilesToScan = this.storeEngine.getStoreFileManager().getFilesForScan(startRow,  
      includeStartRow, stopRow, includeStopRow);
    // 创建 MemStore 的Scanner  
    memStoreScanners = this.memstore.getScanners(readPt);  
  } finally {  
    this.lock.readLock().unlock();  
  }  
  
  // First the store file scanners  
  List<StoreFileScanner> sfScanners = StoreFileScanner.getScannersForStoreFiles(storeFilesToScan,  
    cacheBlocks, usePread, isCompaction, false, matcher, readPt);  
  List<KeyValueScanner> scanners = new ArrayList<>(sfScanners.size() + 1); 
  // 加入 scanner 的 ArrayList中 
  scanners.addAll(sfScanners);  
  // Then the memstore scanners  
  scanners.addAll(memStoreScanners);  
  return scanners;  
}
```
### 6.3.3 创建 KeyValueScanner
```java
public List<KeyValueScanner> getScanners(long readPt) throws IOException {  
  List<KeyValueScanner> list = new ArrayList<>();  
  long order = snapshot.getNumOfSegments(); 
  // 返回 KeyValueScanner 的子类 SegmentScanner
  order = addToScanners(active, readPt, order, list);  
  addToScanners(snapshot.getAllSegments(), readPt, order, list);  
  return list;  
}
```
### 6.3.4 创建 StoreFileScanner
```java
public static List<StoreFileScanner> getScannersForStoreFiles(Collection<HStoreFile> files,  
    boolean cacheBlocks, boolean usePread, boolean isCompaction, boolean canUseDrop,  
    ScanQueryMatcher matcher, long readPt) throws IOException {  
  List<StoreFileScanner> scanners = new ArrayList<>(files.size());  
  PriorityQueue<HStoreFile> sortedFiles =  
      new PriorityQueue<>(files.size(), StoreFileComparators.SEQ_ID);  
  for (HStoreFile file : files) {  
    // 初始化 Reader  
    file.initReader();  
    sortedFiles.add(file);  
  }  
  boolean succ = false;  
  try {  
    for (int i = 0, n = files.size(); i < n; i++) {  
      HStoreFile sf = sortedFiles.remove();  
      StoreFileScanner scanner;  
      if (usePread) {  
        scanner = sf.getPreadScanner(cacheBlocks, readPt, i, canOptimizeForNonNullColumn);  
      } else {  
        scanner = sf.getStreamScanner(canUseDrop, cacheBlocks, isCompaction, readPt, i,  
            canOptimizeForNonNullColumn);  
      }  
      scanners.add(scanner);  
    }  
    succ = true;  
  } finally {  
    if (!succ) {  
      for (StoreFileScanner scanner : scanners) {  
        scanner.close();  
      }  
    }  
  }  
  return scanners;  
}
```
### 6.3.5 判断 Scanner 是否符合查询条件
```java
// StoreScanner.java
protected List<KeyValueScanner> selectScannersFrom(HStore store,  
    List<? extends KeyValueScanner> allScanners) {  
  
  List<KeyValueScanner> scanners = new ArrayList<>(allScanners.size());  
  
  for (KeyValueScanner kvs : allScanners) {  
    boolean isFile = kvs.isFileScanner();  
  
    if (kvs.shouldUseScanner(scan, store, expiredTimestampCutoff)) {  
      scanners.add(kvs);  
    }
  }  
  return scanners;  
}

// SegmentScanner.java
public boolean shouldUseScanner(Scan scan, HStore store, long oldestUnexpiredTS) {  
  return getSegment().shouldSeek(scan.getColumnFamilyTimeRange()  .getOrDefault(store.getColumnFamilyDescriptor().getName(), scan.getTimeRange()), oldestUnexpiredTS);  
}

// StoreFileScanner.java
public boolean shouldUseScanner(Scan scan, HStore store, long oldestUnexpiredTS) {  
  // if the file has no entries, no need to validate or create a scanner.  
  byte[] cf = store.getColumnFamilyDescriptor().getName();  
  TimeRange timeRange = scan.getColumnFamilyTimeRange().get(cf);  
  if (timeRange == null) {  
    timeRange = scan.getTimeRange();  
  }  
  return reader.passesTimerangeFilter(timeRange, oldestUnexpiredTS) && reader  
      .passesKeyRangeFilter(scan) && reader.passesBloomFilter(scan, scan.getFamilyMap().get(cf));  
}
```
### 6.3.6 筛选 Scanners 
```java
// 通过给定的 key 寻找特定的 Scanner
protected void seekScanners(List<? extends KeyValueScanner> scanners, Cell seekKey, boolean isLazy, boolean isParallelSeek)  
    throws IOException {  
  if (isLazy) {  
    for (KeyValueScanner scanner : scanners) {  
      scanner.requestSeek(seekKey, false, true);  
    }  
  } else {  
    if (!isParallelSeek) {  
      long totalScannersSoughtBytes = 0;  
      for (KeyValueScanner scanner : scanners) {  
        if (matcher.isUserScan() && totalScannersSoughtBytes >= maxRowSize) {    
        scanner.seek(seekKey);  
        Cell c = scanner.peek();  
        if (c != null) {  
          totalScannersSoughtBytes += PrivateCellUtil.estimatedSerializedSizeOf(c);  
        }  
      }  
    } else {  
      parallelSeek(scanners, seekKey);  
    }  
  }  
}

// 最终都会调用 KeyValueHeap 的 seek 方法
public boolean seek(Cell seekKey) throws IOException {  
  return generalizedSeek(false,    // This is not a lazy seek  
      seekKey,  
      false,    // forward (false: this is not a reseek)  
      false);   // Not using Bloom filters  
}

private boolean generalizedSeek(boolean isLazy, Cell seekKey,  
    boolean forward, boolean useBloom) throws IOException {  
  KeyValueScanner scanner = current;  
  try {  
    while (scanner != null) {  
      Cell topKey = scanner.peek();  
      // 要找的 key <= topKey
      if (comparator.getComparator().compare(seekKey, topKey) <= 0) {  
        heap.add(scanner);  
        scanner = null;  
        current = pollRealKV();  
        return current != null;  
      }  
    }
  }
}
```
### 6.3.7 Scanner 构建最小堆
```java
protected void resetKVHeap(List<? extends KeyValueScanner> scanners,  
    CellComparator comparator) throws IOException {  
  // Combine all seeked scanners with a heap  
  heap = newKVHeap(scanners, comparator);  
}

protected KeyValueHeap newKVHeap(List<? extends KeyValueScanner> scanners,  
    CellComparator comparator) throws IOException {  
  return new KeyValueHeap(scanners, comparator);  
}

public KeyValueHeap(List<? extends KeyValueScanner> scanners,  
    CellComparator comparator) throws IOException {  
  this(scanners, new KVScannerComparator(comparator));  
}

// 创建一个优先级队列，从小到大排序
KeyValueHeap(List<? extends KeyValueScanner> scanners,  
    KVScannerComparator comparator) throws IOException {  
  this.comparator = comparator;  
  this.scannersForDelayedClose = new ArrayList<>(scanners.size());  
  if (!scanners.isEmpty()) {  
    this.heap = new PriorityQueue<>(scanners.size(), this.comparator);  
    for (KeyValueScanner scanner : scanners) {  
      if (scanner.peek() != null) {  
        this.heap.add(scanner);  
      } else {  
        this.scannersForDelayedClose.add(scanner);  
      }  
    }  
    this.current = pollRealKV();  
  }  
}
```
## 6.4 返回 get 方法读取数据
```java
// 6.2 scanner.next()
public synchronized boolean next(List<Cell> outResults, ScannerContext scannerContext)  
throws IOException {   
  startRegionOperation(Operation.SCAN);  
  try {  
    return nextRaw(outResults, scannerContext);  
  } finally {  
    closeRegionOperation(Operation.SCAN);  
  }  
}

public boolean nextRaw(List<Cell> outResults, ScannerContext scannerContext)  
    throws IOException {  

  boolean moreValues = false;  
  if (outResults.isEmpty()) {  
    // Usually outResults is empty. This is true when next is called  
    // to handle scan or get operation.   
     moreValues = nextInternal(outResults, scannerContext);  
  }
  return moreValues;  
}

private boolean nextInternal(List<Cell> results, ScannerContext scannerContext)  
    throws IOException {

	while (true) {
	// 获取一个 Cell
	Cell current = this.storeHeap.peek();
	// 判断是否到 stopRow
	boolean shouldStop = shouldStop(current);
	if (joinedContinuationRow == null) {
		// 判断有没有产生数据
		populateResult(results, this.storeHeap, scannerContext, current);
		// 取下一个 Cell 
		Cell nextKv = this.storeHeap.peek();  
		// 判断是否要停止
		shouldStop = shouldStop(nextKv);
		}
	}
}

// c = 0, 到了stopRow时，返回 false，向下执行写入结果后
// 第二次进入时返回 false 停止
protected boolean shouldStop(Cell currentRowCell) {  
  if (currentRowCell == null) {  
    return true;  
  }  
  if (stopRow == null || Bytes.equals(stopRow, HConstants.EMPTY_END_ROW)) {  
    return false;  
  }  
  int c = comparator.compareRows(currentRowCell, stopRow, 0, stopRow.length);  
  return c > 0 || (c == 0 && !includeStopRow);  
}
```
# 7 Disruptor
# 8 Shell 操作
```shell
# 进入命令行
hbase shell
# 查看命名空间
list_namespace
# 创建命名空间
create_namespace [namespace_name]
# 创建表格
create 'ns1:t1', 'cf1', 'cf2'
create 'ns2:t1', {NAME => 'f1', VERSIONS => 5}, 保留5个版本
# 删除表格
disable 'ns:t1' # 先禁用
drop 'ns:t1' 
# 查看表信息
describe 'ns:t1'
# 修改表
alter 'ns:t1', NAME => 'f1', VERSIONS => 5
# 写数据
put 'ns:t1', 'r1', 'c1', 'value' # c1,  列族：列名
# 读数据
get 'ns:t1', 'c1'
get 'ns:t1', 'c1', {COLUMN => 'cf:q1'} # 行列过滤
scan 'ns:t1'
# 删除数据
delete 'ns:t1', 'cf:q1' # 删除最新
deleteall
```
# 9 复习
## 9.1 架构
- 外部：`ZK`,  `HDFS`
- 内部：
	- `HMaster`：管理元数据
	- `RegionServer`：管理数据和 Region
		- `WALFactory`：处理和 WAL 相关的请求
		- `BlockCache`
		- `Region`： 表的切片
			- `WAL`
			- `Store`：列族的切片
				- `MemStore`
				- `StoreFile`
## 9.2 写流程
- 客户端首先去内存中寻找元数据信息，刚启动时找不到(4.2.4)
- 与 `zk` 通信，获取 `meta` 表所在的 `RegionServer` (4.2.6)
- 向 `meta` 表所在的 `RS` 发起写请求，获取 `meta` 表的内容(要写的表在哪个 `RS`) (4.3.1)
- 将 `meta` 表内容缓存在本地(4.3.2)
- 通过 `RPC` 向 `Server` 发送 `put` 请求
- 首先写 `WAL`，将数据写入本地缓存(4.4.2.3)，将缓存写入 `HDFS` (4.4.3)，再将数据同步到磁盘(4.4.4)
- 最后将数据写入到 `MemStore` (4.4.5)
## 9.3 读流程
- 缓存 `meta` 表为止的步骤与写流程相同
- 向待写入表发起 Get 请求
- `Server` 收到 `Get` 请求，创建 `MemStore` 和 `StoreFile` 的 `Scanner`
	- `MemStore` 的 `Scanner` 在内存中直接读取
	- `StoreFile` 的 `Scanner` 
		- 首先通过布隆过滤器读取文件的索引部分，对要检索的行的信息进行索引，判断文件中是否有要查询的行，接着根据信息找到行所在的数据块
		- 根据 `Block` 的 `id` 判断是否已经在 `BlockCache` 中缓存过，缓存过的情况下不会读取 `StoreFile`，否则从 `StoreFile` 中扫描 `Block` 进行缓存
	- 因为 `HBase` 中的数据存在版本，查到的数据不一定是版本最大的，因此将从 `MemStore`，`StoreFile`，`BlockCache` 中查到的所有数据进行合并
		- 所有指数据是同一条数据的不同版本 (`ts`) 或不同的类型 (`PUT` / `DELETE`)
	- 返回合并结果(非 `DELETE` 数据)
## 9.4 刷写
- RegionServer 级别：到达堆内存的 `40% * 95% = 38%` 时刷写
	- 达到堆内存的 40% 时会阻塞写入，直到降低到 38% 以下
- Region 级别：
	- 某个 `MemStore` 到达 128M 时，所在 `Region` 的所有 `MemStore` 都会进行刷写
- HLog 文件数达到 32
- 官方不建议使用过多 `ColumnFamily`，是因为当一个 `MemStore` 达到 128M，而其他 `MemStore` 的大小还很小时，刷写就会产生大量的小文件
## 9.5 合并
- 合并是从一个 Region 的一个 Store 中选取部分 HFile 文件进行合并
- 合并有两种：Minor Compaction 和 Major Compaction
	- `MemStore` flush操作结束后会检查当前 `Store` 中 `StoreFile` 的个数，一旦超过了 ` hbase.hstore.compactionThreshold 默认3 `，就会触发合并
	- RS 会在后台启动一个 `CompactionChecker` 线程定期触发检查对应的 `Store` 是否需要执行合并，对应的参数
		- `hbase.server.thread.wakefrequency` 默认 10000ms
		- `hbase.server.compactchecker.interval.multiplier` 默认 1000
		- 不满足 `compactionThreshold` 的条件时，就会去检查是否需要 Major 合并
- Major 合并的参数
	- `hbase.hregion.majorcompaction` * `hbase.hregion.majorcompaction.jitter 默认.50F`
	- 默认 7 天进行 Major 合并
- Minor Compaction 是指选取部分小的、相邻的 `HFile`，将它们合并成一个更大的 HFile
- Major Compaction 是指将一个 `Store` 中所有的 `HFile` 合并成一个 `HFile`，这个过程还会完全清理三类无意义数据：被删除的数据、`TTL` 过期数据、版本号超过设定版本号的数据
## 9.6 切分
- 每个 Table 初始只有一个 Region，随着数据的不断写入，Region 会进行自动切分
- 切分策略的类都继承自 `RegionSplitPolicy`，有两个方法
	- `shouldSplit()`
	- `getSplitPoint()`：返回 `row` 的值，将所有 `row` 分为两段
- 切分的时机： `hbase.regionserver.region.split.policy`
	- 默认 `org.apache.hadoop.hbase.regionserver.SteppingSplitPolicy`
	- 如果 RS 只有一个 Region，按照 `2 * hbase.hregion.memstore.flush.size 刷写memstore大小` 进行切分
	- 否则按照 `habse.hregion.max.filesize 默认10G` 进行切分
## 9.7 数据删除时机
- eg:
```shell
# 向 HBase 中 put 一条数据
put 'table', 'rk', 'colFamily:xxx', 'val1' 
# 更改数据
put 'table', 'rk', 'colFamily:xxx', 'val2'
# 通过 scan 查看历史版本数据
scan 'table', {RAW=>TRUE, VERSIONS>=2}
# 返回
ROW           COLUMN+CELL
rk            column=colFamily:xxx, timestamp=ts2, value=val2
rk            column=colFamily:xxx, timestamp=ts1, value=val1 
# 手动flush, 再进行查询时只能查到val2
flush 'table'
# 再更改一次数据
put 'table', 'rk', 'colFamily:xxx', 'val3'
# flush后查看
ROW           COLUMN+CELL
rk            column=colFamily:xxx, timestamp=ts3, value=val3
rk            column=colFamily:xxx, timestamp=ts2, value=val2 
```
- 原因
	- 第一次刷写后的 `val2` 在文件中，第二次再进行刷写时无法删除
	- 刷写会删除在同一个 `MemStore` 中的过期数据
	- 而合并会删除所有过期数据，`Major` 合并会删除数据的删除标记
## 9.8 RowKey 设计
### 9.8.1 设计原则
- 唯一性，`rowkey` 需要包含事实的主键列
- 散列性
- 长度
- 范围查询的需求应尽量连续紧凑分布
- 随机查询的需求应尽量散列分布，保证负载均衡
### 9.8.2 场景题
#### 9.8.2.1 手机号查询指定事件通话记录
- 预分区(散列性)
	- 00|, 01|, 02|... ‘|’的 `ascii` 码为 124
	- 评估数据量，保证未来单个 `Region` 不超过 10G，同时考虑机器台数，尽量让 `Region` 数为机器台数的整数倍，保证均匀分布
- 分区号
	- 00_, 01_, 02_,...     '\_'的 `ascii` 码为 95
	- 轮询可以保证散列，但是查询困难
	- 考虑查询条件，手机号年(月日)
		- (手机号 + 年月日).hash() mod 分区数  
			- 按年份查找时需要跨分区数过多
		- (手机号 + 年月).hash() mod 分区数
			- 按天和按月查只需要一个分区
			- 按年最多也只需要扫描 12 个分区
		- (手机号 + 年).hash() mod 分区数
		- 手机号.hash() mod 分区数
- 拼接字段
	- 考虑查询条件
		- 0X_13930391234_2023-12-12 12:00:00
		- 0X_2023-12-12 12:00:00_13930391234
	- 手机号在前：
		- start_key: 0X_13930391234_2023-12
		- stop_key: 0X_13930391234_2023-12|
		- 不会多数据，也不少数据
	- 日期在前
		- start_key: 0X_2023-12-00 00:00:00_13930391234
		- stop_key: 0X_2023-13-00 00:00:00_13930391234
		- 有可能混入其他手机号
#### 9.8.2.2 消费金额
```sql
+-----+-------+---------------------+
| id  |  user |         date        | 
+-----+-------+---------------------+
|  1  |   a   | 2022-01-05 09:00:00 |
|  2  |   b   | 2021-12-30 08:00:00 |
|  3  |   c   | 2022-01-04 09:08:00 |
|  4  |   d   | 2021-12-31 09:08:00 |
+-----+-------+---------------------+
```
- 统计某个用户在某个月份消费的总金额
	- id 放在开头，订单不一定连续，不满足紧凑分布的条件
	- start_key: 0X_a_2022-01
	- stop_key: 0X_a_2022-01|
- 统计所有用户在某个月份消费的总金额
	- start_key: 0X_2022-01
	- stop_key: 0X_2022-02
### 9.8.3 项目中 RowKey 的设计
- 需求：主流根据维度表的主键查询单条明细数据
	- 不需要考虑集中性
	- 对于数据量大的表可以考虑预分区
	- 使用 `MySQL` 维表的主键作为 `RowKey`
	- 如果是预分区表，需要将主键 `Hash` 取分区号拼接原先的主键作为 `RowKey`
# 10 面试
- 1 HBase 协处理器
	- 将业务计算代码放在 RS 的协处理器中，将处理好的数据返回给客户端
	- 类型
		- Observer
			- 类似于 `RDBMS` 中的触发器，发生某些事件时调用
			- RegionObserver: 观察 `Region` 上的事件，如 `Get` 和 `Put` 操作
			- RegionServerObserver：观察与 `RS` 相关的事件，合并，启动等
			- MasterObserver
			- WalObserver
			- 可以实现权限管理，监控，二级索引等
		- Endpoint
			- 类似于 `RDBMS` 中的存储过程
			- 存储过程是一组完成特定功能的 SQL 语句集，类似于函数
			- 可以实现聚合功能
- 2 HBase 过滤器
	- 所有的过滤器在服务端生效，类似谓词下推
	- 过滤器的基类是 FilterBase
	- 过滤器主要有三种
		- 比较过滤器，继承自 CompareFilter
		- 专用过滤器，用于小范围过滤，直接继承 FilterBase
		- 包装过滤器
	- 通过 `setFilter()` 方法传给 `Get` 或 `Scan` 对象即可
- 3 HBase 模糊查询
	- Scan 实现
		- `Scan.setRowPrefixFilter()`
	- Filter 实现
		- `PrefixFilter`
- 4 二级索引
	- 索引种类
		- 全局索引
			- 适合读多写少
			- 全局索引的性能损耗来自于写数据，数据的增删改都会更新相关索引表
		- 本地索引
			- 适合写多读少, 或存储空间有限
			- 本地索引的索引数据和原数据存在同一台机器上，网络传输开销少
		- 覆盖索引
			- 把原数据存储在索引数据表中，查询时可以直接返回查询结果
	- 本质是建立各列值与 `rowkey` 间的映射关系
	- `HBase` 的一级索引是 `rowkey`
	- 索引方案
		- 基于协处理器实现
			- 在进行操作时，可以通过协处理器将信息更新到另一张索引表中
		- Phoenix 实现
- 5 优化
	- 客户端
		- 批量 `Get`，减少 `RPC` 次数
		- `Get` 时指定列族，标识符，过滤掉 Scanner，减少 I/O 次数
		- `Scan` 时设置合理的 `startRow` 和 `stopRow`，防止全表扫描
	- 服务端
		- 数据压缩
		- 数据优先缓存在内存，有效提升读取性能
	- 预分区
	- `WAL` 存储级别
		- 