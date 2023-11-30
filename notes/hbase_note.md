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
        rpcServices.isa.getPort(), this, canCreateBaseZNode());  
      // If no master in cluster, skip trying to track one or look for a cluster status.  
      if (!this.masterless) {  
        this.csm = new ZkCoordinatedStateManager(this);  
  
        masterAddressTracker = new MasterAddressTracker(getZooKeeper(), this);  
        masterAddressTracker.start();  
  
        clusterStatusTracker = new ClusterStatusTracker(zooKeeper, this);  
        clusterStatusTracker.start();  
      } else {  
        masterAddressTracker = null;  
        clusterStatusTracker = null;  
      }  
    } else {  
      zooKeeper = null;  
      masterAddressTracker = null;  
      clusterStatusTracker = null;  
    }  
    this.rpcServices.start(zooKeeper);  
    // This violates 'no starting stuff in Constructor' but Master depends on the below chore  
    // and executor being created and takes a different startup route. Lots of overlap between HRS    // and M (An M IS A HRS now). Need to refactor so less duplication between M and its super    // Master expects Constructor to put up web servers. Ugh.    // class HRS. TODO.    this.choreService = new ChoreService(getName(), true);  
    this.executorService = new ExecutorService(getName());  
    putUpWebUI();  
  } catch (Throwable t) {  
    // Make sure we log the exception. HRegionServer is often started via reflection and the  
    // cause of failed startup is lost.    LOG.error("Failed construction RegionServer", t);  
    throw t;  
  }  
}
```