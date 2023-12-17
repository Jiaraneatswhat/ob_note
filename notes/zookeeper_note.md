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
    // 调用run方法 
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
		// 存在 currentEpoch 文件时直接获取 Epoch, 否则通过 Zxid >> 32L得到 Epoch
        long epochOfZxid = ZxidUtils.getEpochFromZxid(lastProcessedZxid);  
        try {  
            currentEpoch = readLongFromFile(CURRENT_EPOCH_FILENAME);  
        } catch (FileNotFoundException e) { // 写入文件中
        }  
        if (epochOfZxid > currentEpoch) {  
	        // 现有的 zxid > 当前的epoch
            // acceptedEpoch.tmp file in snapshot directory  
            File currentTmp = new File(getTxnFactory().getSnapDir(),  
    CURRENT_EPOCH_FILENAME + AtomicFileOutputStream.TMP_EXTENSION);  
            if (currentTmp.exists()) { 
		            // 更新epoch  
                setCurrentEpoch(epochOfTmp);  
            } else {  
                throw new IOException(  
                    "The current epoch, " + ZxidUtils.zxidToString(currentEpoch)  
                        + ", is older than the last zxid, " + lastProcessedZxid);  
            }  
        }  
        try {  
            acceptedEpoch = readLongFromFile(ACCEPTED_EPOCH_FILENAME); 
            // 检查 acceptedEpoch 是否早于 currentEpoch
            // 通过 acceptedEpoch 和 currentEpoch防止选举进入一个特殊状态
        }  
    }
}

public long loadDataBase() throws IOException {  
    long zxid = snapLog.restore(dataTree, sessionsWithTimeouts, commitProposalPlaybackListener);  
    return zxid;  
}
```
### 1.3.2 准备选举
```java
// QuorumPeer将状态初始化为LOOKING
// private ServerState state = ServerState.LOOKING
public synchronized void startLeaderElection() {  
    try {  
        if (getPeerState() == ServerState.LOOKING) {
	        // 创建Vote对象  
            currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());  
        }  
    }
    // 选举算法
    // 
    this.electionAlg = createElectionAlgorithm(electionType);  
}

public Vote(long id, long zxid, long peerEpoch) {  
    this.version = 0x0;  
    this.id = id;  
    this.zxid = zxid;  
    this.electionEpoch = -1;  
    this.peerEpoch = peerEpoch;  
    this.state = ServerState.LOOKING;  
}

protected Election createElectionAlgorithm(int electionAlgorithm) {  
    Election le = null;  
  
    //TODO: use a factory rather than a switch  
    switch (electionAlgorithm) {  
    case 1:  
        throw new UnsupportedOperationException("Election Algorithm 1 is not supported.");  
    case 2:  
        throw new UnsupportedOperationException("Election Algorithm 2 is not supported.");  
    case 3:  
        QuorumCnxManager qcm = createCnxnManager();  
        QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);  
        QuorumCnxManager.Listener listener = qcm.listener;  
        if (listener != null) {  
            listener.start();  
            FastLeaderElection fle = new FastLeaderElection(this, qcm);  
            // 启动 WorkerSender 和 WorkerReceiver线程
            fle.start();  
            le = fle;  
        }  
        break;  
    default:  
        assert false;  
    }  
    return le;  
}

public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager) { 
    this.stop = false;  
    this.manager = manager;  
    starter(self, manager);  
}

