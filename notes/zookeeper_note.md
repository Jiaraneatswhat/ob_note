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
#### 1.3.2.1 启动 WorkerSender
```java
class WorkerSender extends ZooKeeperThread {  
  
    volatile boolean stop;  
    QuorumCnxManager manager;  
  
    WorkerSender(QuorumCnxManager manager) {  
        super("WorkerSender");  
        this.stop = false;  
        this.manager = manager;  
    }  
  
    public void run() {  
        while (!stop) {  
            try {  
	            // 从sendqueue取出一个ToSend对象交给QuorumCnxManager进行处理
                ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);  
                if (m == null) {  
                    continue;  
                }  
                process(m);  
            } catch (InterruptedException e) {  
                break;  
            }  
        }  
        LOG.info("WorkerSender is down");  
    }     
     void process(ToSend m) {  
        ByteBuffer requestBuffer = buildMsg(m.state.ordinal(), m.leader, m.zxid, m.electionEpoch, m.peerEpoch, m.configData);  
  
        manager.toSend(m.sid, requestBuffer);  
    }  

	public void toSend(Long sid, ByteBuffer b) {  
    // sid 相同，说明是发给自己的，加入到 recvQueue 中  
	    if (this.mySid == sid) {  
	        b.position(0);  
	        addToRecvQueue(new Message(b.duplicate(), sid));  
		} else {  
			 // 否则建立链接发给别人
			 BlockingQueue<ByteBuffer> bq = queueSendMap.computeIfAbsent(sid, serverId -> new CircularBlockingQueue<>(SEND_CAPACITY));  
	        addToSendQueue(bq, b);  
	        connectOne(sid);  
		    }  
		}
	}
}
```
#### 1.3.2.2 启动 WorkerReceiver
```java
// run()
// 不断地从QuorumCnxManager中获取其他服务器发来的选举消息，并将其转换成一个选票，然后保存到recvqueue中，在选票接收过程中，如果发现该外部选票的选举轮次小于当前服务器的，那么忽略该外部投票，同时立即发送自己的内部投票

// 选举轮次小于当前和下一轮
if (!validVoter(response.sid)) {  
	// 获取自己的投票
	Vote current = self.getCurrentVote();  
	QuorumVerifier qv = self.getQuorumVerifier();  
	ToSend notmsg = new ToSend(  
		ToSend.mType.notification,  
		current.getId(),  
		current.getZxid(),  
		logicalclock.get(),  
		self.getPeerState(),  
		response.sid,  
		current.getPeerEpoch(),  
		qv.toString().getBytes(UTF_8));  
	// 放入sendqueue中
	sendqueue.offer(notmsg);  
} else {  
	// 包含在可投票的服务器集合中，则根据 Message 解析出投票服务器的投票信息并将其封装为 Notification 
	if (self.getPeerState() == QuorumPeer.ServerState.LOOKING) {  
		recvqueue.offer(n);  
		// 选举的 epoch < 逻辑时钟
	    if ((ackstate == QuorumPeer.ServerState.LOOKING)  
                            && (n.electionEpoch < logicalclock.get())) {  
        // 创建新的投票
		Vote v = getVote();  
		QuorumVerifier qv = self.getQuorumVerifier();  
		ToSend notmsg = new ToSend(...);  
		sendqueue.offer(notmsg);  
		}  
	} else {  
	Vote current = self.getCurrentVote();  
	if (ackstate == QuorumPeer.ServerState.LOOKING) {  
		if (self.leader != null) {  
			if (leadingVoteSet != null) {  
		        self.leader.setLeadingVoteSet(leadingVoteSet);
		        leadingVoteSet = null;  
                                }  
                self.leader.reportLookingSid(response.sid);  
                            }  
				QuorumVerifier qv = self.getQuorumVerifier();  
				ToSend notmsg = new ToSend(  
					ToSend.mType.notification,  
					current.getId(),  
					current.getZxid(),  
					current.getElectionEpoch(),  
					self.getPeerState(),  
					response.sid,  
					current.getPeerEpoch(),  
					qv.toString().getBytes());  
				sendqueue.offer(notmsg);  
                        }  
                    }  
                }  
            }
}
```
### 1.3.3 开启选举
```java
public void run() {   
  
    try {  
        /*  
         * Main loop         */        
         while (running) {  
            if (unavailableStartTime == 0) {  
                unavailableStartTime = Time.currentElapsedTime();  
            }  
  
            switch (getPeerState()) {  
            case LOOKING:  
                LOG.info("LOOKING");  
  
                if (Boolean.getBoolean("readonlymode.enabled")) { 
                // 默认false
                } else {  
                    try {  
                    setCurrentVote(makeLEStrategy().lookForLeader());  
                    } 
                }  
                break;   
        }  
    }  
}
```
### 1.3.4 选举逻辑
#### 1.3.4.1 启动时投自己一票
```java
public Vote lookForLeader() throws InterruptedException {   
  
    try {  
         // 存放当前选举的Vote对象
	    Map<Long, Vote> recvset = new HashMap<Long, Vote>();  
  
        // 存放当前和此次选举的Vote对象
        Map<Long, Vote> outofelection = new HashMap<Long, Vote>();  
  
        synchronized (this) {  
            logicalclock.incrementAndGet();  
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());  
        }  
  
        LOG.info(  
            "New election. My id = {}, proposed zxid=0x{}",  
            self.getMyId(),  
            Long.toHexString(proposedZxid));  
        sendNotifications();  
		// 交换选票
		while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {...}
  }
}

// 第一次默认投自己，并更新当前服务协议的领导者信息的值proposedLeader、proposedZxid、proposedEpoch
synchronized void updateProposal(long leader, long zxid, long epoch) { 
    proposedLeader = leader;  
    proposedZxid = zxid;  
    proposedEpoch = epoch;  
}

// 广播自己的投票
private void sendNotifications() {  
    for (long sid : self.getCurrentAndNextConfigVoters()) {  
        QuorumVerifier qv = self.getQuorumVerifier();  
        // 封装一个ToSend对象放入队列
        ToSend notmsg = new ToSend(  
            ToSend.mType.notification,  
            proposedLeader,  
            proposedZxid,  
            logicalclock.get(),  
            QuorumPeer.ServerState.LOOKING,  
            sid,  
            proposedEpoch,  
            qv.toString().getBytes(UTF_8));   
  
        sendqueue.offer(notmsg);  
    }  
}
```
#### 1.3.4.2 交换选票
```java
while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {  
    /*  
     * Remove next notification from queue, times out after 2 times     * the termination time     
     */    
     Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);  
  
    /*  
     * Sends more notifications if haven't received enough.     * Otherwise processes new notification.     */    
     if (n == null) {  
        if (manager.haveDelivered()) {  
            sendNotifications();  
        } else {  
            manager.connectAll();  
        }  
  
        /*  
         * Exponential backoff         */        int tmpTimeOut = notTimeout * 2;  
        notTimeout = Math.min(tmpTimeOut, maxNotificationInterval);  
        LOG.info("Notification time out: {}", notTimeout);  
    } else if (validVoter(n.sid) && validVoter(n.leader)) {  
        /*  
         * Only proceed if the vote comes from a replica in the current or next         * voting view for a replica in the current or next voting view.         */        switch (n.state) {  
        case LOOKING:  
            if (getInitLastLoggedZxid() == -1) {  
                LOG.debug("Ignoring notification as our zxid is -1");  
                break;  
            }  
            if (n.zxid == -1) {  
                LOG.debug("Ignoring notification from member with -1 zxid {}", n.sid);  
                break;  
            }  
            // If notification > current, replace and send messages out  
            if (n.electionEpoch > logicalclock.get()) {  
                logicalclock.set(n.electionEpoch);  
                recvset.clear();  
                if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {  
                    updateProposal(n.leader, n.zxid, n.peerEpoch);  
                } else {  
                    updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());  
                }  
                sendNotifications();  
            } else if (n.electionEpoch < logicalclock.get()) {  
                    LOG.debug(  
                        "Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x{}, logicalclock=0x{}",  
                        Long.toHexString(n.electionEpoch),  
                        Long.toHexString(logicalclock.get()));  
                break;  
            } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {  
                updateProposal(n.leader, n.zxid, n.peerEpoch);  
                sendNotifications();  
            }  
  
            LOG.debug(  
                "Adding vote: from={}, proposed leader={}, proposed zxid=0x{}, proposed election epoch=0x{}",  
                n.sid,  
                n.leader,  
                Long.toHexString(n.zxid),  
                Long.toHexString(n.electionEpoch));  
  
            // don't care about the version if it's in LOOKING state  
            recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));  
  
            voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));  
  
            if (voteSet.hasAllQuorums()) {  
  
                // Verify if there is any change in the proposed leader  
                while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {  
                    if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {  
                        recvqueue.put(n);  
                        break;  
                    }  
                }  
  
                /*  
                 * This predicate is true once we don't read any new                 * relevant message from the reception queue                 */                if (n == null) {  
                    setPeerState(proposedLeader, voteSet);  
                    Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);  
                    leaveInstance(endVote);  
                    return endVote;  
                }  
            }  
            break;  
        case OBSERVING:  
            LOG.debug("Notification from observer: {}", n.sid);  
            break;  
        case FOLLOWING:  
        case LEADING:  
            /*  
             * Consider all notifications from the same epoch             * together.             */            if (n.electionEpoch == logicalclock.get()) {  
                recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));  
                voteSet = getVoteTracker(recvset, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));  
                if (voteSet.hasAllQuorums() && checkLeader(recvset, n.leader, n.electionEpoch)) {  
                    setPeerState(n.leader, voteSet);  
                    Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);  
                    leaveInstance(endVote);  
                    return endVote;  
                }  
            }  
  
            /*  
             * Before joining an established ensemble, verify that             * a majority are following the same leader.             *             * Note that the outofelection map also stores votes from the current leader election.             * See ZOOKEEPER-1732 for more information.             */            outofelection.put(n.sid, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));  
            voteSet = getVoteTracker(outofelection, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));  
  
            if (voteSet.hasAllQuorums() && checkLeader(outofelection, n.leader, n.electionEpoch)) {  
                synchronized (this) {  
                    logicalclock.set(n.electionEpoch);  
                    setPeerState(n.leader, voteSet);  
                }  
                Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);  
                leaveInstance(endVote);  
                return endVote;  
            }  
            break;  
        default:  
            LOG.warn("Notification state unrecognized: {} (n.state), {}(n.sid)", n.state, n.sid);  
            break;  
        }  
    } else {  
        if (!validVoter(n.leader)) {  
            LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);  
        }  
        if (!validVoter(n.sid)) {  
            LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);  
        }  
    }  
}
```
