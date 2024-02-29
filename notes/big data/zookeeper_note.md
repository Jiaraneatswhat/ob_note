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
    try {  
    main.initializeAndRun(args);}
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
	    // 1. 服务端的通信组件的初始化
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
    // 配置类 QuorumPeerConfig 中指定了 int electionType = 3
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
	        // 逻辑时钟自增
            logicalclock.incrementAndGet(); 
            // 初始化选票 
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
    // 从 recvqueue取出选票 
     Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);  
	 // 没有获取到选票 
     if (n == null) {  
	    // 如果 manager 已经发送了所有选票
        if (manager.haveDelivered()) {  
	        // 向其他服务器发送消息
            sendNotifications();  
        } else {  
	        // 否则进行连接
            manager.connectAll();  
        }   
    } else if (validVoter(n.sid) && validVoter(n.leader)) { 
        switch (n.state) {  
        case LOOKING:  
	        // 首先比较epoch
            // If notification > current, replace and send messages out  
            // 服务器自身的选举轮次落后于该外部投票对应服务器的选举轮次
            if (n.electionEpoch > logicalclock.get()) { 
	            // 立即更新自己的选举轮次 
                logicalclock.set(n.electionEpoch);  
                // 清空所有已经收到的投票
                recvset.clear();  
                // 使用初始化的投票与外部投票做比对，判定是否变更内部投票, 处理完成之后，再将内部投票发送出去
                if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {  
                    updateProposal(n.leader, n.zxid, n.peerEpoch);  
                } else {  
                    updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());  
                }  
                sendNotifications();  
            // 外部投票的选举轮次小于内部投票, 直接忽略
            } else if (n.electionEpoch < logicalclock.get()) {  
                break;  
            } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {  
                updateProposal(n.leader, n.zxid, n.peerEpoch);  
                sendNotifications();  
            }  
  
            // don't care about the version if it's in LOOKING state  
            // 将刚收到的外部投票放入选票集合recvset中进行归档
            // recvset用于记录当前服务器在本轮次的Leader选举中收到的所有外部投票
            recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));  
			// 添加ack
            voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));  
			// 判断票数过半
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
            break;  }  
    }
}
```
##### 1.3.4.2.1 比较选票
```java
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {  
  
    if (self.getQuorumVerifier().getWeight(newId) == 0) {  
        return false;  
    }  
  
    /*  
     * We return true if one of the following three cases hold:     
     * 1- New epoch is higher     
     * 2- New epoch is the same as current epoch, but new zxid is higher     
     * 3- New epoch is the same as current epoch, new zxid is the same     
     *  as current zxid, but server id is higher.     *
     */  
    // 先比较 epoch, 再比较 zxid, 最后比较myid
    return ((newEpoch > curEpoch)  
            || ((newEpoch == curEpoch)  
                && ((newZxid > curZxid)  
                    || ((newZxid == curZxid)  
                        && (newId > curId)))));  
}
```
##### 1.3.4.2.2 添加ack
```java
// 给定 vote 集合，返回检查票数的 SyncedLearnerTracker
protected SyncedLearnerTracker getVoteTracker(Map<Long, Vote> votes, Vote vote) {  
    SyncedLearnerTracker voteSet = new SyncedLearnerTracker();  
    voteSet.addQuorumVerifier(self.getQuorumVerifier());  
    /*  
     * First make the views consistent. Sometimes peers will have different     
     * zxids for a server depending on timing.     
     */    
     for (Map.Entry<Long, Vote> entry : votes.entrySet()) {  
        if (vote.equals(entry.getValue())) {  
            voteSet.addAck(entry.getKey());  
        }  
    }  
    return voteSet;  
} 

public boolean addAck(Long sid) {  
    boolean change = false;  
    for (QuorumVerifierAcksetPair qvAckset : qvAcksetPairs) {  
        if (qvAckset.getQuorumVerifier().getVotingMembers().containsKey(sid)) {  
            qvAckset.getAckset().add(sid);  
            change = true;  
        }  
    }  
    return change;  
}
```
##### 1.3.4.2.3 判断半数
```java
public boolean hasAllQuorums() {  
    for (QuorumVerifierAcksetPair qvAckset : qvAcksetPairs) {  
        if (!qvAckset.getQuorumVerifier().containsQuorum(qvAckset.getAckset())) { 
            return false;  
        }  
    }  
    return true;  
}

