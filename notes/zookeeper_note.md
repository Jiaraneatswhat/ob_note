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
  
        if (config.getClientPortAddress() != null) { 
	         // 通过反射创建一个ServerCnxnFactory对象
            cnxnFactory = ServerCnxnFactory.createFactory();  
            cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), false);  
        }  
  
        if (config.getSecureClientPortAddress() != null) {  
            secureCnxnFactory = ServerCnxnFactory.createFactory();  
            secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), true);  
        }  
  
        quorumPeer = getQuorumPeer();  
        quorumPeer.setTxnFactory(new FileTxnSnapLog(config.getDataLogDir(), config.getDataDir()));  
        quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());  
        quorumPeer.enableLocalSessionsUpgrading(config.isLocalSessionsUpgradingEnabled());  
        //quorumPeer.setQuorumPeers(config.getAllMembers());  
        quorumPeer.setElectionType(config.getElectionAlg());  
        quorumPeer.setMyid(config.getServerId());  
        quorumPeer.setTickTime(config.getTickTime());  
        quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());  
        quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());  
        quorumPeer.setInitLimit(config.getInitLimit());  
        quorumPeer.setSyncLimit(config.getSyncLimit());  
        quorumPeer.setConnectToLearnerMasterLimit(config.getConnectToLearnerMasterLimit());  
        quorumPeer.setObserverMasterPort(config.getObserverMasterPort());  
        quorumPeer.setConfigFileName(config.getConfigFilename());  
        quorumPeer.setClientPortListenBacklog(config.getClientPortListenBacklog());  
        quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));  
        quorumPeer.setQuorumVerifier(config.getQuorumVerifier(), false);  
        if (config.getLastSeenQuorumVerifier() != null) {  
            quorumPeer.setLastSeenQuorumVerifier(config.getLastSeenQuorumVerifier(), false);  
        }  
        quorumPeer.initConfigInZKDatabase();  
        quorumPeer.setCnxnFactory(cnxnFactory);  
        quorumPeer.setSecureCnxnFactory(secureCnxnFactory);  
        quorumPeer.setSslQuorum(config.isSslQuorum());  
        quorumPeer.setUsePortUnification(config.shouldUsePortUnification());  
        quorumPeer.setLearnerType(config.getPeerType());  
        quorumPeer.setSyncEnabled(config.getSyncEnabled());  
        quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());  
        if (config.sslQuorumReloadCertFiles) {  
            quorumPeer.getX509Util().enableCertFileReloading();  
        }  
        quorumPeer.setMultiAddressEnabled(config.isMultiAddressEnabled());  
        quorumPeer.setMultiAddressReachabilityCheckEnabled(config.isMultiAddressReachabilityCheckEnabled());  
        quorumPeer.setMultiAddressReachabilityCheckTimeoutMs(config.getMultiAddressReachabilityCheckTimeoutMs());  
  
        // sets quorum sasl authentication configurations  
        quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);  
        if (quorumPeer.isQuorumSaslAuthEnabled()) {  
            quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);  
            quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);  
            quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);  
            quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);  
            quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);  
        }  
        quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);  
        quorumPeer.initialize();  
  
        if (config.jvmPauseMonitorToRun) {  
            quorumPeer.setJvmPauseMonitor(new JvmPauseMonitor(config));  
        }  
  
        quorumPeer.start();  
        ZKAuditProvider.addZKStartStopAuditLog();  
        quorumPeer.join();  
    } catch (InterruptedException e) {  
        // warn, but generally this is ok  
        LOG.warn("Quorum Peer interrupted", e);  
    } finally {  
        try {  
            metricsProvider.stop();  
        } catch (Throwable error) {  
            LOG.warn("Error while stopping metrics", error);  
        }  
    }  
}
```