private void starter(QuorumPeer self, QuorumCnxManager manager) {  
    this.self = self;  
    proposedLeader = -1;  
    proposedZxid = -1;  
  
    sendqueue = new LinkedBlockingQueue<ToSend>();  
    recvqueue = new LinkedBlockingQueue<Notification>();  
    this.messenger = new Messenger(manager);  
}
```
### 1.3.3 选举逻辑
```java
public void run() {   
  
    try {  
        /*  
         * Main loop         */        
         while (running) {  
    
            switch (getPeerState()) {  
            case LOOKING:  
                LOG.info("LOOKING");  
                    // Create read-only server but don't start it immediately  
                    final ReadOnlyZooKeeperServer roZk = new ReadOnlyZooKeeperServer(logFactory, this, this.zkDb);  
  
                    // Instead of starting roZk immediately, wait some grace  
                    // period before we decide we're partitioned.                    //                    // Thread is used here because otherwise it would require                    // changes in each of election strategy classes which is                    // unnecessary code coupling.                    
                    Thread roZkMgr = new Thread() {  
                        public void run() {  
                            try {  
                                // lower-bound grace period to 2 secs  
                                sleep(Math.max(2000, tickTime));  
                                if (ServerState.LOOKING.equals(getPeerState())) {  
                                    roZk.startup();  
                                }  
                            } catch (InterruptedException e) {  
                                LOG.info("Interrupted while attempting to start ReadOnlyZooKeeperServer, not started");  
                            } catch (Exception e) {  
                                LOG.error("FAILED to start ReadOnlyZooKeeperServer", e);  
                            }  
                        }  
                    };  
                    try {  
                        roZkMgr.start();  
                        reconfigFlagClear();  
                        if (shuttingDownLE) {  
                            shuttingDownLE = false;  
                            startLeaderElection();  
                        }  
                        setCurrentVote(makeLEStrategy().lookForLeader());  
                    } catch (Exception e) {  
                        LOG.warn("Unexpected exception", e);  
                        setPeerState(ServerState.LOOKING);  
                    } finally {  
                        // If the thread is in the the grace period, interrupt  
                        // to come out of waiting.                        roZkMgr.interrupt();  
                        roZk.shutdown();  
                    }  
                } else {  
                    try {  
                        reconfigFlagClear();  
                        if (shuttingDownLE) {  
                            shuttingDownLE = false;  
                            startLeaderElection();  
                        }  
                        setCurrentVote(makeLEStrategy().lookForLeader());  
                    } catch (Exception e) {  
                        LOG.warn("Unexpected exception", e);  
                        setPeerState(ServerState.LOOKING);  
                    }  
                }  
                break;  
            case OBSERVING:  
                try {  
                    LOG.info("OBSERVING");  
                    setObserver(makeObserver(logFactory));  
                    observer.observeLeader();  
                } catch (Exception e) {  
                    LOG.warn("Unexpected exception", e);  
                } finally {  
                    observer.shutdown();  
                    setObserver(null);  
                    updateServerState();  
  
                    // Add delay jitter before we switch to LOOKING  
                    // state to reduce the load of ObserverMaster                    if (isRunning()) {  
                        Observer.waitForObserverElectionDelay();  
                    }  
                }  
                break;  
            case FOLLOWING:  
                try {  
                    LOG.info("FOLLOWING");  
                    setFollower(makeFollower(logFactory));  
                    follower.followLeader();  
                } catch (Exception e) {  
                    LOG.warn("Unexpected exception", e);  
                } finally {  
                    follower.shutdown();  
                    setFollower(null);  
                    updateServerState();  
                }  
                break;  
            case LEADING:  
                LOG.info("LEADING");  
                try {  
                    setLeader(makeLeader(logFactory));  
                    leader.lead();  
                    setLeader(null);  
                } catch (Exception e) {  
                    LOG.warn("Unexpected exception", e);  
                } finally {  
                    if (leader != null) {  
                        leader.shutdown("Forcing shutdown");  
                        setLeader(null);  
                    }  
                    updateServerState();  
                }  
                break;  
            }  
        }  
    } finally {  
        LOG.warn("QuorumPeer main thread exited");  
        MBeanRegistry instance = MBeanRegistry.getInstance();  
        instance.unregister(jmxQuorumBean);  
        instance.unregister(jmxLocalPeerBean);  
  
        for (RemotePeerBean remotePeerBean : jmxRemotePeerBean.values()) {  
            instance.unregister(remotePeerBean);  
        }  
  
        jmxQuorumBean = null;  
        jmxLocalPeerBean = null;  
        jmxRemotePeerBean = null;  
    }  
}
```