// QuorumMaj.java
half = votingMembers.size() / 2;
public boolean containsQuorum(Set<Long> ackSet) {  
    return (ackSet.size() > half);  
}
```
#### 1.3.4.3 收尾
```java
if (voteSet.hasAllQuorums()) {  
  
    // Verify if there is any change in the proposed leader  
    // 遍历所有Notification，查看是否有更好的投票
    while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {  
        if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {  
            recvqueue.put(n);  
            break;  
        }  
    }  
  
    /*  
     * This predicate is true once we don't read any new     
     * relevant message from the reception queue     
     */    
     if (n == null) {  
	    // 更新Peer状态
        setPeerState(proposedLeader, voteSet);  
        Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);  
        // 清空recvqueue
        leaveInstance(endVote);  
        return endVote;  
    }  
}

private void setPeerState(long proposedLeader, SyncedLearnerTracker voteSet) {
	// 推举的 leader 跟自己的myid相同时，更新状态为LEADING
	// 否则调用learningState()
    ServerState ss = (proposedLeader == self.getMyId()) ? ServerState.LEADING : learningState();  
    self.setPeerState(ss);  
    if (ss == ServerState.LEADING) {  
        leadingVoteSet = voteSet;  
    }  
}

private ServerState learningState() { 
	// 3.3.0版本后新增加了 observer, 防止集群规模过大时，需要半数的ACK，增加 observer, 可以处理读，但是不参与投票
	// 既保证了集群的扩展性，又避免过多服务器参与投票导致的集群处理请求能力下降
    if (self.getLearnerType() == LearnerType.PARTICIPANT) {  
        LOG.debug("I am a participant: {}", self.getMyId());  
        return ServerState.FOLLOWING;  
    } else {  
        LOG.debug("I am an observer: {}", self.getMyId());  
        return ServerState.OBSERVING;  
    }  
}
```
### 1.3.5 Follower 开始 follow
```java
// 回到 1.3.3 QuorumPeer 的 run() 中:
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

// 创建一个follower对象
protected Follower makeFollower(FileTxnSnapLog logFactory) throws IOException {  
    return new Follower(this, new FollowerZooKeeperServer(logFactory, this, this.zkDb));  
}

// 跟随 leader
void followLeader() throws InterruptedException {  
    LOG.info("FOLLOWING - LEADER ELECTION TOOK - {} {}", electionTimeTaken, QuorumPeer.FLE_TIME_UNIT);  
    try {  
        self.setZabState(QuorumPeer.ZabState.DISCOVERY);  
        // 从 vote 中获取 leader 地址
        QuorumServer leaderServer = findLeader();  
        try {  
	        // 连接 leader 的 28881 端口
            connectToLeader(leaderServer.addr, leaderServer.hostname); 
            // 注册, 汇报自己的 zxid
            long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);  
	        // 向主节点查询数据数据恢复策略，并开始进行数据恢复
            syncWithLeader(newEpochZxid);  
            // 不停监听主节点的消息
            while (this.isRunning()) {  
                readPacket(qp);  
                processPacket(qp);  
            }  
        } 
    }
}
```
### 1.3.6 Leader 开始 lead
```java
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

// 创建leader对象
protected Leader makeLeader(FileTxnSnapLog logFactory) throws IOException, X509Exception {  
    return new Leader(this, new LeaderZooKeeperServer(logFactory, this, this.zkDb));  
}

