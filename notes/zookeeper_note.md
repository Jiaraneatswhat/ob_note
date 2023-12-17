# 1. 启动
## 1.1 脚本
```shell
# 执行zkEnv.sh 寻找类路径, java的路径等
if [ -e "$ZOOBIN/../libexec/zkEnv.sh" ]; then
  . "$ZOOBINDIR"/../libexec/zkEnv.sh
else
  . "$ZOOBINDIR"/zkEnv.sh
fi

if [ "x$JMXDISABLE" = "x" ] || [ "$JMXDISABLE" = 'false' ]
then
  echo "ZooKeeper JMX enabled by default" >&2
  if [ "x$JMXPORT" = "x" ]
  then
    ZOOMAIN="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=$JMXLOCALONLY org.apache.zookeeper.server.quorum.QuorumPeerMain"
fi

case $1 in
start)
    echo  -n "Starting zookeeper ... "
	# 启动java进程
    nohup "$JAVA" $ZOO_DATADIR_AUTOCREATE "-Dzookeeper.log.dir=${ZOO_LOG_DIR}" \
    "-Dzookeeper.log.file=${ZOO_LOG_FILE}" "-Dzookeeper.root.logger=${ZOO_LOG4J_PROP}" \
    -XX:+HeapDumpOnOutOfMemoryError -XX:OnOutOfMemoryError='kill -9 %p' \
    -cp "$CLASSPATH" $JVMFLAGS $ZOOMAIN "$ZOOCFG" > "$_ZOO_DAEMON_OUT" 2>&1 < /dev/null &
```
## 1.2 QuorumPeerMain.main()
```java
public static void main(String[] args) {  
    QuorumPeerMain main = new QuorumPeerMain();    
ServiceUtils.requestSystemExit(ExitCode.EXECUTION_FINISHED.getValue()); 
}

protected void initializeAndRun(String[] args) throws ConfigException, IOException, AdminServerException {  
    QuorumPeerConfig config = new QuorumPeerConfig();  
    if (args.length == 1) {  
        config.parse(args[0]);  
    }  
  
    // Start and schedule the the purge task  

	// 集群模式
    if (args.length == 1 && config.isDistributed()) {  
        runFromConfig(config);  
    }  
}

public void runFromConfig(QuorumPeerConfig config) throws IOException, AdminServerException {   
  
    LOG.info("Starting quorum peer, myid=" + config.getServerId());  
    try {   
        ServerCnxnFactory cnxnFactory = null;  
        ServerCnxnFactory secureCnxnFactory = null;  
	    // 1. 服务端的通信组件 的初始化
        if (config.getClientPortAddress() != null) { 
	         // 通过反射创建一个ServerCnxnFactory对象
            cnxnFactory = ServerCnxnFactory.createFactory();  
            cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), false);  
        }   
		// 2. 抽象一个zk节点并启动, QuorumPeer是服务器资源的抽象
        quorumPeer = getQuorumPeer();  
        // ZKDatabase 管理会话、DataTree存储和事务日志
        quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));  
     
        quorumPeer.initialize();  

        quorumPeer.start();  
        ZKAuditProvider.addZKStartStopAuditLog();  
        quorumPeer.join();  
    }
}
```
## 1.3 QuorumPeer 启动
```java
protected QuorumPeer getQuorumPeer() throws SaslException {  
    return new QuorumPeer();  
}

// QuorumPeer管理节点的状态，节点可以有三种状态：选举Leader, Leader以及Follower, 继承了ZooKeeperThread
public synchronized void start() {  
    if (!getView().containsKey(myid)) {  
        throw new RuntimeException("My id " + myid + " not in the peer list");  
    }  
    // 从磁盘恢复节点数据到内存
    loadDataBase();  
    // 启动服务端通信组件工厂
    startServerCnxnFactory();  
    try {  
	    // 启动 AdminServer
        adminServer.start();  
    } catch (AdminServerException e) {  
        LOG.warn("Problem starting AdminServer", e);  
    }  
    // 准备选举的一些必要操作(初始化一些队列和一些线程)
    startLeaderElection();  
    startJvmPauseMonitor();  
    super.start();  
}
```
### 1.3.1 冷启动恢复节点数据
```java
private void loadDataBase() {  
    try {  
	    // 返回zxid
        zkDb.loadDataBase();  

        // load the epochs  
		long lastProcessedZxid = zkDb.getDataTree().lastProcessedZxid;  
		// 存在currentEpoch文件时直接获取epoch, 否则通过 Zxid >> 32L得到Epoch
        long epochOfZxid = ZxidUtils.getEpochFromZxid(lastProcessedZxid);  
        try {  
            currentEpoch = readLongFromFile(CURRENT_EPOCH_FILENAME);  
        } catch (FileNotFoundException e) {  
            // pick a reasonable epoch number  
            // this should only happen once when moving to a            // new code version            currentEpoch = epochOfZxid;  
            LOG.info(  
                "{} not found! Creating with a reasonable default of {}. "  
                    + "This should only happen when you are upgrading your installation",  
                CURRENT_EPOCH_FILENAME,  
                currentEpoch);  
            writeLongToFile(CURRENT_EPOCH_FILENAME, currentEpoch);  
        }  
        if (epochOfZxid > currentEpoch) {  
            // acceptedEpoch.tmp file in snapshot directory  
            File currentTmp = new File(getTxnFactory().getSnapDir(),  
                CURRENT_EPOCH_FILENAME + AtomicFileOutputStream.TMP_EXTENSION);  
            if (currentTmp.exists()) {  
                long epochOfTmp = readLongFromFile(currentTmp.getName());  
                LOG.info("{} found. Setting current epoch to {}.", currentTmp, epochOfTmp);  
                setCurrentEpoch(epochOfTmp);  
            } else {  
                throw new IOException(  
                    "The current epoch, " + ZxidUtils.zxidToString(currentEpoch)  
                        + ", is older than the last zxid, " + lastProcessedZxid);  
            }  
        }  
        try {  
            acceptedEpoch = readLongFromFile(ACCEPTED_EPOCH_FILENAME);  
        } catch (FileNotFoundException e) {  
            // pick a reasonable epoch number  
            // this should only happen once when moving to a            // new code version            acceptedEpoch = epochOfZxid;  
            LOG.info(  
                "{} not found! Creating with a reasonable default of {}. "  
                    + "This should only happen when you are upgrading your installation",  
                ACCEPTED_EPOCH_FILENAME,  
                acceptedEpoch);  
            writeLongToFile(ACCEPTED_EPOCH_FILENAME, acceptedEpoch);  
        }  
        if (acceptedEpoch < currentEpoch) {  
            throw new IOException("The accepted epoch, "  
                                  + ZxidUtils.zxidToString(acceptedEpoch)  
                                  + " is less than the current epoch, "  
                                  + ZxidUtils.zxidToString(currentEpoch));  
        }  
    } catch (IOException ie) {  
        LOG.error("Unable to load database on disk", ie);  
        throw new RuntimeException("Unable to run quorum server ", ie);  
    }  
}

public long loadDataBase() throws IOException {  
    long zxid = snapLog.restore(dataTree, sessionsWithTimeouts, commitProposalPlaybackListener);  
    return zxid;  
}
```