void lead() throws IOException, InterruptedException {
    try {  
        // Start thread that waits for connection requests from  
        // new followers.
        // 监听连接       
        cnxAcceptor = new LearnerCnxAcceptor();  
        cnxAcceptor.start(); 
	    // 开启 zk 服务
        startZkServer();  
		// follower 心跳
        while (true) {  
            synchronized (this) {  
                long start = Time.currentElapsedTime();  
                long cur = start;  
                long end = start + self.tickTime / 2;  
                while (cur < end) {  
                    wait(end - cur);  
                    cur = Time.currentElapsedTime();  
                }  
  
                for (LearnerHandler f : getLearners()) {  
                    if (f.synced()) {  
	                    // 收到心跳添加到 syncedAckSet 中
                        syncedAckSet.addAck(f.getSid());  
                    }

				if (!tickSkip && !syncedAckSet.hasAllQuorums()) {  
					// 和半数以上的节点 ping 失败，则结束循环，退出 leader 角色
				    break;  
				}  
				tickSkip = !tickSkip;
			}  
            for (LearnerHandler f : getLearners()) {  
                f.ping();  
            }  
        }  
        if (shutdownMessage != null) {  
            shutdown(shutdownMessage);  
            // leader goes in looking state  
        }  
    } finally {  
        zk.unregisterJMX(this);  
    }  
}
```
#### 1.3.6.1 监听连接
```java
public void run() {  
    if (!stop.get() && !serverSockets.isEmpty()) {  
        ExecutorService executor = Executors.newFixedThreadPool(serverSockets.size());  
        CountDownLatch latch = new CountDownLatch(serverSockets.size());  
  
        serverSockets.forEach(serverSocket ->  
                executor.submit(new LearnerCnxAcceptorHandler(serverSocket, latch)));   
    }  
}

// LearnerCnxAcceptorHandler 是 Leader 的内部类
public void run() {  
    try {  
        Thread.currentThread().setName("LearnerCnxAcceptorHandler-" + serverSocket.getLocalSocketAddress());  
  
        while (!stop.get()) {  
            acceptConnections();  
        }  
    }
}

private void acceptConnections() throws IOException {  
    Socket socket = null;  
    boolean error = false;  
    try {  
        LearnerHandler fh = new LearnerHandler(socket, is, Leader.this);  
        fh.start();  
    } 
}

// LearnerHandler 监听 follower 的数据包（一对一）
public void run() {  
    try {  
        learnerMaster.addLearnerHandler(this); 
  
        QuorumPacket qp = new QuorumPacket();  
        ia.readRecord(qp, "packet");  
        // 传输 NEWLEADER 包, 广播给所有 follower，自己是 new leader
        if (getVersion() < 0x10000) {  
            QuorumPacket newLeaderQP = new QuorumPacket(Leader.NEWLEADER, newLeaderZxid, null, null);  
            oa.writeRecord(newLeaderQP, "packet");  
        } else {  
            QuorumPacket newLeaderQP = new QuorumPacket(Leader.NEWLEADER, newLeaderZxid, learnerMaster.getQuorumVerifierBytes(), null);  
            queuedPackets.add(newLeaderQP);  
        }  
        bufferedOutput.flush();  
  
        // Start thread that blast packets in the queue to learner  
        startSendingPackets();  
      
        qp = new QuorumPacket();  
        ia.readRecord(qp, "packet");  
  
        learnerMaster.waitForNewLeaderAck(getSid(), qp.getZxid());  
  
        syncLimitCheck.start();  
        // sync ends when NEWLEADER-ACK is received  
        syncThrottler.endSync();  
        if (needSnap) {  
			try {
				// 告诉 follower 采取的恢复策略  
				oa.writeRecord(new QuorumPacket(Leader.SNAP, zxidToSend, null, null), "packet");  
    messageTracker.trackSent(Leader.SNAP);  
    bufferedOutput.flush();
				  }
			  }
		// 不停接收来自 follower 的包
        while (true) {  
            qp = new QuorumPacket();  
            ia.readRecord(qp, "packet");  
        }  
    }
}
```

## 1.4 启动 Cli
```java
// zkCli.sh 运行 org.apache.zookeeper.ZooKeeperMain
public static void main(String[] args) throws IOException, InterruptedException {  
    ZooKeeperMain main = new ZooKeeperMain(args);  
    main.run();  
}

void run() throws IOException, InterruptedException {  
    if (cl.getCommand() == null) {...} 
    else {  
        // Command line args non-null.  Run what was passed.  
        processCmd(cl);  
    }  
}

protected boolean processCmd(MyCommandOptions co) throws IOException, InterruptedException {  
    boolean watch = false;  
    try {  
        watch = processZKCmd(co);  
    }
    return watch;  
}

protected boolean processZKCmd(MyCommandOptions co) throws CliException, IOException, InterruptedException {  
    String[] args = co.getArgArray();  
    String cmd = co.getCommand();  
    // 连接server 
	conncectToZk();
    // execute from commandMap  
    CliCommand cliCmd = commandMapCli.get(cmd);  
    if (cliCmd != null) {  
        cliCmd.setZk(zk);  
        watch = cliCmd.parse(args).exec();  
    }
    return watch;  
}

protected void connectToZK(String newHost) throws InterruptedException, IOException {  
  
    host = newHost;  
  
    ZKClientConfig clientConfig = null;  
  
    if (cl.getOption("client-configuration") != null) {  
        try {  
            clientConfig = new ZKClientConfig(cl.getOption("client-configuration"));  
        } 
    }  
    int timeout = Integer.parseInt(cl.getOption("timeout"));  
    zk = new ZooKeeperAdmin(host, timeout, new MyWatcher(), readOnly, clientConfig);   
}

public ZooKeeperAdmin(  
    String connectString,  
    int sessionTimeout,  
    Watcher watcher) throws IOException {  
    // ZooKeeperAdmin 继承了 ZooKeeper
    super(connectString, sessionTimeout, watcher);  
}

public ZooKeeper(  
    String connectString,  
    int sessionTimeout,  
    Watcher watcher,  
    boolean canBeReadOnly,  
    HostProvider hostProvider,  
    ZKClientConfig clientConfig  
) throws IOException {  
    this.clientConfig = clientConfig != null ? clientConfig : new ZKClientConfig();  
    this.hostProvider = hostProvider;  
    ConnectStringParser connectStringParser = new ConnectStringParser(connectString);  

	// 创建连接对象并启动
    cnxn = createConnection(  
        connectStringParser.getChrootPath(),  
        hostProvider,  
        sessionTimeout,  
        this.clientConfig,  
        watcher,  
        getClientCnxnSocket(),  
        canBeReadOnly);  
    cnxn.start();  
}
```
# 2. 数据结构
- `zk` 在内存中维护了一个类似文件系统的树形结构实现命名空间，树中的节点称为 `znode`
## 2.1 DataTree
```java
public class DataTree {

	// zk 自己封装的一个 ConcurrentHashMap，用于存放 DataNode
	// 保存树形结构的 所有路径-节点对象 映射关系
	private final NodeHashMap nodes;
	// 根节点
	private static final String rootZookeeper = "/";
	// 初始化时创建 /zookeeper 路径节点, 作为管理节点
	private static final String procZookeeper = Quotas.procZookeeper;
	// 实现配额管理的父节点 /zookeeper/quota
	private static final String quotaZookeeper = Quotas.quotaZookeeper;
	// 监听事件
	private IWatchManager dataWatches;
	private IWatchManager childWatches;
	// 保存 quota 节点的路径树信息
	// 统计某个节点下的节点数量及数据大小, 在不修改原有节点数据结构的情况下，通过 quota 保存相关信息，将路径信息以单词字典树的形式存放在 PathTrie 中
	private final PathTrie pTrie = new PathTrie();
}
```
## 2.2 DataNode
```java
// DataNode 继承自 Record类
// zk 中数据的 serde 使用 Jute 组件中的 Record 接口，定义了序列化/反序列化两个方法
public class DataNode implements Record {  

    // the data for this datanode
    // 父节点不存储数据, 最大存储数据量是1M
    byte[] data;  
	// 记录客户端对znode节点的访问权限
    Long acl;  
	// 节点的状态信息，事务id、版本号、时间戳等
    public StatPersisted stat;  
    // 子节点引用信息 
    private Set<String> children = null;
}

public class Stat implements Record {  
  private long czxid; // 节点事务id
  private long mzxid; // 节点最后一次修改事务id
  private long ctime; // 节点创建时间
  private long mtime; // 最后一次修改时间
  private int version;  
  private int cversion;  
  private int aversion;  
  private long ephemeralOwner; // 持久化 -> 0, 临时 -> session id
  private int dataLength;  
  private int numChildren;  
  private long pzxid; 
}
```
### 2.2.1 节点的类型
- 按生命周期分：持久节点和临时节点
- 按是否有序分：顺序节点和一般节点
- 持久节点的存活时间不依赖于客户端会话，只有客户端在显式执行删除节点操作时，节点才消失
- 临时节点的存活时间依赖于客户端会话，当会话结束，临时节点将会被自动删除
- <font color='red'> 临时节点不能拥有子节点 </font>
- 创建顺序节点时，zk 会在路径后面自动追加一个 **递增的序列号** ，可以保证在**同一个父节点下是唯一的**，利用该特性可以实现**分布式锁**等功能
```java
// StatPersisted 中定义了 long 类型的 ephemeralOwner
// DataNode 通过 getClientEphemeralOwner 可以获取到节点的持久化类型
private static long getClientEphemeralOwner(StatPersisted stat) {  
    EphemeralType ephemeralType = EphemeralType.get(stat.getEphemeralOwner());  
    // 持久节点
    if (ephemeralType != EphemeralType.NORMAL) {  
        return 0;  
    }  
    // 否则返回创建临时节点的SessionId
    return stat.getEphemeralOwner();  
}

public enum EphemeralType {  
    /**  
     * Not ephemeral     */    
     VOID,  
    /**  
     * Standard, pre-3.5.x EPHEMERAL     */   
     NORMAL,  
    /**  
     * Container node     */    
     CONTAINER,  
    /**  
     * TTL node     */    
     TTL() {...}
}
```

# 3. Watcher 监听流程
- `DataTree` 中有两个 `IWatchManager` 对象，`dataWatches` 和 `childWatches`，针对不同事件，会交给不同的 `Watcher` 进行处理
- `dataWatches` 管理的是触发节点本身，而 `childWatches` 管理的则是触发节点的父节点
	- 客户端调用 `exists()` 和 `getData()` 方法，服务器端接收到需要监听的请求后将会把路径和监听对象交给 `dataWatches` 管理
	- 客户端调用 `getChildren()` 方法，服务器端接收到需要监听的请求后会把路径和监听对象交给 `childWatches` 管理
	- 发生节点的增删时，`childWatches` 只会响应 `NodeCreated`、`NodeDeleted` 两种操作
## 3.1 Cli 发起监听操作(以 exists() 为例)

- Cli 在调用 `exists()`、`getData()` 和 `getChildren()` 三个方法的时候，如果传了 `Watcher` 对象，客户端将会把这个对象和对应的路径保存在本地
- `WatchRegistration` 决定将 `Watcher` 存放在哪个映射表中
- `exists()` 对应 `ExistsWatchRegistration`，存放在 `dataWatches` 中
- `getData()` 对应 `DataWatchRegistration`，存放在 `dataWatches` 中
- `getChildren()` `对应ChildWatchRegistration`

```java
public Stat exists(final String path, Watcher watcher) throws KeeperException, InterruptedException {  
  
    WatchRegistration wcb = null;  
    if (watcher != null) {  
	    // 传入watcher，创建对应的ExistsWatchRegistration
        wcb = new ExistsWatchRegistration(watcher, clientPath);  
    }  

	// request 对象
    ExistsRequest request = new ExistsRequest();  
    // 客户端会根据此值来判断请求是否需要添加到监听表中
    request.setWatch(watcher != null);  
    SetDataResponse response = new SetDataResponse();  
    // 使用ClientCnxn对象发送 Packet
    // ClientCnxn 类主要负责维护客户端与服务端的网络连接和信息交互
    cnxn.queuePacket(h, new ReplyHeader(), request, response, cb, clientPath, serverPath, ctx, wcb);
    return response.getStat().getCzxid() == -1 ? null : response.getStat();  
}

public Packet queuePacket(...) {  
    Packet packet = null;  
    // 创建packet 
    packet = new Packet(h, r, request, response, watchRegistration);  
    packet.watchDeregistration = watchDeregistration;  

	// cnxn 中定义了一个sendThread
	// packetAdded 唤醒 NIO 的 selector，将 packet 发往 Server
    sendThread.getClientCnxnSocket().packetAdded();  
    return packet;  
}
```
## 3.2 RequestProcessor 处理请求
- `RequestProccssor` 通过构建成不同的调用链来完成不同角色下对请求的处理的逻辑
- 
![[zk_request_processor.svg]]

- `SyncRequestProcessor` 会将事务请求写入到日志文件中进行持久化
- `AckRequestProcessor` 通知 `leader` 这个事务在本地已经同步完成，在 `leader` 确认已经有过半的服务器已经完成了事务日志的持久化后进行 `commit` 操作
- `FinalRequestProcessor` 真正处理客户端的请求并返回处理结果
```java
// 当客户端发送请求数据包 Packet 被服务端接收到后, 最终由FinalRequestProcessor 进行处理
public void processRequest(Request request) {

	ServerCnxn cnxn = request.cnxn;
	try{
		switch (request.type) {
			case OpCode.exists: {   
			    ExistsRequest existsRequest = new ExistsRequest();  
			    path = existsRequest.getPath(); 
			    // 把cnxn客户端对象当成 cli 监听器放到 DataTree 的监听表中
			    Stat stat = zks.getZKDatabase().statNode(path,                             existsRequest.getWatch() ? cnxn : null);  
		    rsp = new ExistsResponse(stat);  
		    requestPathMetricsCollector.registerRequest(request.type, path);  
		    break;  
			}
		}
	}
	// 返回响应
	try {  
    if (path == null || rsp == null) {...} 
    else {  
        int opCode = request.type;  
        switch (opCode) {  
            case OpCode.getData : {...}  
            case OpCode.getChildren2 : {...}  
            default:  
                responseSize = cnxn.sendResponse(hdr, rsp, "response");  
        }  
    }
}
```
## 3.3 客户端接收响应
```java
// cnxn 的内部类 SendThread 处理响应
void readResponse(ByteBuffer incomingBuffer) throws IOException {
	finishPacket(packet);
}

protected void finishPacket(Packet p) {  
    int err = p.replyHeader.getErr();  
    if (p.watchRegistration != null) {  
	    // 进行注册，将其添加到ZK客户端的监听表中
        p.watchRegistration.register(err);  
    }
}

// rc the result code of the operation that attempted to add the watch on the path
public void register(int rc) {  
    if (shouldAddWatch(rc)) {  
        Map<String, Set<Watcher>> watches = getWatches(rc);  
        synchronized (watches) {  
            Set<Watcher> watchers = watches.get(clientPath);  
            if (watchers == null) {  
                watchers = new HashSet<Watcher>();  
                watches.put(clientPath, watchers);  
            } 
            // 添加到 Set 中 
            watchers.add(watcher);  
        }  
    }  
}
```
## 3.4 server 节点内容发生变化
```java
// DataTree
public Stat setData(String path, byte[] data, int version, long zxid, long time) throws KeeperException.NoNodeException {
	// 对应 NodeDataChanged 事件
	// 触发监听
    dataWatches.triggerWatch(path, EventType.NodeDataChanged);  
    return s;  
}

public WatcherOrBitSet triggerWatch(String path, EventType type) {  
    return triggerWatch(path, type, null);  
}

public WatcherOrBitSet triggerWatch(String path, EventType type, WatcherOrBitSet supress) {  
    WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path);  
    Set<Watcher> watchers = new HashSet<>();  
    for (Watcher w : watchers) {  
        w.process(e);  
    }  
    return new WatcherOrBitSet(watchers);  
}

// NIOServerCnxn.java
public void process(WatchedEvent event) {  
	// int NOTIFICATION_XID = -1    -1表示事件触发通知
    ReplyHeader h = new ReplyHeader(ClientCnxn.NOTIFICATION_XID, -1L, 0);  
  
    // Convert WatchedEvent to a type that can be sent over the wire  
    WatcherEvent e = event.getWrapper();  
      
    int responseSize = sendResponse(h, e, "notification", null, null, ZooDefs.OpCode.error);  
}
```
## 3.5 客户端接收通知并回调监听
```java
void readResponse(ByteBuffer incomingBuffer) throws IOException {  
    ReplyHeader replyHdr = new ReplyHeader();  
	switch (replyHdr.getXid()) {
		case NOTIFICATION_XID: 
			// WatcherEvent 是事件通信对象
			// WatchedEvent 是客户端事件对象
		    WatcherEvent event = new WatcherEvent();  
		    event.deserialize(bbia, "response");  
		    WatchedEvent we = new WatchedEvent(event);
		    // 交给事件线程处理  
		    eventThread.queueEvent(we);  
	    return;
}

// cnxn 的内部类 EventThread
private void queueEvent(WatchedEvent event, Set<Watcher> materializedWatchers) {  
    final Set<Watcher> watchers;
    // 创建 WatcherSetEventPair 加入 LinkedBlockingQueue waitingEvents 中
    WatcherSetEventPair pair = new WatcherSetEventPair(watchers, event);  
    // queue the pair (watch set & event) for later processing  
    waitingEvents.add(pair);  
}

public void run() {  
    try {  
        isRunning = true;  
        while (true) {  
            Object event = waitingEvents.take();  
            if (event == eventOfDeath) {  
                wasKilled = true;  
            } else {  
	            // 处理事件
                processEvent(event);  
            }   
        }  
    } 
}

private void processEvent(Object event) {  
    try {  
        if (event instanceof WatcherSetEventPair) {  
            // each watcher will process the event  
            WatcherSetEventPair pair = (WatcherSetEventPair) event;  
            for (Watcher watcher : pair.watchers) {  
                try {  
	                // 调用客户端实现的监听器
                    watcher.process(pair.event);  
                }  
            }
		}
	}
}
```
# 4. Shell 操作
```shell
# 创建节点 
# -s 有序节点, -e 临时节点
create [-s] [-e] path [data] [acl]

# 删除节点
delete [-v version] path
# 删除节点及其子节点 
deleteall path

# 查看节点
ls 查看子节点名称
get 查看节点存储内容
```
# 5. 面试
- 1 ZK 是否满足 CAP 
	- `ZK` 可以保证 `CP`
	- `ZK` 不能保证每次服务请求的可用性
	- `ZK` 在选举 `Leader` 时是不可用状态
- 2 ZK 数据导致服务器磁盘空间满处理方法
	- ZK 会存储 log 和 snapshot 文件，这些文件通过 `autopurge.purgeInterval 默认 0，表示不开启` 和 `autopurge.snapRetainCount 默认 3` 控制
- 3 ZK 的分布式锁
	- `ZK` 通过每个线程在同一父目录下创建临时有序节点，通过比较节点 `id` 的大小来实现分布式锁功能
	- 通过 `ZK` 的 `Watch` 机制来实时获取节点的状态，业务执行完释放锁，删除节点后立即重新争抢锁
- 4 选举机制
	- 启动 `QuorumPeer` 进程，初始状态为 `LOOKING`，创建一个 `Vote` 对象
	- 准备选举阶段，开启两个线程 `WorkerSender` 和 `WorkerReceiver`，分别用于接收别人的选票和发送自己的选票
	- 每个节点上线时会先投自己一票，然后将选票进行广播，当一个节点收到其他节点广播的选票时，会进行比较
	- 先比较 `epoch`, 再比较 `zxid`, 最后比较 `myid`，更新选票
	- 选票超过半数后，将 `peerId` 与选票推选的 `Leader` 相同的节点设置为 `Leader`，其他的为 `Follower` 
- 5 ZAB(ZK 原子广播)
	- 消息广播：
		- `Leader` 将 `Client` 的请求转换为 `Proposal`
		- `Leader` 向 `Follower` 发送 `Proposal` 消息
		- `Follower` 收到 `Proposal` 后，将其以事务日志的方式写入本地磁盘，发送 `ack`
		- 收到半数以上的 `ack` 反馈后，向 `Follower` 发送 `Commit`，自身完成事务提交，`Follower` 接收到 `Commit` 消息后提交事务
	- 崩溃恢复
		- `Leader` 选举
		- 数据同步