# 1 HDFS
- HDFS 的架构
![[hdfs.svg]]
## 1.1 HDFS 的启动
### 1.1 脚本
#### 1.1.1 start-dfs.sh
```shell
#---------------------------------------------------------
# namenodes
# 索取namenode的配置
NAMENODES=$("${HADOOP_HDFS_HOME}/bin/hdfs" getconf -namenodes 2>/dev/null)

if [[ -z "${NAMENODES}" ]]; then
  NAMENODES=$(hostname)
fi

# 通过hdfs namenode 启动namenode
echo "Starting namenodes on [${NAMENODES}]"
hadoop_uservar_su hdfs namenode "${HADOOP_HDFS_HOME}/bin/hdfs" \
    --workers \
    --config "${HADOOP_CONF_DIR}" \
    --hostnames "${NAMENODES}" \
    --daemon start \
    namenode ${nameStartOpt}

HADOOP_JUMBO_RETCOUNTER=$?

#---------------------------------------------------------
# datanodes (using default workers file)

echo "Starting datanodes"
# 通过hdfs datanode 启动namenode
hadoop_uservar_su hdfs datanode "${HADOOP_HDFS_HOME}/bin/hdfs" \
    --workers \
    --config "${HADOOP_CONF_DIR}" \
    --daemon start \
    datanode ${dataStartOpt}
(( HADOOP_JUMBO_RETCOUNTER=HADOOP_JUMBO_RETCOUNTER + $? ))

#---------------------------------------------------------
# secondary namenodes (if any)

SECONDARY_NAMENODES=$("${HADOOP_HDFS_HOME}/bin/hdfs" getconf -secondarynamenodes 2>/dev/null)

if [[ -n "${SECONDARY_NAMENODES}" ]]; then

  if [[ "${NAMENODES}" =~ , ]]; then

    hadoop_error "WARNING: Highly available NameNode is configured."
    hadoop_error "WARNING: Skipping SecondaryNameNode."

  else

    if [[ "${SECONDARY_NAMENODES}" == "0.0.0.0" ]]; then
      SECONDARY_NAMENODES=$(hostname)
    fi

    echo "Starting secondary namenodes [${SECONDARY_NAMENODES}]"
    # # 通过hdfs secondarynamenode 启动secondarynamenode
    hadoop_uservar_su hdfs secondarynamenode "${HADOOP_HDFS_HOME}/bin/hdfs" \
      --workers \
      --config "${HADOOP_CONF_DIR}" \
      --hostnames "${SECONDARY_NAMENODES}" \
      --daemon start \
      secondarynamenode
    (( HADOOP_JUMBO_RETCOUNTER=HADOOP_JUMBO_RETCOUNTER + $? ))
  fi
fi
```
#### 1.1.2 hdfs
```shell
#!/usr/bin/env bash
## @description  Default command handler for hadoop command
## @audience     public
## @stability    stable
## @replaceable  no
## @param        CLI arguments
function hdfscmd_case
{
  subcmd=$1
  shift

  case ${subcmd} in
   # 设置HADOOP_SUBCMD_SUPPORTDAEMONIZATION为true，以及类名
    datanode)
      HADOOP_SUBCMD_SUPPORTDAEMONIZATION="true"
      HADOOP_SECURE_CLASSNAME="org.apache.hadoop.hdfs.server.datanode.SecureDataNodeStarter"
      HADOOP_CLASSNAME='org.apache.hadoop.hdfs.server.datanode.DataNode'
      hadoop_deprecate_envvar HADOOP_SECURE_DN_PID_DIR HADOOP_SECURE_PID_DIR
      hadoop_deprecate_envvar HADOOP_SECURE_DN_LOG_DIR HADOOP_SECURE_LOG_DIR
    ;;
    # 客户端交互时的shell
    dfs)
      HADOOP_CLASSNAME=org.apache.hadoop.fs.FsShell
    ;;
    # namenode
    namenode)
      HADOOP_SUBCMD_SUPPORTDAEMONIZATION="true"
      HADOOP_CLASSNAME='org.apache.hadoop.hdfs.server.namenode.NameNode'
      hadoop_add_param HADOOP_OPTS hdfs.audit.logger "-Dhdfs.audit.logger=${HDFS_AUDIT_LOGGER}"
    ;;
    # 2NN
    secondarynamenode)
      HADOOP_SUBCMD_SUPPORTDAEMONIZATION="true"
      HADOOP_CLASSNAME='org.apache.hadoop.hdfs.server.namenode.SecondaryNameNode'
      hadoop_add_param HADOOP_OPTS hdfs.audit.logger "-Dhdfs.audit.logger=${HDFS_AUDIT_LOGGER}"
    ;;
    ...
   esac
}

# let's locate libexec...
if [[ -n "${HADOOP_HOME}" ]]; then
  HADOOP_DEFAULT_LIBEXEC_DIR="${HADOOP_HOME}/libexec"
else
  bin=$(cd -P -- "$(dirname -- "${MYNAME}")" >/dev/null && pwd -P)
  HADOOP_DEFAULT_LIBEXEC_DIR="${bin}/../libexec"
fi

HADOOP_LIBEXEC_DIR="${HADOOP_LIBEXEC_DIR:-$HADOOP_DEFAULT_LIBEXEC_DIR}"
HADOOP_NEW_CONFIG=true
# 定位到libexec目录，如果存在hdfs-config.sh则执行
if [[ -f "${HADOOP_LIBEXEC_DIR}/hdfs-config.sh" ]]; then
  # shellcheck source=./hadoop-hdfs-project/hadoop-hdfs/src/main/bin/hdfs-config.sh
  . "${HADOOP_LIBEXEC_DIR}/hdfs-config.sh"
else
  echo "ERROR: Cannot execute ${HADOOP_LIBEXEC_DIR}/hdfs-config.sh." 2>&1
  exit 1
fi

# 调用/libexec/hadoop-functions.sh中定义的函数，准备执行命令
hadoop_generic_java_subcmd_handler
```
#### 1.1.3 hadoop-functions.sh
```java 
function hadoop_generic_java_subcmd_handler
{
  ...
  # do the hard work of launching a daemon or just executing our interactive
  # java class
  # 判断是否在安全模式下
  if [[ "${HADOOP_SUBCMD_SUPPORTDAEMONIZATION}" = true ]]; then
    ...
  else
    hadoop_java_exec "${HADOOP_SUBCMD}" "${HADOOP_CLASSNAME}" "${HADOOP_SUBCMD_ARGS[@]}"
  fi
}

# 运行java进程
function hadoop_java_exec
{
  # run a java command.  this is used for
  # non-daemons

  local command=$1
  local class=$2
  shift 2

  hadoop_debug "Final CLASSPATH: ${CLASSPATH}"
  hadoop_debug "Final HADOOP_OPTS: ${HADOOP_OPTS}"
  hadoop_debug "Final JAVA_HOME: ${JAVA_HOME}"
  hadoop_debug "java: ${JAVA}"
  hadoop_debug "Class name: ${class}"
  hadoop_debug "Command line options: $*"

  export CLASSPATH
  #shellcheck disable=SC2086
  exec "${JAVA}" "-Dproc_${command}" ${HADOOP_OPTS} "${class}" "$@"
}
```
### 1.2 启动 NameNode
#### 1.2.1 启动 9870 端口 服务
##### 1.2.1.1 NameNode.main()
```java
public static void main(String argv[]) throws Exception {
    if (DFSUtil.parseHelpArgument(argv, NameNode.USAGE, System.out, true)) {
        System.exit(0);
    }

    try {
        StringUtils.startupShutdownMessage(NameNode.class, argv, LOG);
        // 创建namenode
        NameNode namenode = createNameNode(argv, null);
        if (namenode != null) {
            namenode.join();
        }
    }
}
```
##### 1.2.1.2 createNameNode()
```java
public static NameNode createNameNode(String argv[], Configuration conf)
    throws IOException {
    LOG.info("createNameNode " + Arrays.asList(argv));
    // 初始化传进来的conf为null
    if (conf == null)
        // 创建HdfsConfiguration对象，会调用父类Configuration的构造器，加载默认参数
        conf = new HdfsConfiguration();
    // 解析两次参数
    // Parse out some generic args into Configuration.
    GenericOptionsParser hParser = new GenericOptionsParser(conf, argv);
    argv = hParser.getRemainingArgs();
    // Parse the rest, NN specific args.
    StartupOption startOpt = parseArguments(argv);
    if (startOpt == null) {
        printUsage(System.err);
        return null;
    }
    setStartupOption(conf, startOpt);

    boolean aborted = false;
    switch (startOpt) {
        // 初始化namenode:
        case FORMAT:
            aborted = format(conf, startOpt.getForceFormat(),
                             startOpt.getInteractiveFormat());
            terminate(aborted ? 1 : 0);
            return null; // avoid javac warning
        ...
        // 启动时调用
        default:
	        // 注册用于JMX监控的MBean对象
            DefaultMetricsSystem.initialize("NameNode");
            return new NameNode(conf);
    }
}
```
##### 1.2.1.3 new NameNode()
```java
public NameNode(Configuration conf) throws IOException {
    this(conf, NamenodeRole.NAMENODE);
}
// NamenodeRole是一个枚举类，定义了nn的状态
// NAMENODE  ("NameNode"),
// BACKUP    ("Backup Node"),
// CHECKPOINT("Checkpoint Node");
protected NameNode(Configuration conf, NamenodeRole role)
    throws IOException {
    ...
    try {
        initializeGenericKeys(conf, nsId, namenodeId);
        // 初始化
        initialize(getConf());
        state.prepareToEnterState(haContext);
        try {
            haContext.writeLock();
            state.enterState(haContext);
        } finally {
            haContext.writeUnlock();
        }
    } catch (IOException e) {
        ...
}
```
##### 1.2.1.4 initialize()
```java
protected void initialize(Configuration conf) throws IOException {
    if (conf.get(HADOOP_USER_GROUP_METRICS_PERCENTILES_INTERVALS) == null) {
        String intervals = conf.get(DFS_METRICS_PERCENTILES_INTERVALS_KEY);
        if (intervals != null) {
            conf.set(HADOOP_USER_GROUP_METRICS_PERCENTILES_INTERVALS,
                     intervals);
        }
    }

    UserGroupInformation.setConfiguration(conf);
    loginAsNameNodeUser(conf);
	// 初始化度量系统
    NameNode.initMetrics(conf, this.getRole());
    StartupProgressMetrics.register(startupProgress);
	
	...
	// nn的角色为NAMENODE时        
    if (NamenodeRole.NAMENODE == role) {
        startHttpServer(conf);
    }
	// 加载元数据
    loadNamesystem(conf);
    startAliasMapServerIfNecessary(conf);
	//创建PRCServer
    rpcServer = createRpcServer(conf);

    initReconfigurableBackoffKey();

    if (clientNamenodeAddress == null) {
        // This is expected for MiniDFSCluster. Set it now using 
        // the RPC server's bind address.
        clientNamenodeAddress = 
            NetUtils.getHostPortString(getNameNodeAddress());
        LOG.info("Clients are to use " + clientNamenodeAddress + " to access"
                 + " this namenode/service.");
    }
    if (NamenodeRole.NAMENODE == role) {
        httpServer.setNameNodeAddress(getNameNodeAddress());
        httpServer.setFSImage(getFSImage());
        if (levelDBAliasMapServer != null) {
            httpServer.setAliasMap(levelDBAliasMapServer.getAliasMap());
        }
    }
	// 检查启动资源
    startCommonServices(conf);
    startMetricsLogger(conf);
}
```
##### 1.2.1.5 startHttpServer()
```java
private void startHttpServer(final Configuration conf) throws IOException {
    // 新建NameNodeHttpServer对象
    httpServer = new NameNodeHttpServer(conf, this, getHttpServerBindAddress(conf));
    // 启动server
    httpServer.start();
    httpServer.setStartupProgress(startupProgress);
}

protected InetSocketAddress getHttpServerBindAddress(Configuration conf) {
    InetSocketAddress bindAddress = getHttpServerAddress(conf);

    // If DFS_NAMENODE_HTTP_BIND_HOST_KEY exists then it overrides the
    // host name portion of DFS_NAMENODE_HTTP_ADDRESS_KEY.
    final String bindHost = conf.getTrimmed(DFS_NAMENODE_HTTP_BIND_HOST_KEY);
    if (bindHost != null && !bindHost.isEmpty()) {
        bindAddress = new InetSocketAddress(bindHost, bindAddress.getPort());
    }

    return bindAddress;
}

protected InetSocketAddress getHttpServerAddress(Configuration conf) {
    return getHttpAddress(conf);
}
/*
 DFSConfigKeys.java中定义了:
 DFS_NAMENODE_HTTP_ADDRESS_DEFAULT = "0.0.0.0:" +  DFS_NAMENODE_HTTP_PORT_DEFAULT;
 DFS_NAMENODE_HTTP_PORT_DEFAULT =
      HdfsClientConfigKeys.DFS_NAMENODE_HTTP_PORT_DEFAULT;
 HdfsClientConfigKeys中定义了默认的端口号
 int DFS_NAMENODE_HTTP_PORT_DEFAULT = 9870;
*/
public static InetSocketAddress getHttpAddress(Configuration conf) {
    return  NetUtils.createSocketAddr(
        conf.getTrimmed(DFS_NAMENODE_HTTP_ADDRESS_KEY, DFS_NAMENODE_HTTP_ADDRESS_DEFAULT));
} // 通过 NetUtils.java 创建一个 Socket 地址

void start() throws IOException {
    ...
        // 创建一个hadoop封装的HttpServer的Builder对象
        HttpServer2.Builder builder = DFSUtil.httpServerTemplateForNNAndJN(conf,
                                                                           httpAddr, httpsAddr, "hdfs",
                                                                           DFSConfigKeys.DFS_NAMENODE_KERBEROS_INTERNAL_SPNEGO_PRINCIPAL_KEY, DFSConfigKeys.DFS_NAMENODE_KEYTAB_FILE_KEY);
	...
	// 创建httpServer对象
    httpServer = builder.build();
	...
    // 配置Servlet
    setupServlets(httpServer);
    // 启动WebServer
    httpServer.start();

    int connIdx = 0;
    if (policy.isHttpEnabled()) {
        httpAddress = httpServer.getConnectorAddress(connIdx++);
        conf.set(DFSConfigKeys.DFS_NAMENODE_HTTP_ADDRESS_KEY,
                 NetUtils.getHostPortString(httpAddress));
    }

    if (policy.isHttpsEnabled()) {
        httpsAddress = httpServer.getConnectorAddress(connIdx);
        conf.set(DFSConfigKeys.DFS_NAMENODE_HTTPS_ADDRESS_KEY,
                 NetUtils.getHostPortString(httpsAddress));
    }
}
```
#### 1.2.2 加载文件系统
```java
protected void loadNamesystem(Configuration conf) throws IOException {
    this.namesystem = FSNamesystem.loadFromDisk(conf);
}

static FSNamesystem loadFromDisk(Configuration conf) throws IOException {

    checkConfiguration(conf);
    // 新建FSImage对象
    FSImage fsImage = new FSImage(conf,
                                  // 获取NamespaceDirs和NamespaceEditsDirs
                                  FSNamesystem.getNamespaceDirs(conf),
                                  FSNamesystem.getNamespaceEditsDirs(conf));
    FSNamesystem namesystem = new FSNamesystem(conf, fsImage, false);
    StartupOption startOpt = NameNode.getStartupOption(conf);
    // RECOVER模式会进入安全模式
    if (startOpt == StartupOption.RECOVER) {
        namesystem.setSafeMode(SafeModeAction.SAFEMODE_ENTER);
    }

    long loadStart = monotonicNow();
    try {
        // 加载镜像文件
        namesystem.loadFSImage(startOpt);
    } catch (IOException ioe) {
        LOG.warn("Encountered exception loading fsimage", ioe);
        fsImage.close();
        throw ioe;
    }
    return namesystem;
}
```
##### 1.2.2.1 获取文件路径
```java
// FSNamesystem.java
// namespace位置
public static Collection<URI> getNamespaceDirs(Configuration conf) {
    return getStorageDirs(conf, DFS_NAMENODE_NAME_DIR_KEY);
}

private static Collection<URI> getStorageDirs(Configuration conf,
                                              String propertyName) {
    Collection<String> dirNames = conf.getTrimmedStringCollection(propertyName);
    StartupOption startOpt = NameNode.getStartupOption(conf);
    // StartupOption枚举类中定义: IMPORT  ("-importCheckpoint")
    if(startOpt == StartupOption.IMPORT) {
        ...
    } else if (dirNames.isEmpty()) {
        dirNames = Collections.singletonList(
            // 默认值"file:///tmp/hadoop/dfs/name"
            DFSConfigKeys.DFS_NAMENODE_EDITS_DIR_DEFAULT);
    }
    return Util.stringCollectionAsURIs(dirNames);
}
// edits文件位置
public static List<URI> getNamespaceEditsDirs(Configuration conf)
    throws IOException {
    return getNamespaceEditsDirs(conf, true);
}

public static List<URI> getNamespaceEditsDirs(Configuration conf,
                                              boolean includeShared)
    throws IOException {
    // Use a LinkedHashSet so that order is maintained while we de-dup
    // the entries.
    LinkedHashSet<URI> editsDirs = new LinkedHashSet<URI>();
	...
    // DFS_NAMENODE_EDITS_DIR_KEY默认值"dfs.namenode.edits.dir"
    for (URI dir : getStorageDirs(conf, DFS_NAMENODE_EDITS_DIR_KEY)) {
        if (!editsDirs.add(dir)) {
            LOG.warn("Edits URI " + dir + " listed multiple times in " + 
                     DFS_NAMENODE_SHARED_EDITS_DIR_KEY + " and " +
                     DFS_NAMENODE_EDITS_DIR_KEY + ". Ignoring duplicates.");
        }
    }
    if (editsDirs.isEmpty()) {
        // If this is the case, no edit dirs have been explicitly configured.
        // Image dirs are to be used for edits too.
        return Lists.newArrayList(getNamespaceDirs(conf));
    } else {
        return Lists.newArrayList(editsDirs);
    }
}
```
##### 1.2.2.2 加载 FSImage 文件
```java
private void loadFSImage(StartupOption startOpt) throws IOException {
    final FSImage fsImage = getFSImage();

    // format before starting up if requested
    // 格式化
    if (startOpt == StartupOption.FORMAT) {
        // reuse current id
        fsImage.format(this, fsImage.getStorage().determineClusterId(), false);

        startOpt = StartupOption.REGULAR;
    }
    boolean success = false;
    writeLock();
    try {
        // 创建MetaRecoveryContext对象用于读取fsImage文件
        MetaRecoveryContext recovery = startOpt.createRecoveryContext();
        final boolean staleImage
            // 读取fsImage
            = fsImage.recoverTransitionRead(startOpt, this, recovery);
        if (RollingUpgradeStartupOption.ROLLBACK.matches(startOpt)) {
            rollingUpgradeInfo = null;
        }
}

// FsImage.java
boolean recoverTransitionRead(StartupOption startOpt, FSNamesystem target,
                              MetaRecoveryContext recovery)
    throws IOException {
    assert startOpt != StartupOption.FORMAT : 
    "NameNode formatting should be performed before reading the image";
	
    // 获取路径
    Collection<URI> imageDirs = storage.getImageDirectories();
    Collection<URI> editsDirs = editLog.getEditURIs();

    switch(startOpt) {
        case UPGRADE:
        case UPGRADEONLY:
            doUpgrade(target);
            return false; // upgrade saved image already
        case IMPORT:
            doImportCheckpoint(target);
            return false; // import checkpoint saved image already
        case ROLLBACK:
            throw new AssertionError("Rollback is now a standalone command, " +
                                     "NameNode should not be starting with this option.");
        case REGULAR:
        // 启动时只进行加载
        default:
            // just load the image
    }
    return loadFSImage(target, startOpt, recovery);
}

private boolean loadFSImage(FSNamesystem target, StartupOption startOpt,
                            MetaRecoveryContext recovery)
    throws IOException {
    ...
    FSImageFile imageFile = null;
    for (int i = 0; i < imageFiles.size(); i++) {
        try {
            imageFile = imageFiles.get(i);
            loadFSImageFile(target, recovery, imageFile, startOpt);
            break;
        } catch (IllegalReservedPathException ie) {...}
    }
    ...
    return needToSave;
}

void loadFSImageFile(FSNamesystem target, MetaRecoveryContext recovery,
                     FSImageFile imageFile, StartupOption startupOption) throws IOException {
    LOG.info("Planning to load image: " + imageFile);
    StorageDirectory sdForProperties = imageFile.sd;
    storage.readProperties(sdForProperties, startupOption);

    if (NameNodeLayoutVersion.supports(
        LayoutVersion.Feature.TXID_BASED_LAYOUT, getLayoutVersion())) {
        ...
    } else if (NameNodeLayoutVersion.supports(
        // 通过CHECKSUM校验完整性
        LayoutVersion.Feature.FSIMAGE_CHECKSUM, getLayoutVersion())) {
        String md5 = storage.getDeprecatedProperty(
            NNStorage.DEPRECATED_MESSAGE_DIGEST_PROPERTY);
        if (md5 == null) {...}
        // 加载fsImage
        loadFSImage(imageFile.getFile(), new MD5Hash(md5), target, recovery,
                    false);
    } else {
        // We don't have any record of the md5sum
        loadFSImage(imageFile.getFile(), null, target, recovery, false);
    }
}

private void loadFSImage(File curFile, MD5Hash expectedMd5,
                         FSNamesystem target, MetaRecoveryContext recovery,
                         boolean requireSameLayoutVersion) throws IOException {
    // BlockPoolId is required when the FsImageLoader loads the rolling upgrade
    // information. Make sure the ID is properly set.
    target.setBlockPoolId(this.getBlockPoolID());
	
    // 通过LoaderDelegator对象来加载文件
    FSImageFormat.LoaderDelegator loader = FSImageFormat.newLoader(conf, target);
    loader.load(curFile, requireSameLayoutVersion);

    // 检验完整性
    MD5Hash readImageMd5 = loader.getLoadedImageMd5();
    if (expectedMd5 != null &&
        !expectedMd5.equals(readImageMd5)) {
        throw new IOException("Image file " + curFile +
                              " is corrupt with MD5 checksum of " + readImageMd5 +
                              " but expecting " + expectedMd5);
    }

    long txId = loader.getLoadedImageTxId();
    LOG.info("Loaded image for txid " + txId + " from " + curFile);
    lastAppliedTxId = txId;
    storage.setMostRecentCheckpointInfo(txId, curFile.lastModified());
}
```
#### 1.2.3 开启 RPC 服务
```java
protected NameNodeRpcServer createRpcServer(Configuration conf)
    throws IOException {
    return new NameNodeRpcServer(conf, this);
}

// NameNodeRpcServer.java
public NameNodeRpcServer(Configuration conf, NameNode nn)
    throws IOException {
    this.nn = nn;
    this.namesystem = nn.getNamesystem();
    this.retryCache = namesystem.getRetryCache();
    this.metrics = NameNode.getNameNodeMetrics();
    ...
    InetSocketAddress serviceRpcAddr = nn.getServiceRpcServerAddress(conf);
    if (serviceRpcAddr != null) {
        // 创建server
        serviceRpcServer = new RPC.Builder(conf)
            .setProtocol(
            org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolPB.class)
            .setInstance(clientNNPbService)
            .setBindAddress(bindHost)
            .setPort(serviceRpcAddr.getPort())
            .setNumHandlers(serviceHandlerCount)
            .setVerbose(false)
            .setSecretManager(namesystem.getDelegationTokenSecretManager())
            .build();
}
```
#### 1.2.4 启动资源检查
```java
private void startCommonServices(Configuration conf) throws IOException {
    namesystem.startCommonServices(conf, haContext);
}

void startCommonServices(Configuration conf, HAContext haContext) throws IOException {
    this.registerMBean(); // register the MBean for the FSNamesystemState
    writeLock();
    this.haContext = haContext;
    try {
        // 新建一个NameNodeResourceChecker对象检查资源
        // 在实例化时，会通过getRequiredNamespaceEditsDirs()获取需要检查的路径
        //  其中DFS_NAMENODE_DU_RESERVED_DEFAULT默认值是1024 * 1024 * 100
        nnResourceChecker = new NameNodeResourceChecker(conf);
        // 检查资源
        checkAvailableResources();
        assert !blockManager.isPopulatingReplQueues();
        StartupProgress prog = NameNode.getStartupProgress();
        prog.beginPhase(Phase.SAFEMODE);
        long completeBlocksTotal = getCompleteBlocksTotal();
        prog.setTotal(Phase.SAFEMODE, STEP_AWAITING_REPORTED_BLOCKS,
                      completeBlocksTotal);
        // 激活blockManager
        blockManager.activate(conf, completeBlocksTotal);
    }
}
```
##### 1.2.4.1 资源检查
```java
void checkAvailableResources() {
    long resourceCheckTime = monotonicNow();
    Preconditions.checkState(nnResourceChecker != null,
                             "nnResourceChecker not initialized");
    hasResourcesAvailable = nnResourceChecker.hasAvailableDiskSpace();
    resourceCheckTime = monotonicNow() - resourceCheckTime;
    NameNode.getNameNodeMetrics().addResourceCheckTime(resourceCheckTime);
}

public boolean hasAvailableDiskSpace() {
    return NameNodeResourcePolicy.areResourcesAvailable(
    volumes.values(), minimumRedundantVolumes);
}

// NameNodeResourcePolicy.java
static boolean areResourcesAvailable(
    Collection<? extends CheckableNameNodeResource> resources,
    int minimumRedundantResources) {
	...
    for (CheckableNameNodeResource resource : resources) {
        if (!resource.isRequired()) {
            redundantResourceCount++;
            // 判断是否可用
            if (!resource.isResourceAvailable()) {
                disabledRedundantResourceCount++;
            }
        } else {
            requiredResourceCount++;
            if (!resource.isResourceAvailable()) {
                // Short circuit - a required resource is not available.
                return false;
            }
        }
    }
}

// CheckableNameNodeResource接口有两个实现类：NameNodeResourceChecker以及JournalAndStream
// NameNodeResourceChecker.java
@Override
public boolean isResourceAvailable() {
    long availableSpace = df.getAvailable();
    // 检查大小是否够100M
    if (availableSpace < duReserved) {
        LOG.warn("Space available on volume '" + volume + "' is "
                 + availableSpace +
                 ", which is below the configured reserved amount " + duReserved);
        return false;
    } else {
        return true;
    }
}
```
##### 1.2.4.2 激活 BlockManager，进行心跳和安全模式的管理
```java
// BlockManager.java
public void activate(Configuration conf, long blockTotal) {
    pendingReconstruction.start();
    datanodeManager.activate(conf);
    this.redundancyThread.setName("RedundancyMonitor");
    this.redundancyThread.start();
    this.markedDeleteBlockScrubberThread.
        setName("MarkedDeleteBlockScrubberThread");
    this.markedDeleteBlockScrubberThread.start();
    this.blockReportThread.start();
    mxBeanName = MBeans.register("NameNode", "BlockStats", this);
    // 启动安全模式
    bmSafeMode.activate(blockTotal);
  }
```
###### 1.2.4.2.1 HeartbeatManager 进行心跳检查
```java
// DataNodeManager.java
void activate(final Configuration conf) {
    datanodeAdminManager.activate(conf);
    heartbeatManager.activate();
}

// HeartbeatManager.java
void activate() {
    heartbeatThread.start();
}

private final Daemon heartbeatThread = new Daemon(new Monitor());

@Override
public void run() {
    while(namesystem.isRunning()) {
        restartHeartbeatStopWatch();
        try {
            final long now = Time.monotonicNow();
            // lastHeartbeatCheck距离现在已经超过了heartbeatRecheckInterval
            if (lastHeartbeatCheck + heartbeatRecheckInterval < now) {
                // 进行check
                heartbeatCheck();
                // 更新check时间
                lastHeartbeatCheck = now;
            }
    }
}

void heartbeatCheck() {
    final DatanodeManager dm = blockManager.getDatanodeManager();
    // It's OK to check safe mode w/o taking the lock here, we re-check
    // for safe mode after taking the lock before removing a datanode.
    if (namesystem.isInStartupSafeMode()) {
        return;
    }
    boolean allAlive = false;
    while (!allAlive) {
        // locate the first dead node.
        DatanodeDescriptor dead = null;

        // locate the first failed storage that isn't on a dead node.
        DatanodeStorageInfo failedStorage = null;

        // check the number of stale storages
        int numOfStaleStorages = 0;
        List<DatanodeDescriptor> staleNodes = new ArrayList<>();
        synchronized(this) {
            // 遍历DN
            for (DatanodeDescriptor d : datanodes) {
                // check if an excessive GC pause has occurred
                if (shouldAbortHeartbeatCheck(0)) {
                    return;
                }
                // isDatanodeDead判断是否死亡
                if (dead == null && dm.isDatanodeDead(d)) {
                    stats.incrExpiredHeartbeats();
                    dead = d;
                    // remove the node from stale list to adjust the stale list size
                    // before setting the stale count of the DatanodeManager
                    removeNodeFromStaleList(d);
                } else {...}
}

boolean isDatanodeDead(DatanodeDescriptor node) {
    return (node.getLastUpdateMonotonic() <
            (monotonicNow() - heartbeatExpireInterval));
}

this.heartbeatExpireInterval = 2 * heartbeatRecheckInterval
        + 10 * 1000 * heartbeatIntervalSeconds; // 10min30s

heartbeatRecheckInterval = conf.getInt(
        DFSConfigKeys.DFS_NAMENODE_HEARTBEAT_RECHECK_INTERVAL_KEY, 
        DFSConfigKeys.DFS_NAMENODE_HEARTBEAT_RECHECK_INTERVAL_DEFAULT); 

public static final int DFS_NAMENODE_HEARTBEAT_RECHECK_INTERVAL_DEFAULT = 5 * 60 * 1000; // 5mins

heartbeatIntervalSeconds = conf.getTimeDuration(
        DFSConfigKeys.DFS_HEARTBEAT_INTERVAL_KEY,
        DFSConfigKeys.DFS_HEARTBEAT_INTERVAL_DEFAULT, TimeUnit.SECONDS);

// DFS_HEARTBEAT_INTERVAL_DEFAULT = 3; --> 3s
```
###### 1.2.4.2.2 BlockManagerSafeMode 安全模式管理
```java
// BlockManagerSafeMode.java
void activate(long total) {
    // total: 初始块的数量
    assert namesystem.hasWriteLock();
    assert status == BMSafeModeStatus.OFF;

    startTime = monotonicNow();
    setBlockTotal(total); // total * 0.999f
    if (areThresholdsMet()) {
        boolean exitResult = leaveSafeMode(false);
        Preconditions.checkState(exitResult, "Failed to leave safe mode.");
    } else {
        // enter safe mode
        status = BMSafeModeStatus.PENDING_THRESHOLD;
        initializeReplQueuesIfNecessary();
        reportStatus("STATE* Safe mode ON.", true);
        lastStatusReport = monotonicNow();
    }
}

void setBlockTotal(long total) {
    assert namesystem.hasWriteLock();
    synchronized (this) {
        this.blockTotal = total;
        this.blockThreshold = (long) (total * threshold);
    }
    this.blockReplQueueThreshold = (long) (total * replQueueThreshold);
}
// BlockManagerSafeMode的属性
//this.threshold = conf.getFloat(DFS_NAMENODE_SAFEMODE_THRESHOLD_PCT_KEY,
//        DFS_NAMENODE_SAFEMODE_THRESHOLD_PCT_DEFAULT);
//DFS_NAMENODE_SAFEMODE_THRESHOLD_PCT_DEFAULT = 0.999f;

private boolean areThresholdsMet() {
    assert namesystem.hasWriteLock();
    // blockSafe: 安全启动的block个数
    synchronized (this) {
        boolean isBlockThresholdMet = (blockSafe >= blockThreshold);
        boolean isDatanodeThresholdMet = true;
        // datanodeThreshold默认0
        if (isBlockThresholdMet && datanodeThreshold > 0) {
            int datanodeNum = blockManager.getDatanodeManager().
                getNumLiveDataNodes();
            isDatanodeThresholdMet = (datanodeNum >= datanodeThreshold);
        }
        return isBlockThresholdMet && isDatanodeThresholdMet;
    }
}
```
### 1.3 启动 DataNode
#### 1.3.1 创建 DN 对象
```java
public static void main(String args[]) {
    if (DFSUtil.parseHelpArgument(args, DataNode.USAGE, System.out, true)) {
        System.exit(0);
    }

    secureMain(args, null);
}

public static void secureMain(String args[], SecureResources resources) {
    int errorCode = 0;
    try {
        StringUtils.startupShutdownMessage(DataNode.class, args, LOG);
        // 创建DataNode
        DataNode datanode = createDataNode(args, null, resources);
        if (datanode != null) {
            // 线程阻塞
            datanode.join();
        }
        ...
    }
}

public static DataNode createDataNode(String args[], Configuration conf,
                                      SecureResources resources) throws IOException {
    // 初始化DataNode，返回DataNode对象
    DataNode dn = instantiateDataNode(args, conf, resources);
    if (dn != null) {
        // 启动DataNode后台线程
        dn.runDatanodeDaemon();
    }
    return dn;
}

public static DataNode instantiateDataNode(String args [], Configuration conf,
                                           SecureResources resources) throws IOException {
    if (conf == null)
        conf = new HdfsConfiguration();

    if (args != null) {
        // parse generic hadoop options
        GenericOptionsParser hParser = new GenericOptionsParser(conf, args);
        args = hParser.getRemainingArgs();
    }

    if (!parseArguments(args, conf)) {
        printUsage(System.err);
        return null;
    }
    // 获取存储位置
    // DFS_DATANODE_DATA_DIR_KEY = "dfs.datanode.data.dir"
    Collection<StorageLocation> dataLocations = getStorageLocations(conf);
    return makeInstance(dataLocations, conf, resources);
}

static DataNode makeInstance(Collection<StorageLocation> dataDirs,
                             Configuration conf, SecureResources resources) throws IOException {
    List<StorageLocation> locations;
    StorageLocationChecker storageLocationChecker =
        new StorageLocationChecker(conf, new Timer());
    try {
        // 确认可以创建dataDirs及其父目录
        locations = storageLocationChecker.check(conf, dataDirs);
    } catch (InterruptedException ie) {
        throw new IOException("Failed to instantiate DataNode", ie);
    }
    DefaultMetricsSystem.initialize("DataNode");

    assert locations.size() > 0 : "number of data directories should be > 0";
    return new DataNode(conf, locations, storageLocationChecker, resources);
}

DataNode(final Configuration conf,
         final List<StorageLocation> dataDirs,
         final StorageLocationChecker storageLocationChecker,
         final SecureResources resources) throws IOException {
    ...
    this.blockScanner = new BlockScanner(this);
    ...
    try {
        hostName = getHostName(conf);
        LOG.info("Configured hostname is {}", hostName);
        // 启动DataNode
        startDataNode(dataDirs, resources);
    } catch (IOException ie) {
        shutdown();
        throw ie;
    }
    final int dncCacheMaxSize =
        conf.getInt(DFS_DATANODE_NETWORK_COUNTS_CACHE_MAX_SIZE_KEY,
                    DFS_DATANODE_NETWORK_COUNTS_CACHE_MAX_SIZE_DEFAULT) ;
    // 构建者设计模式
    datanodeNetworkCounts =
        CacheBuilder.newBuilder()
        .maximumSize(dncCacheMaxSize)
        .build(new CacheLoader<String, Map<String, Long>>() {
            @Override
            public Map<String, Long> load(String key) {
                final Map<String, Long> ret = new ConcurrentHashMap<>();
                ret.put(NETWORK_ERRORS, 0L);
                return ret;
            }
        });

    initOOBTimeout();
    this.storageLocationChecker = storageLocationChecker;
}
```
#### 1.3.2 启动 DN
```java
// 确定路径和资源启动DataNode
void startDataNode(List<StorageLocation> dataDirectories,
                   SecureResources resources
                  ) throws IOException {
    ...
    
    storage = new DataStorage();

    // global DN settings
    registerMXBean();
    // 初始化TcpPeerServer用于接收tcp请求, 实例化DataXceiverServer，用于接收客户端已经其他DataNode节点之间的数据服务，并设置成守护进程的方式在后台运行
    initDataXceiver();
    // 启动HttpServer
    startInfoServer();
    pauseMonitor = new JvmPauseMonitor();
    pauseMonitor.init(getConf());
    pauseMonitor.start();

    // BlockPoolTokenSecretManager is required to create ipc server.
    this.blockPoolTokenSecretManager = new BlockPoolTokenSecretManager();
	...
    // 初始化RPC的服务
    initIpcServer();
    ...
    // 创建了BlockPoolManager
    // BlockPool，一个集群就有一个BlockPool 
    blockPoolManager = new BlockPoolManager(this);
     // 周期性与NameNode通信，保持心跳，汇报情况
    blockPoolManager.refreshNamenodes(getConf());

    // Create the ReadaheadPool from the DataNode context so we can
    // exit without having to explicitly shutdown its thread pool.
    readaheadPool = ReadaheadPool.getInstance();
    saslClient = new SaslDataTransferClient(dnConf.getConf(),
                                            dnConf.saslPropsResolver, dnConf.trustedChannelResolver);
    saslServer = new SaslDataTransferServer(dnConf, blockPoolTokenSecretManager);
    startMetricsLogger();

    if (dnConf.diskStatsEnabled) {
        diskMetrics = new DataNodeDiskMetrics(this,
                                              dnConf.outliersReportIntervalMs, getConf());
    }
}
```
##### 1.3.2.1 启动 DataXceiver 服务，接收客户端和其他 DN 发送的数据
```java
private void initDataXceiver() throws IOException {
    // find free port or use privileged port provided
    TcpPeerServer tcpPeerServer;
    if (secureResources != null) {
        // 新建一个TcpPeerServer对象
        tcpPeerServer = new TcpPeerServer(secureResources);
    } else {...}
    ...
    // 通过TcpPeerServer对象创建DataXceiverServer的对象
    // DataXceiverServer用于接收或发送块数据，可以监听客户端或其他dataNode的请求
    xserver = new DataXceiverServer(tcpPeerServer, getConf(), this);
    this.dataXceiverServer = new Daemon(threadGroup, xserver);
    this.threadGroup.setDaemon(true); // 设置为守护进程
}


```
##### 1.3.2.2 启动 HttpServer
```java
private void startInfoServer()
    throws IOException {
    // SecureDataNodeStarter will bind the privileged port to the channel if
    // the DN is started by JSVC, pass it along.
    ServerSocketChannel httpServerChannel = secureResources != null ?
        secureResources.getHttpServerChannel() : null;
	// 新建DatanodeHttpServer并启动
    httpServer = new DatanodeHttpServer(getConf(), this, httpServerChannel);
    httpServer.start();
    if (httpServer.getHttpAddress() != null) {
        infoPort = httpServer.getHttpAddress().getPort();
    }
    if (httpServer.getHttpsAddress() != null) {
        infoSecurePort = httpServer.getHttpsAddress().getPort();
    }
}
```
##### 1.3.2.3 连接到 NN
```java
// BlockManager.java
void refreshNamenodes(Configuration conf)
    throws IOException {
    LOG.info("Refresh request received for nameservices: " +
             conf.get(DFSConfigKeys.DFS_NAMESERVICES));

    Map<String, Map<String, InetSocketAddress>> newAddressMap = null;
    Map<String, Map<String, InetSocketAddress>> newLifelineAddressMap = null;

    try {
        newAddressMap =
            DFSUtil.getNNServiceRpcAddressesForCluster(conf);
        newLifelineAddressMap =
            DFSUtil.getNNLifelineRpcAddressesForCluster(conf);
    } catch (IOException ioe) {
        LOG.warn("Unable to get NameNode addresses.", ioe);
    }

    if (newAddressMap == null || newAddressMap.isEmpty()) {
        throw new IOException("No services to connect, missing NameNode " +
                              "address.");
    }
    // new Object()
    synchronized (refreshNamenodesLock) {
        doRefreshNamenodes(newAddressMap, newLifelineAddressMap);
    }
}

// BlockPoolManager.java
private void doRefreshNamenodes(
    Map<String, Map<String, InetSocketAddress>> addrMap,
    Map<String, Map<String, InetSocketAddress>> lifelineAddrMap)
    throws IOException {
    assert Thread.holdsLock(refreshNamenodesLock);

    Set<String> toRefresh = Sets.newLinkedHashSet();
    Set<String> toAdd = Sets.newLinkedHashSet();
    Set<String> toRemove;

    synchronized (this) {
        // 循环遍历addrMap，一般只有一个nameservice，HA或联邦模式下会存在多个nameservice
        for (String nameserviceId : addrMap.keySet()) {
            if (bpByNameserviceId.containsKey(nameserviceId)) {
                // 加入待fresh的LinkedHashSet中
                toRefresh.add(nameserviceId);
            } else {
                toAdd.add(nameserviceId);
            }
        }

        // 其他模式下存在需要添加的nameservice
        if (!toAdd.isEmpty()) {
            LOG.info("Starting BPOfferServices for nameservices: " +
                     Joiner.on(",").useForNull("<default>").join(toAdd));
			//遍历所有的联邦集群，一个联邦里面会有两个NameNode(HA)
            //如果是2个联邦集群，那么这个地方就会有两个值
     	    //BPOfferService 对应 一个联邦集群
            for (String nsToAdd : toAdd) {
                Map<String, InetSocketAddress> nnIdToAddr = addrMap.get(nsToAdd);
                Map<String, InetSocketAddress> nnIdToLifelineAddr =
                    lifelineAddrMap.get(nsToAdd);
                ArrayList<InetSocketAddress> addrs =
                    Lists.newArrayListWithCapacity(nnIdToAddr.size());
                ArrayList<String> nnIds =
                    Lists.newArrayListWithCapacity(nnIdToAddr.size());
                ArrayList<InetSocketAddress> lifelineAddrs =
                    Lists.newArrayListWithCapacity(nnIdToAddr.size());
                for (String nnId : nnIdToAddr.keySet()) {
                    addrs.add(nnIdToAddr.get(nnId));
                    nnIds.add(nnId);
                    lifelineAddrs.add(nnIdToLifelineAddr != null ?
                                      nnIdToLifelineAddr.get(nnId) : null);
                }
                //一个联邦对应一个BPOfferService
       		    //一个联邦里面的一个NameNode就是一个BPServiceActor
        	    //也就是正常来说一个BPOfferService对应两个BPServiceActor
                BPOfferService bpos = createBPOS(nsToAdd, nnIds, addrs,
                                                 lifelineAddrs);
                bpByNameserviceId.put(nsToAdd, bpos);
                offerServices.add(bpos);
            }
        }
        startAll();
    }
}
```
###### 1.3.2.3.1 开启 BPOfferService
```java
synchronized void startAll() throws IOException {
    try {
        UserGroupInformation.getLoginUser().doAs(
            new PrivilegedExceptionAction<Object>() {
                @Override
                public Object run() throws Exception {
                     // 遍历所有的BPOfferService
                    for (BPOfferService bpos : offerServices) {
                        bpos.start();
                    }
                    return null;
                }
            });
    } catch (InterruptedException ex) {
        IOException ioe = new IOException();
        ioe.initCause(ex.getCause());
        throw ioe;
    }
}

void start() {
    for (BPServiceActor actor : bpServices) {
        // 一个BPOfferService里有多个actor
      actor.start();
    }
}

void start() {
    if ((bpThread != null) && (bpThread.isAlive())) {
        //Thread is started already
        return;
    }
    bpThread = new Thread(this);
    bpThread.setDaemon(true); // needed for JUnit testing

    if (lifelineSender != null) {
        lifelineSender.start();
    }
    // 调用run()
    bpThread.start();
}

public void run() {
    LOG.info(this + " starting to offer service");

    try {
        while (true) {
            // init stuff
            try {
                // 向NN注册
                connectToNNAndHandshake();
                break;
            } catch (IOException ioe) {
                // Initial handshake, storage recovery or registration failed
                runningState = RunningState.INIT_FAILED;
                if (shouldRetryInit()) {
                    // Retry until all namenode's of BPOS failed initialization
                    LOG.error("Initialization failed for " + this + " "
                              + ioe.getLocalizedMessage());
                    // 注册失败sleep5s
                    sleepAndLogInterrupts(5000, "initializing");
                } else {
                    runningState = RunningState.FAILED;
                    LOG.error("Initialization failed for " + this + ". Exiting. ", ioe);
                    return;
                }
            }
        }

        runningState = RunningState.RUNNING;
        if (initialRegistrationComplete != null) {
            initialRegistrationComplete.countDown();
        }

        while (shouldRun()) {
            try {
                // 注册结束发送心跳
                offerService();
            } catch (Exception ex) {
                LOG.error("Exception in BPOfferService for " + this, ex);
                sleepAndLogInterrupts(5000, "offering service");
            }
        }
        runningState = RunningState.EXITED;
    } catch (Throwable ex) {
        LOG.warn("Unexpected exception in block pool " + this, ex);
        runningState = RunningState.FAILED;
    } finally {
        LOG.warn("Ending block pool service for: " + this);
        cleanUp();
    }
}
```
###### 1.3.2.3.2 连接到 NN 进行注册
```java
private void connectToNNAndHandshake() throws IOException {
    // 通过DatanodeProtocolClientSideTranslatorPB对象获取NN的Proxy
    bpNamenode = dn.connectToNN(nnAddr);

    // 握手的第一阶段请求获取NN的信息
    NamespaceInfo nsInfo = retrieveNamespaceInfo();

    // Verify that this matches the other NN in this HA pair.
    // This also initializes our block pool in the DN if we are
    // the first NN connection for this BP.
    bpos.verifyAndSetNamespaceInfo(this, nsInfo);

    /* set thread name again to include NamespaceInfo when it's available. */
    this.bpThread.setName(formatThreadName("heartbeating", nnAddr));

    // 握手第二阶段-注册
    register(nsInfo);
}

void register(NamespaceInfo nsInfo) throws IOException {
    // The handshake() phase loaded the block pool storage
    // off disk - so update the bpRegistration object from that info
    DatanodeRegistration newBpRegistration = bpos.createRegistration();

    LOG.info(this + " beginning handshake with NN");

    while (shouldRun()) {
        try {
            // Use returned registration from namenode with updated fields
            newBpRegistration = bpNamenode.registerDatanode(newBpRegistration);
            // 注册完成
            newBpRegistration.setNamespaceInfo(nsInfo);
            bpRegistration = newBpRegistration;
            break;
        } catch(EOFException e) {...}

    // random short delay - helps scatter the BR from all DNs
    scheduler.scheduleBlockReport(dnConf.initialBlockReportDelayMs, true);
}
```
###### 1.3.2.3.3 发送心跳
```java
private void offerService() throws Exception {
    ...
    //
    // Now loop for a long time....
    //
    while (shouldRun()) {
        try {
            DataNodeFaultInjector.get().startOfferService();
            final long startTime = scheduler.monotonicNow();

            //
            // Every so often, send heartbeat or block-report
            //
            final boolean sendHeartbeat = scheduler.isHeartbeatDue(startTime);
            // 初始化HeartbeatResponse对象
            HeartbeatResponse resp = null;
            if (sendHeartbeat) {
                //
                // All heartbeat messages include following info:
                // -- Datanode name
                // -- data transfer port
                // -- Total capacity
                // -- Bytes remaining
                //
                boolean requestBlockReportLease = (fullBlockReportLeaseId == 0) &&
                    scheduler.isBlockReportDue(startTime);
                if (!dn.areHeartbeatsDisabledForTests()) {
                    // 发送心跳
                    resp = sendHeartBeat(requestBlockReportLease);
                    ...
}

HeartbeatResponse sendHeartBeat(boolean requestBlockReportLease)
    throws IOException {
    // 调度下次的心跳
    scheduler.scheduleNextHeartbeat();
    // DataNode存储信息
    StorageReport[] reports =
        dn.getFSDataset().getStorageReports(bpos.getBlockPoolId());
    if (LOG.isDebugEnabled()) {
        LOG.debug("Sending heartbeat with " + reports.length +
                  " storage reports from service actor: " + this);
    }

    final long now = monotonicNow();
    // 将上次更新心跳的时间更新成现在的时间，准备进行心跳
    scheduler.updateLastHeartbeatTime(now);
    ...
    // 获取NameNode代理对象bpNamenode，调用发送心跳的方法
    HeartbeatResponse response = bpNamenode.sendHeartbeat(bpRegistration,
        reports,
        dn.getFSDataset().getCacheCapacity(),
        dn.getFSDataset().getCacheUsed(),
        dn.getXmitsInProgress(),
        dn.getActiveTransferThreadCount(),
        numFailedVolumes,
        volumeFailureSummary,
        requestBlockReportLease,
        slowPeers,
        slowDisks);
    ...
    return response;
}

long scheduleNextHeartbeat() {
    // Numerical overflow is possible here and is okay.
    // 
    nextHeartbeatTime = monotonicNow() + heartbeatIntervalMs;
    scheduleNextLifeline(nextHeartbeatTime);
    return nextHeartbeatTime;
}

// bpNamenode获取的是NameNode的RPC代理对象, DatanodeProtocolClientSideTranslatorPB,对应的类实际上是NameNodeRpcServer
public HeartbeatResponse sendHeartbeat(DatanodeRegistration nodeReg,
      StorageReport[] report, long dnCacheCapacity, long dnCacheUsed,
      int xmitsInProgress, int xceiverCount,
      int failedVolumes, VolumeFailureSummary volumeFailureSummary,
      boolean requestFullBlockReportLease,
      @Nonnull SlowPeerReports slowPeers,
      @Nonnull SlowDiskReports slowDisks)
          throws IOException {
    // 检查NameNode是否启动
    checkNNStartup();
    verifyRequest(nodeReg);
    // 处理心跳
    return namesystem.handleHeartbeat(nodeReg, report,
        dnCacheCapacity, dnCacheUsed, xceiverCount, xmitsInProgress,
        failedVolumes, volumeFailureSummary, requestFullBlockReportLease,
        slowPeers, slowDisks);
  }
```
##### 1.3.2.4 NN 收到心跳，进行处理
```java
// FSNamesystem.java
HeartbeatResponse handleHeartbeat(DatanodeRegistration nodeReg,
      StorageReport[] reports, long cacheCapacity, long cacheUsed,
      int xceiverCount, int xmitsInProgress, int failedVolumes,
      VolumeFailureSummary volumeFailureSummary,
      boolean requestFullBlockReportLease,
      @Nonnull SlowPeerReports slowPeers,
      @Nonnull SlowDiskReports slowDisks)
          throws IOException {
    readLock();
    try {
      //get datanode commands
      final int maxTransfer = blockManager.getMaxReplicationStreams()
          - xmitsInProgress;
      // 从FSNamesystem类中获取blockManager对象，通过DatanodeManager对DataNode发送的心跳信息进行处理，最后返回指令cmds
      DatanodeCommand[] cmds = blockManager.getDatanodeManager().handleHeartbeat(
          nodeReg, reports, getBlockPoolId(), cacheCapacity, cacheUsed,
          xceiverCount, maxTransfer, failedVolumes, volumeFailureSummary,
          slowPeers, slowDisks);

      return new HeartbeatResponse(cmds, haState, rollingUpgradeInfo,
          blockReportLeaseId);
    } finally {
      readUnlock("handleHeartbeat");
    }
  }

// DatanodeManager.java
public DatanodeCommand[] handleHeartbeat(DatanodeRegistration nodeReg,
      StorageReport[] reports, final String blockPoolId,
      long cacheCapacity, long cacheUsed, int xceiverCount, 
      int maxTransfers, int failedVolumes,
      VolumeFailureSummary volumeFailureSummary,
      @Nonnull SlowPeerReports slowPeers,
      @Nonnull SlowDiskReports slowDisks) throws IOException {
    final DatanodeDescriptor nodeinfo;
    try {
      // 从DatanodeRegistration中获取Datanode信息
      nodeinfo = getDatanode(nodeReg);
    } catch (UnregisteredNodeException e) {
      return new DatanodeCommand[]{RegisterCommand.REGISTER};
    }
    ...
    // 更新心跳
    heartbeatManager.updateHeartbeat(nodeinfo, reports, cacheCapacity,
        cacheUsed, xceiverCount, failedVolumes, volumeFailureSummary);
    ...
    return new DatanodeCommand[0];
  }

synchronized void updateHeartbeat(final DatanodeDescriptor node,
      StorageReport[] reports, long cacheCapacity, long cacheUsed,
      int xceiverCount, int failedVolumes,
      VolumeFailureSummary volumeFailureSummary) {
    stats.subtract(node);
    blockManager.updateHeartbeat(node, reports, cacheCapacity, 	cacheUsed,
        xceiverCount, failedVolumes, volumeFailureSummary);
    stats.add(node);
}

void updateHeartbeat(DatanodeDescriptor node, StorageReport[] reports,
      long cacheCapacity, long cacheUsed, int xceiverCount, int failedVolumes,
      VolumeFailureSummary volumeFailureSummary) {

    for (StorageReport report: reports) {
      providedStorageMap.updateStorage(node, report.getStorage());
    }
    node.updateHeartbeat(reports, cacheCapacity, cacheUsed, xceiverCount,
        failedVolumes, volumeFailureSummary);
  }
// DatanodeDescriptor.java
void updateHeartbeat(StorageReport[] reports, long cacheCapacity,
      long cacheUsed, int xceiverCount, int volFailures,
      VolumeFailureSummary volumeFailureSummary) {
    updateHeartbeatState(reports, cacheCapacity, cacheUsed, xceiverCount,
        volFailures, volumeFailureSummary);
    heartbeatedSinceRegistration = true;
  }

void updateHeartbeatState(StorageReport[] reports, long cacheCapacity,
      long cacheUsed, int xceiverCount, int volFailures,
      VolumeFailureSummary volumeFailureSummary) {
    // 更新存储状态
    updateStorageStats(reports, cacheCapacity, cacheUsed, xceiverCount,
        volFailures, volumeFailureSummary);
    // 更新上次更新时间
    setLastUpdate(Time.now());
    setLastUpdateMonotonic(Time.monotonicNow());
    rollBlocksScheduled(getLastUpdateMonotonic());
  }
```

## 1.2 写流程

### 1.2.1 创建 DataStreamer，用于后续传输数据
#### 1.2.1.1 FileSystem.create()
```java
// 客户端发起写请求，调用create方法
public FSDataOutputStream create(Path f) throws IOException {
    return create(f, true);
}

public FSDataOutputStream create(Path f, boolean overwrite)
    throws IOException {
    return create(f, overwrite,
                  getConf().getInt(IO_FILE_BUFFER_SIZE_KEY,
                                   IO_FILE_BUFFER_SIZE_DEFAULT),
                  getDefaultReplication(f),
                  getDefaultBlockSize(f));
}

public FSDataOutputStream create(Path f,
                                 boolean overwrite,
                                 int bufferSize,
                                 short replication,
                                 long blockSize) throws IOException {
    return create(f, overwrite, bufferSize, replication, blockSize, null);
}

public FSDataOutputStream create(Path f,
                                 boolean overwrite,
                                 int bufferSize,
                                 short replication,
                                 long blockSize,
                                 Progressable progress
                                ) throws IOException {
    return this.create(f, FsCreateModes.applyUMask(
        FsPermission.getFileDefault(), FsPermission.getUMask(getConf())),
                       overwrite, bufferSize, replication, blockSize, progress);
}

// 抽象方法通过DistributedFileSystem来实现
public abstract FSDataOutputStream create(Path f,
                                          FsPermission permission,
                                          boolean overwrite,
                                          int bufferSize,
                                          short replication,
                                          long blockSize,
                                          Progressable progress) throws IOException;
```
#### 1.2.1.2 DistributedFileSystem.create()
```java
// DistributedFileSystem.java
@Override
public FSDataOutputStream create(Path f, FsPermission permission,
      boolean overwrite, int bufferSize, short replication, long blockSize,
      Progressable progress) throws IOException {
    return this.create(f, permission,
                       overwrite ? EnumSet.of(CreateFlag.CREATE, CreateFlag.OVERWRITE)
                       : EnumSet.of(CreateFlag.CREATE), bufferSize, replication,
                       blockSize, progress, null);
    
@Override
public FSDataOutputStream create(
  final Path f, final FsPermission permission,
  final EnumSet<CreateFlag> cflags, final int bufferSize,
  final short replication, final long blockSize,
  final Progressable progress, final ChecksumOpt checksumOpt)
  throws IOException {
    statistics.incrementWriteOps(1);
    storageStatistics.incrementOpCounter(OpType.CREATE);
    Path absF = fixRelativePart(f);
    // FileSystemLinkResolver是一个抽象类，定义了doCall方法和next方法
    // public T doCall(final Path p)
    // Params: p – Path on which to perform an operation 
    // Returns: Generic type returned by operation
    
    // public T next(final FileSystem fs, final Path p)
    // Params: fs – FileSystem with which to retry call, 
    // p – Resolved Target of path
	// Returns: Generic type determined by implementation
    return new FileSystemLinkResolver<FSDataOutputStream>() {
      @Override
      public FSDataOutputStream doCall(final Path p) throws IOException {
        // 调用完此处的create方法后，返回的DFSOutputStream对象传给safelyCreateWrappedOutputStream方法进行封装
        // 通过DFSClient创建DFSOutputStream
        final DFSOutputStream dfsos = dfs.create(getPathName(p), permission,
            cflags, replication, blockSize, progress, bufferSize,
            checksumOpt);
        return safelyCreateWrappedOutputStream(dfsos);
      }
      @Override
      public FSDataOutputStream next(final FileSystem fs, final Path p)
          throws IOException {
        return fs.create(p, permission, cflags, bufferSize,
            replication, blockSize, progress, checksumOpt);
      }
}.resolve(this, absF);
```
#### 1.2.1.3 DFSClient.create()
```java
// 向文件树添加INodeFile，添加契约管理，启动DataStreamer，与NNRPCServer通信
public DFSOutputStream create(String src, FsPermission permission,
      EnumSet<CreateFlag> flag, short replication, long blockSize,
      Progressable progress, int buffersize, ChecksumOpt checksumOpt)
      throws IOException {
    return create(src, permission, flag, true,
        replication, blockSize, progress, buffersize, checksumOpt, null);
}

public DFSOutputStream create(String src, FsPermission permission,
      EnumSet<CreateFlag> flag, boolean createParent, short replication,
      long blockSize, Progressable progress, int buffersize,
      ChecksumOpt checksumOpt, InetSocketAddress[] favoredNodes)
      throws IOException {
    return create(src, permission, flag, createParent, replication, blockSize,
        progress, buffersize, checksumOpt, favoredNodes, null);
}

public DFSOutputStream create(String src, FsPermission permission,
      EnumSet<CreateFlag> flag, boolean createParent, short replication,
      long blockSize, Progressable progress, int buffersize,
      ChecksumOpt checksumOpt, InetSocketAddress[] favoredNodes,
      String ecPolicyName) throws IOException {
    return create(src, permission, flag, createParent, replication, blockSize,
        progress, buffersize, checksumOpt, favoredNodes, ecPolicyName, null);
}

public DFSOutputStream create(String src, FsPermission permission,
      EnumSet<CreateFlag> flag, boolean createParent, short replication,
      long blockSize, Progressable progress, int buffersize,
      ChecksumOpt checksumOpt, InetSocketAddress[] favoredNodes,
      String ecPolicyName, String storagePolicy)
      throws IOException {
    checkOpen();
    final FsPermission masked = applyUMask(permission);
    LOG.debug("{}: masked={}", src, masked);
    
    final DFSOutputStream result = DFSOutputStream.newStreamForCreate(this,
        src, masked, flag, createParent, replication, blockSize, progress,
        dfsClientConf.createChecksum(checksumOpt),
        getFavoredNodesStr(favoredNodes), ecPolicyName, storagePolicy);
    beginFileLease(result.getFileId(), result);
    return result;
}

// DFSOutputStream.java
static DFSOutputStream newStreamForCreate(DFSClient dfsClient, String src,
      FsPermission masked, EnumSet<CreateFlag> flag, boolean createParent,
      short replication, long blockSize, Progressable progress,
      DataChecksum checksum, String[] favoredNodes, String ecPolicyName,
      String storagePolicy)
      throws IOException {
    try (TraceScope ignored =
             dfsClient.newPathTraceScope("newStreamForCreate", src)) {
      HdfsFileStatus stat = null;

      // Retry the create if we get a RetryStartFileException up to a maximum
      // number of times
      boolean shouldRetry = true;
      int retryCount = CREATE_RETRY_COUNT;
        // 不断重试确保创建成功
      while (shouldRetry) {
        shouldRetry = false;
        try {
          // NameNodeRPC.create()
          stat = dfsClient.namenode.create(src, masked, dfsClient.clientName,
              new EnumSetWritable<>(flag), createParent, replication,
              blockSize, SUPPORTED_CRYPTO_VERSIONS, ecPolicyName,
              storagePolicy);
          break; // 成功break否则抛异常并重试
        } catch (RemoteException re) {
          ...
      }
      Preconditions.checkNotNull(stat, "HdfsFileStatus should not be null!");
      final DFSOutputStream out;
      if(stat.getErasureCodingPolicy() != null) {
        out = new DFSStripedOutputStream(dfsClient, src, stat,
            flag, progress, checksum, favoredNodes);
      } else {
        // 创建一个DataStreamer
        out = new DFSOutputStream(dfsClient, src, stat,
            flag, progress, checksum, favoredNodes, true);
      }
      // 启动
      // start方法获取到DataStreamer并启动
      // DataStreamer启动后还没有Packet进行传输，会阻塞
      out.start();
      return out;
    }
  }
```
#### 1.2.1.4 new DFSOutputStream()
```java
protected DFSOutputStream(DFSClient dfsClient, String src,
    HdfsFileStatus stat, EnumSet<CreateFlag> flag, Progressable progress,
    DataChecksum checksum, String[] favoredNodes, boolean createStreamer) {
  this(dfsClient, src, flag, progress, stat, checksum);
  this.shouldSyncBlock = flag.contains(CreateFlag.SYNC_BLOCK);
  // 计算数据单元的值
  // block(128M) packet(64k) chunk(512byte) chunksum(4byte)
  computePacketChunkSize(dfsClient.getConf().getWritePacketSize(),
      bytesPerChecksum);

  if (createStreamer) {
    // 新建DataStreamer对象
    streamer = new DataStreamer(stat, null, dfsClient, src, progress,
        checksum, cachingStrategy, byteArrayManager, favoredNodes,
        addBlockFlags);
  }
}
```

### 1.2.2 开启 Rpc 服务，与 NN 交互
#### 1.2.2.1 开启 Rpc 服务，向 NN 申请创建文件
```java
public HdfsFileStatus create(String src, FsPermission masked,
      String clientName, EnumSetWritable<CreateFlag> flag,
      boolean createParent, short replication, long blockSize,
      CryptoProtocolVersion[] supportedVersions, String ecPolicyName,
      String storagePolicy)
      throws IOException {
    // 检查NN的状态
    checkNNStartup();
    try {
      PermissionStatus perm = new PermissionStatus(getRemoteUser()
          .getShortUserName(), null, masked);
      // 在namespace中创建一个file
      status = namesystem.startFile(src, perm, clientName, clientMachine,
          flag.get(), createParent, replication, blockSize, supportedVersions,
          ecPolicyName, storagePolicy, cacheEntry != null);
    } finally {
      RetryCache.setState(cacheEntry, status != null, status);
    }
    metrics.incrFilesCreated();
    metrics.incrCreateFileOps();
    return status;
}

HdfsFileStatus startFile(String src, PermissionStatus permissions,
      String holder, String clientMachine, EnumSet<CreateFlag> flag,
      boolean createParent, short replication, long blockSize,
      CryptoProtocolVersion[] supportedVersions, String ecPolicyName,
      String storagePolicy, boolean logRetryCache) throws IOException {

    HdfsFileStatus status;
    try {
      status = startFileInt(src, permissions, holder, clientMachine, flag,
          createParent, replication, blockSize, supportedVersions, ecPolicyName,
          storagePolicy, logRetryCache);
    }
    return status;
  }

private HdfsFileStatus startFileInt(String src,
      PermissionStatus permissions, String holder, String clientMachine,
      EnumSet<CreateFlag> flag, boolean createParent, short replication,
      long blockSize, CryptoProtocolVersion[] supportedVersions,
      String ecPolicyName, String storagePolicy, boolean logRetryCache)
      throws IOException {

      try {
        stat = FSDirWriteFileOp.startFile(this, iip, permissions, holder, clientMachine, flag, createParent, replication, blockSize, feInfo, toRemoveBlocks, shouldReplicate, ecPolicyName, storagePolicy,
            logRetryCache);
      }
    return stat;
}
```
#### 1.2.2.2 NN 检查目录树
```java
// FSDirWriteFileOp.java
static HdfsFileStatus startFile(
      FSNamesystem fsn, INodesInPath iip,
      PermissionStatus permissions, String holder, String clientMachine,
      EnumSet<CreateFlag> flag, boolean createParent,
      short replication, long blockSize,
      FileEncryptionInfo feInfo, INode.BlocksMapUpdateInfo toRemoveBlocks,
      boolean shouldReplicate, String ecPolicyName, String storagePolicy,
      boolean logRetryEntry)
      throws IOException {
    assert fsn.hasWriteLock();
    // 之前create方法中设置的是否覆盖
    boolean overwrite = flag.contains(CreateFlag.OVERWRITE);
    boolean isLazyPersist = flag.contains(CreateFlag.LAZY_PERSIST);

    final String src = iip.getPath();
    FSDirectory fsd = fsn.getFSDirectory();
	
    // inode已存在
    if (iip.getLastINode() != null) {
      if (overwrite) {
        // 需要覆盖：删除替换
        List<INode> toRemoveINodes = new ChunkedArrayList<>();
        List<Long> toRemoveUCFiles = new ChunkedArrayList<>();
        long ret = FSDirDeleteOp.delete(fsd, iip, toRemoveBlocks,
                                        toRemoveINodes, toRemoveUCFiles, now());
        if (ret >= 0) {
          iip = INodesInPath.replace(iip, iip.length() - 1, null);
          FSDirDeleteOp.incrDeletedFileCount(ret);
          fsn.removeLeasesAndINodes(toRemoveUCFiles, toRemoveINodes, true);
        }
      } else {
        // 否则抛异常
        // If lease soft limit time is expired, recover the lease
        fsn.recoverLeaseInternal(FSNamesystem.RecoverLeaseOp.CREATE_FILE, iip,
                                 src, holder, clientMachine, false);
        throw new FileAlreadyExistsException(src + " for client " +
            clientMachine + " already exists");
      }
    }
    fsn.checkFsObjectLimit();
    INodeFile newNode = null;
    // 创建父目录
    INodesInPath parent =
        FSDirMkdirOp.createAncestorDirectories(fsd, iip, permissions);
    if (parent != null) {
      // 添加至目录树中
      iip = addFile(fsd, parent, iip.getLastLocalName(), permissions,
          replication, blockSize, holder, clientMachine, shouldReplicate,
          ecPolicyName, storagePolicy);
      newNode = iip != null ? iip.getLastINode().asFile() : null;
    }
    if (newNode == null) {
      throw new IOException("Unable to add " + src +  " to namespace");
    }
    // 添加契约
    fsn.leaseManager.addLease(
        newNode.getFileUnderConstructionFeature().getClientName(),
        newNode.getId());
    if (feInfo != null) {
      FSDirEncryptionZoneOp.setFileEncryptionInfo(fsd, iip, feInfo,
          XAttrSetFlag.CREATE);
    }
    setNewINodeStoragePolicy(fsd.getBlockManager(), iip, isLazyPersist);
    fsd.getEditLog().logOpenFile(src, newNode, overwrite, logRetryEntry);
    if (NameNode.stateChangeLog.isDebugEnabled()) {
      NameNode.stateChangeLog.debug("DIR* NameSystem.startFile: added " +
          src + " inode " + newNode.getId() + " " + holder);
    }
    return FSDirStatAndListingOp.getFileInfo(fsd, iip, false, false);
  }

private static INodesInPath addFile(
      FSDirectory fsd, INodesInPath existing, byte[] localName,
      PermissionStatus permissions, short replication, long preferredBlockSize,
      String clientName, String clientMachine, boolean shouldReplicate,
      String ecPolicyName, String storagePolicy) throws IOException {

      ...
      // 创建INode
      INodeFile newNode = newINodeFile(fsd.allocateNewInodeId(), permissions,
          modTime, modTime, replicationFactor, ecPolicyID, preferredBlockSize,
          storagepolicyid, blockType);
      newNode.setLocalName(localName);
      newNode.toUnderConstruction(clientName, clientMachine);
      // 将INode添加到namespace中
      newiip = fsd.addINode(existing, newNode, permissions.getPermission());
    } finally {
      fsd.writeUnlock();
    }
    if (newiip == null) {
      NameNode.stateChangeLog.info("DIR* addFile: failed to add " +
          existing.getPath() + "/" + DFSUtil.bytes2String(localName));
      return null;
    }

    if(NameNode.stateChangeLog.isDebugEnabled()) {
      NameNode.stateChangeLog.debug("DIR* addFile: " +
          DFSUtil.bytes2String(localName) + " is added");
    }
    return newiip;
  }

INodesInPath addINode(INodesInPath existing, INode child,
                        FsPermission modes)
      throws QuotaExceededException, UnresolvedLinkException {
    cacheName(child);
    writeLock();
    try {
      return addLastINode(existing, child, modes, true);
    } finally {
      writeUnlock();
    }
  }
```
#### 1.2.2.3 NN 返回可用的块
```java
// DataStreamer的run方法:
public void run() {
    TraceScope scope = null;
    while (!streamerClosed && dfsClient.clientRunning) {
      // if the Responder encountered an error, shutdown Responder
      if (errorState.hasError()) {
        closeResponder();
      }
      
      DFSPacket one;
      try {
        // process datanode IO errors if any
        boolean doSleep = processDatanodeOrExternalError();

        synchronized (dataQueue) {
          // 等待传送packet
          while ((!shouldStop() && dataQueue.isEmpty()) || doSleep) {
            long timeout = 1000;
            try {
              dataQueue.wait(timeout);
            }
            doSleep = false;
          }
          // 产生了packet后，进行下面的逻辑
          // get packet to be sent.
          one = dataQueue.getFirst(); // regular data packet
          
		// 建立管道向NN申请block
        if (stage == BlockConstructionStage.PIPELINE_SETUP_CREATE) {
          LOG.debug("Allocating new block: {}", this);
          // nextBlockOutputStream向NN申请block用于写入数据，以及选择存放block的DN的策略
          setPipeline(nextBlockOutputStream());
          // 初始化DataStreaming服务，监听packet发送的状态
          initDataStreaming();
        }

        if (one.isLastPacketInBlock()) {
          // wait for all data packets have been successfully acked
          waitForAllAcks();
          if(shouldStop()) {
            continue;
          }
          stage = BlockConstructionStage.PIPELINE_CLOSE;
        }

        // send the packet
        SpanContext spanContext = null;
        synchronized (dataQueue) {
          // move packet from dataQueue to ackQueue
          if (!one.isHeartbeatPacket()) {
            if (scope != null) {
              one.setSpan(scope.span());
              spanContext = scope.span().getContext();
              scope.close();
            }
            scope = null;
            // 发送完的packet加入ackQueue
            dataQueue.removeFirst();
            ackQueue.addLast(one);
            packetSendTime.put(one.getSeqno(), Time.monotonicNow());
            // 唤醒wait线程
            dataQueue.notifyAll();
          }
        }

        LOG.debug("{} sending {}", this, one);

        // write out data to remote datanode
        try (TraceScope ignored = dfsClient.getTracer().
            newScope("DataStreamer#writeTo", spanContext)) {
          // 发送packet，向DN写数据
          sendPacket(one);
        } 
    closeInternal();
  }}

protected LocatedBlock nextBlockOutputStream() throws IOException {
    LocatedBlock lb;
    DatanodeInfo[] nodes;
    StorageType[] nextStorageTypes;
    String[] nextStorageIDs;
    int count = dfsClient.getConf().getNumBlockWriteRetry();
    boolean success;
    final ExtendedBlock oldBlock = block.getCurrentBlock();
    do {
      errorState.resetInternalError();
      lastException.clear();

      DatanodeInfo[] excluded = getExcludedNodes();
      // 定位可用的块
      lb = locateFollowingBlock(
          excluded.length > 0 ? excluded : null, oldBlock);
      block.setCurrentBlock(lb.getBlock());
      block.setNumBytes(0);
      bytesSent = 0;
      accessToken = lb.getBlockToken();
      nodes = lb.getLocations();
      nextStorageTypes = lb.getStorageTypes();
      nextStorageIDs = lb.getStorageIDs();

      // Connect to first DataNode in the list.
      // 连接DN
      success = createBlockOutputStream(nodes, nextStorageTypes, nextStorageIDs,
          0L, false);

      if (!success) {
        LOG.warn("Abandoning " + block);
        dfsClient.namenode.abandonBlock(block.getCurrentBlock(),
            stat.getFileId(), src, dfsClient.clientName);
        block.setCurrentBlock(null);
        final DatanodeInfo badNode = nodes[errorState.getBadNodeIndex()];
        LOG.warn("Excluding datanode " + badNode);
        excludedNodes.put(badNode, badNode);
      }
    } while (!success && --count >= 0);

    if (!success) {
      throw new IOException("Unable to create new block.");
    }
    return lb;
  }
```
##### 1.2.2.3.1 locateFollowingBlock()
```java
private LocatedBlock locateFollowingBlock(DatanodeInfo[] excluded,
      ExtendedBlock oldBlock) throws IOException {
    return DFSOutputStream.addBlock(excluded, dfsClient, src, oldBlock,
        stat.getFileId(), favoredNodes, addBlockFlags);
}

static LocatedBlock addBlock(DatanodeInfo[] excludedNodes,
      DFSClient dfsClient, String src, ExtendedBlock prevBlock, long fileId,
      String[] favoredNodes, EnumSet<AddBlockFlag> allocFlags)
      throws IOException {
    ...
    while (true) {
      try {
        // 和NN通信
        return dfsClient.namenode.addBlock(src, dfsClient.clientName, prevBlock,
            excludedNodes, fileId, favoredNodes, allocFlags);
      } catch (RemoteException e) {...}
  
```
##### 1.2.2.3.2 addBlock()
```java
// NameNodeRpcServer.java
public LocatedBlock addBlock(String src, String clientName,
    ExtendedBlock previous, DatanodeInfo[] excludedNodes, long fileId,
    String[] favoredNodes, EnumSet<AddBlockFlag> addBlockFlags)
    throws IOException {
  checkNNStartup();
  LocatedBlock locatedBlock = namesystem.getAdditionalBlock(src, fileId,
      clientName, previous, excludedNodes, favoredNodes, addBlockFlags);
  return locatedBlock;
}

LocatedBlock getAdditionalBlock(
      String src, long fileId, String clientName, ExtendedBlock previous,
      DatanodeInfo[] excludedNodes, String[] favoredNodes,
      EnumSet<AddBlockFlag> flags) throws IOException {
    // 选择块
    DatanodeStorageInfo[] targets = FSDirWriteFileOp.chooseTargetForNewBlock(
        blockManager, src, excludedNodes, favoredNodes, flags, r);
    return lb;
}

static DatanodeStorageInfo[] chooseTargetForNewBlock(
      BlockManager bm, String src, DatanodeInfo[] excludedNodes,
      String[] favoredNodes, EnumSet<AddBlockFlag> flags,
      ValidateAddBlockResult r) throws IOException {
	...
    // choose targets for the new block to be allocated.
    return bm.chooseTarget4NewBlock(src, r.numTargets, clientNode,
                                    excludedNodesSet, r.blockSize,
                                    favoredNodesList, r.storagePolicyID,
                                    r.blockType, r.ecPolicy, flags);
}

public DatanodeStorageInfo[] chooseTarget4NewBlock(final String src,
      final int numOfReplicas, final Node client,
      final Set<Node> excludedNodes,
      final long blocksize,
      final List<String> favoredNodes,
      final byte storagePolicyID,
      final BlockType blockType,
      final ErasureCodingPolicy ecPolicy,
      final EnumSet<AddBlockFlag> flags) throws IOException {
    
    final DatanodeStorageInfo[] targets = blockplacement.chooseTarget(src,
        numOfReplicas, client, excludedNodes, blocksize, 
        favoredDatanodeDescriptors, storagePolicy, flags);
    return targets;
}
```
##### 1.2.2.3.3 chooseTarget()
```java
// BlockPlacementPolicy.java
DatanodeStorageInfo[] chooseTarget(String src,
      int numOfReplicas, Node writer,
      Set<Node> excludedNodes,
      long blocksize,
      List<DatanodeDescriptor> favoredNodes,
      BlockStoragePolicy storagePolicy,
      EnumSet<AddBlockFlag> flags) {
    // This class does not provide the functionality of placing
    // a block in favored datanodes. The implementations of this class
    // are expected to provide this functionality

    return chooseTarget(src, numOfReplicas, writer, 
        new ArrayList<DatanodeStorageInfo>(numOfReplicas), false,
        excludedNodes, blocksize, storagePolicy, flags);
}

// BlockPlacementPolicyDefault.java
public DatanodeStorageInfo[] chooseTarget(String srcPath,
                                    int numOfReplicas,
                                    Node writer,
                                    List<DatanodeStorageInfo> chosenNodes,
                                    boolean returnChosenNodes,
                                    Set<Node> excludedNodes,
                                    long blocksize,
                                    final BlockStoragePolicy storagePolicy,
                                    EnumSet<AddBlockFlag> flags) {
    return chooseTarget(numOfReplicas, writer, chosenNodes, returnChosenNodes,
        excludedNodes, blocksize, storagePolicy, flags, null);
}

private DatanodeStorageInfo[] chooseTarget(int numOfReplicas,
                                    Node writer,
                                    List<DatanodeStorageInfo> chosenStorage,
                                    boolean returnChosenNodes,
                                    Set<Node> excludedNodes,
                                    long blocksize,
                                    final BlockStoragePolicy storagePolicy,
                                    EnumSet<AddBlockFlag> addBlockFlags,
                                    EnumMap<StorageType, Integer> sTypes) {
    ...
    if (avoidLocalNode && results == null) {
      results = new ArrayList<>(chosenStorage);
      Set<Node> excludedNodeCopy = new HashSet<>(excludedNodes);
      if (writer != null) {
        excludedNodeCopy.add(writer);
      }
      // 选择本地节点
      localNode = chooseTarget(numOfReplicas, writer,
          excludedNodeCopy, blocksize, maxNodesPerRack, results,
          avoidStaleNodes, storagePolicy,
          EnumSet.noneOf(StorageType.class), results.isEmpty(), sTypes);
}

private Node chooseTarget(final int numOfReplicas,
                            Node writer,
                            final Set<Node> excludedNodes,
                            final long blocksize,
                            final int maxNodesPerRack,
                            final List<DatanodeStorageInfo> results,
                            final boolean avoidStaleNodes,
                            final BlockStoragePolicy storagePolicy,
                            final EnumSet<StorageType> unavailableStorages,
                            final boolean newBlock,
                            EnumMap<StorageType, Integer> storageTypes) {
    ...
    try {
      // 机架感知
      writer = chooseTargetInOrder(numOfReplicas, writer, excludedNodes, blocksize,
          maxNodesPerRack, results, avoidStaleNodes, newBlock, storageTypes);
    } 
    
    return writer;
}

protected Node chooseTargetInOrder(int numOfReplicas, 
                                 Node writer,
                                 final Set<Node> excludedNodes,
                                 final long blocksize,
                                 final int maxNodesPerRack,
                                 final List<DatanodeStorageInfo> results,
                                 final boolean avoidStaleNodes,
                                 final boolean newBlock,
                                 EnumMap<StorageType, Integer> storageTypes)
                                 throws NotEnoughReplicasException {
    final int numOfResults = results.size();
    if (numOfResults == 0) {
      // 第一块选择本地机架  
      DatanodeStorageInfo storageInfo = chooseLocalStorage(writer,
          excludedNodes, blocksize, maxNodesPerRack, results, avoidStaleNodes,
          storageTypes, true);

      writer = (storageInfo != null) ? storageInfo.getDatanodeDescriptor()
                                     : null;

      if (--numOfReplicas == 0) {
        return writer;
      }
    }
    final DatanodeDescriptor dn0 = results.get(0).getDatanodeDescriptor();
    // 第二块选择与dn0不同的节点
    if (numOfResults <= 1) {
      chooseRemoteRack(1, dn0, excludedNodes, blocksize, maxNodesPerRack,
          results, avoidStaleNodes, storageTypes);
      if (--numOfReplicas == 0) {
        return writer;
      }
    }
    // 第三块
    if (numOfResults <= 2) {
      final DatanodeDescriptor dn1 = results.get(1).getDatanodeDescriptor();
      // 如果dn0、dn1在同一个机架上，则选择与dn0不同的机架
      if (clusterMap.isOnSameRack(dn0, dn1)) {
        chooseRemoteRack(1, dn0, excludedNodes, blocksize, maxNodesPerRack,
            results, avoidStaleNodes, storageTypes);
      } else if (newBlock){
        // 否则选择和dn1不同的机架
        chooseLocalRack(dn1, excludedNodes, blocksize, maxNodesPerRack,
            results, avoidStaleNodes, storageTypes);
      } else {
        chooseLocalRack(writer, excludedNodes, blocksize, maxNodesPerRack,
            results, avoidStaleNodes, storageTypes);
      }
      if (--numOfReplicas == 0) {
        return writer;
      }
    }
    chooseRandom(numOfReplicas, NodeBase.ROOT, excludedNodes, blocksize,
        maxNodesPerRack, results, avoidStaleNodes, storageTypes);
    return writer;
}
```
### 1.2.3 与 DN 建立连接，发送数据
#### 1.2.3.1 与 DN 建立连接
```java
boolean createBlockOutputStream(DatanodeInfo[] nodes,
      StorageType[] nodeStorageTypes, String[] nodeStorageIDs,
      long newGS, boolean recoveryFlag) {
    int refetchEncryptionKey = 1;
    while (true) {
      boolean result = false;
      DataOutputStream out = null;
      try {
		// 获取socket的输入/出流用于后续发送packet
        OutputStream unbufOut = NetUtils.getOutputStream(s, writeTimeout);
        InputStream unbufIn = NetUtils.getInputStream(s, readTimeout);
        IOStreamPair saslStreams = dfsClient.saslClient.socketSend(s,
            unbufOut, unbufIn, dfsClient, accessToken, nodes[0]);
        unbufOut = saslStreams.out;
        unbufIn = saslStreams.in;
        out = new DataOutputStream(new BufferedOutputStream(unbufOut,
            DFSUtilClient.getSmallBufferSize(dfsClient.getConfiguration())));
        blockReplyStream = new DataInputStream(unbufIn);

        // 发送写入块的请求
        new Sender(out).writeBlock(...);

        // receive ack for connect

      return result;
    }
  }

// Sender.writeBlock()
public void writeBlock(final ExtendedBlock blk,
      final StorageType storageType,
      final Token<BlockTokenIdentifier> blockToken,
      final String clientName,
      final DatanodeInfo[] targets,
      final StorageType[] targetStorageTypes,
      final DatanodeInfo source,
      final BlockConstructionStage stage,
      final int pipelineSize,
      final long minBytesRcvd,
      final long maxBytesRcvd,
      final long latestGenerationStamp,
      DataChecksum requestedChecksum,
      final CachingStrategy cachingStrategy,
      final boolean allowLazyPersist,
      final boolean pinning,
      final boolean[] targetPinnings,
      final String storageId,
      final String[] targetStorageIds) throws IOException {
      
    send(out, Op.WRITE_BLOCK, proto.build());
}

private static void send(final DataOutputStream out, final Op opcode,
      final Message proto) throws IOException {
    LOG.trace("Sending DataTransferOp {}: {}",
        proto.getClass().getSimpleName(), proto);
    op(out, opcode);
    proto.writeDelimitedTo(out);
    out.flush();
}
```
#### 1.2.3.2 initDataStreaming()
```java
// 返回到DataStreamer的run方法中
private void initDataStreaming() {
    // 创建并启动ResponseProcessor线程
    response = new ResponseProcessor(nodes);
    // ResponseProcessor是DataStreamer的私有内部类
    response.start();
    stage = BlockConstructionStage.DATA_STREAMING;
    lastPacket = Time.monotonicNow();
}

@Override
public void run() {
    setName("ResponseProcessor for block " + block);
    // 返回DN写数据结果
    PipelineAck ack = new PipelineAck();

    TraceScope scope = null;
    while (!responderClosed && dfsClient.clientRunning && !isLastPacketInBlock) {
        // process responses from datanodes.
        try {
            // read an ack from the pipeline
            // 读取下游处理结果
            ack.readFields(blockReplyStream);

            synchronized (dataQueue) {
                lastAckedSeqno = seqno;
                pipelineRecoveryCount = 0;
                // 发送成功从ack队列移除
                ackQueue.removeFirst();
    
                packetSendTime.remove(seqno);
                dataQueue.notifyAll();

                one.releaseBuffer(byteArrayManager);
            }
        } catch (Throwable e) {...}
    }
}
```

#### 1.2.3.3 发送数据
##### 1.2.3.3.1 write()
```java
// 通过最初创建的FSDataOutputStream调用write方法
// fos.write()
@Override
    public void write(byte[] b) throws IOException {
        write(b, 0, b.length);
}

// 调用FSOutputSummer的run方法
public synchronized void write(int b) throws IOException {
    buf[count++] = (byte)b;
    if(count == buf.length) {
      // 写文件
      flushBuffer();
    }
}

protected synchronized void flushBuffer() throws IOException {
    flushBuffer(false, true);
}

protected synchronized int flushBuffer(boolean keep,
      boolean flushPartial) throws IOException {
    int bufLen = count;
    int partialLen = bufLen % sum.getBytesPerChecksum();
    int lenToFlush = flushPartial ? bufLen : bufLen - partialLen;
    // 没有需要flush的数据时检验checksum
    if (lenToFlush != 0) {
      writeChecksumChunks(buf, 0, lenToFlush);
      if (!flushPartial || keep) {
        count = partialLen;
        System.arraycopy(buf, bufLen - count, buf, 0, count);
      } else {
        count = 0;
      }
    }

    // total bytes left minus unflushed bytes left
    return count - (bufLen - lenToFlush);
}

private void writeChecksumChunks(byte b[], int off, int len)
  throws IOException {
    // 计算chunk的校验和
    sum.calculateChunkedSums(b, off, len, checksum, 0);
    TraceScope scope = createWriteTraceScope();
    try {
      // 先写chunk和checksum，够127个形成一个packet
      // 按chunk的大小遍历数据
      for (int i = 0; i < len; i += sum.getBytesPerChecksum()) {
        int chunkLen = Math.min(sum.getBytesPerChecksum(), len - i);
        int ckOffset = i / sum.getBytesPerChecksum() * getChecksumSize();
        // 向chunk写数据
        writeChunk(b, off + i, chunkLen, checksum, ckOffset,
            getChecksumSize());
      }
    } finally {
      if (scope != null) {
        scope.close();
      }
    }
}
```
##### 1.2.3.3.2 writeChunk()
```java
// DFSOutputStream.java
protected synchronized void writeChunk(byte[] b, int offset, int len,
      byte[] checksum, int ckoff, int cklen) throws IOException {
	// 创建Packet
    writeChunkPrepare(len, ckoff, cklen);
	// 向packet写chunk的校验和
    currentPacket.writeChecksum(checksum, ckoff, cklen);
    // 向packet里写入一个chunk
    currentPacket.writeData(b, offset, len);
    // 计算写入了多少个chunk，写满了127个就成为一个完整的packet
    currentPacket.incNumChunks();
    getStreamer().incBytesCurBlock(len);

    // If packet is full, enqueue it for transmission
    // packet写满或block写满后将packet加入队列，队列满的时候阻塞
    if (currentPacket.getNumChunks() == currentPacket.getMaxChunks() ||
        getStreamer().getBytesCurBlock() == blockSize) {
      enqueueCurrentPacketFull();
    }
}

private synchronized void writeChunkPrepare(int buflen,
      int ckoff, int cklen) throws IOException {
    if (currentPacket == null) {
      // 创建一个packet
      currentPacket = createPacket(packetSize, chunksPerPacket, getStreamer().getBytesCurBlock(), getStreamer().getAndIncCurrentSeqno(), false);
    }
}

protected DFSPacket createPacket(int packetSize, int chunksPerPkt,
      long offsetInBlock, long seqno, boolean lastPacketInBlock)
      throws InterruptedIOException {

    return new DFSPacket(buf, chunksPerPkt, offsetInBlock, seqno,
        getChecksumSize(), lastPacketInBlock);
  }
```
##### 1.2.3.3.3 enqueueCurrentPacketFull
```java
// 将packet加入队列
synchronized void enqueueCurrentPacketFull() throws IOException {
    // 加入队列
    enqueueCurrentPacket();
    adjustChunkBoundary();
    // 一个block写完后，再创建一个空的packet标识块的结束
    endBlock();
}

void enqueueCurrentPacket() throws IOException {
    getStreamer().waitAndQueuePacket(currentPacket);
    currentPacket = null;
}

void waitAndQueuePacket(DFSPacket packet) throws IOException {
    synchronized (dataQueue) {
        try {
            // If queue is full, then wait till we have enough space
            boolean firstWait = true;
       
            // 第一次进入，队列不是满的
            checkClosed();
            queuePacket(packet);
        } catch (ClosedChannelException cce) {
            LOG.debug("Closed channel exception", cce);
        }
    }
}

void queuePacket(DFSPacket packet) {
    synchronized (dataQueue) {
      if (packet == null) return;
      packet.addTraceParent(Tracer.getCurrentSpan());
      dataQueue.addLast(packet);
      lastQueuedSeqno = packet.getSeqno();
      LOG.debug("Queued {}, {}", packet, this);
      // 唤醒dataQueue
      dataQueue.notifyAll();
    }
}

// 创建空的block标志结束
void endBlock() throws IOException {
    if (getStreamer().getBytesCurBlock() == blockSize) {
      setCurrentPacketToEmpty();
      enqueueCurrentPacket();
      getStreamer().setBytesCurBlock(0);
      lastFlushOffset = 0;
    }
}

void setCurrentPacketToEmpty() throws InterruptedIOException {
    // 创建一个空的packet
    currentPacket = createPacket(0, 0, getStreamer().getBytesCurBlock(),
        getStreamer().getAndIncCurrentSeqno(), true);
    currentPacket.setSyncBlock(shouldSyncBlock);
}
```
### 1.2.4 DN 开启 DataXceiver 服务，接收数据返回 ack
#### 1.2.4.1 DataXceiver.run()
```java
// DN启动时作为守护进程
public void run() {
    Peer peer = null;
    while (datanode.shouldRun && !datanode.shutdownForUpgrade) {
      try {
        peer = peerServer.accept();

        // Make sure the xceiver count is not exceeded
        int curXceiverCount = datanode.getXceiverCount();
        if (curXceiverCount > maxXceiverCount) {
          throw new IOException("Xceiver count " + curXceiverCount
              + " exceeds the limit of concurrent xcievers: "
              + maxXceiverCount);
        }
		// 通过DataXceiverServer创建启动DataXceiver线程
        new Daemon(datanode.threadGroup,
            DataXceiver.create(peer, datanode, this))
            .start();
    closeAllPeers();
}

public static DataXceiver create(Peer peer, DataNode dn,
                                 DataXceiverServer dataXceiverServer) throws IOException {
    return new DataXceiver(peer, dn, dataXceiverServer);
}

public void run() {
    int opsProcessed = 0;
    Op op = null;
    try {
        ...
            // protected void initialize(final DataInputStream in) {
            // this.in = in;
            // }
            super.initialize(new DataInputStream(input));

        // We process requests in a loop, and stay around for a short timeout.
        // This optimistic behaviour allows the other end to reuse connections.
        // Setting keepalive timeout to 0 disable this behavior.
        do {
            updateCurrentThreadName("Waiting for operation #" + (opsProcessed + 1));

            try {
                    // 读取操作的类型，此处传入的是Sender在writeBlock时传入的Op.WRITE_BLOCK
                    op = readOp();
            } 
                processOp(op);
            ++opsProcessed;
        }
    } 
}

protected final void processOp(Op op) throws IOException {
    switch(op) {
    case WRITE_BLOCK:
      opWriteBlock(in);
      break;
    default:
      throw new IOException("Unknown op " + op + " in data stream");
    }
}
```
#### 1.2.4.2 opWriteBlock()
```java
// Receiver.java
private void opWriteBlock(DataInputStream in) throws IOException {
    final OpWriteBlockProto proto = OpWriteBlockProto.parseFrom(vintPrefixed(in));
    final DatanodeInfo[] targets = PBHelperClient.convert(proto.getTargetsList());
    TraceScope traceScope = continueTraceSpan(proto.getHeader(),
        proto.getClass().getSimpleName());
    try {
      writeBlock(...);
    } 
}

public void writeBlock(final ExtendedBlock block,
      final StorageType storageType, 
      final Token<BlockTokenIdentifier> blockToken,
      final String clientname,
      final DatanodeInfo[] targets,
      final StorageType[] targetStorageTypes,
      final DatanodeInfo srcDataNode,
      final BlockConstructionStage stage,
      final int pipelineSize,
      final long minBytesRcvd,
      final long maxBytesRcvd,
      final long latestGenerationStamp,
      DataChecksum requestedChecksum,
      CachingStrategy cachingStrategy,
      boolean allowLazyPersist,
      final boolean pinning,
      final boolean[] targetPinnings,
      final String storageId,
      final String[] targetStorageIds) throws IOException {
    
    // 开始写    
    try {
      final Replica replica;
      if (isDatanode || 
          stage != BlockConstructionStage.PIPELINE_CLOSE_RECOVERY) {
        // open a block receiver
        // 创建BlockReceiver
        setCurrentBlockReceiver(getBlockReceiver(block, storageType, in,
            peer.getRemoteAddressString(),
            peer.getLocalAddressString(),
            stage, latestGenerationStamp, minBytesRcvd, maxBytesRcvd,
            clientname, srcDataNode, datanode, requestedChecksum,
            cachingStrategy, allowLazyPersist, pinning, storageId));
        replica = blockReceiver.getReplica();
      } else {...}
      storageUuid = replica.getStorageUuid();
      isOnTransientStorage = replica.isOnTransientStorage();

      //
      // Connect to downstream machine, if appropriate
      // 如果存在多个节点
      if (targets.length > 0) {
        InetSocketAddress mirrorTarget = null;
        // Connect to backup machine
        mirrorNode = targets[0].getXferAddr(connectToDnViaHostname);
        LOG.debug("Connecting to datanode {}", mirrorNode);
        mirrorTarget = NetUtils.createSocketAddr(mirrorNode);
        mirrorSock = datanode.newSocket();
        try {

          DataNodeFaultInjector.get().failMirrorConnection();

          int timeoutValue = dnConf.socketTimeout +
              (HdfsConstants.READ_TIMEOUT_EXTENSION * targets.length);
          int writeTimeout = dnConf.socketWriteTimeout +
              (HdfsConstants.WRITE_TIMEOUT_EXTENSION * targets.length);
          NetUtils.connect(mirrorSock, mirrorTarget, timeoutValue);
          mirrorSock.setTcpNoDelay(dnConf.getDataTransferServerTcpNoDelay());
          mirrorSock.setSoTimeout(timeoutValue);
          mirrorSock.setKeepAlive(true);
          if (dnConf.getTransferSocketSendBufferSize() > 0) {
            mirrorSock.setSendBufferSize(
                dnConf.getTransferSocketSendBufferSize());
          }

          OutputStream unbufMirrorOut = NetUtils.getOutputStream(mirrorSock,
              writeTimeout);
          InputStream unbufMirrorIn = NetUtils.getInputStream(mirrorSock);
          DataEncryptionKeyFactory keyFactory =
            datanode.getDataEncryptionKeyFactoryForBlock(block);
          SecretKey secretKey = null;
          if (dnConf.overwriteDownstreamDerivedQOP) {
            String bpid = block.getBlockPoolId();
            BlockKey blockKey = datanode.blockPoolTokenSecretManager
                .get(bpid).getCurrentKey();
            secretKey = blockKey.getKey();
          }
          IOStreamPair saslStreams = datanode.saslClient.socketSend(
              mirrorSock, unbufMirrorOut, unbufMirrorIn, keyFactory,
              blockToken, targets[0], secretKey);
          unbufMirrorOut = saslStreams.out;
          unbufMirrorIn = saslStreams.in;
          mirrorOut = new DataOutputStream(new BufferedOutputStream(unbufMirrorOut,
              smallBufferSize));
          mirrorIn = new DataInputStream(unbufMirrorIn);

          String targetStorageId = null;
          if (targetStorageIds.length > 0) {
            // Older clients may not have provided any targetStorageIds
            targetStorageId = targetStorageIds[0];
          }
          if (targetPinnings != null && targetPinnings.length > 0) {
            // 向下一个DN发送写信息
            new Sender(mirrorOut).writeBlock(originalBlock, targetStorageTypes[0],
                blockToken, clientname, targets, targetStorageTypes,
                srcDataNode, stage, pipelineSize, minBytesRcvd, maxBytesRcvd,
                latestGenerationStamp, requestedChecksum, cachingStrategy,
                allowLazyPersist, targetPinnings[0], targetPinnings,
                targetStorageId, targetStorageIds);
          } else {
            new Sender(mirrorOut).writeBlock(originalBlock, targetStorageTypes[0],
                blockToken, clientname, targets, targetStorageTypes,
                srcDataNode, stage, pipelineSize, minBytesRcvd, maxBytesRcvd,
                latestGenerationStamp, requestedChecksum, cachingStrategy,
                allowLazyPersist, false, targetPinnings,
                targetStorageId, targetStorageIds);
          }
  }
```
#### 1.2.4.3 DN 将数据持久化到磁盘
```java
BlockReceiver getBlockReceiver(
      final ExtendedBlock block, final StorageType storageType,
      final DataInputStream in,
      final String inAddr, final String myAddr,
      final BlockConstructionStage stage,
      final long newGs, final long minBytesRcvd, final long maxBytesRcvd,
      final String clientname, final DatanodeInfo srcDataNode,
      final DataNode dn, DataChecksum requestedChecksum,
      CachingStrategy cachingStrategy,
      final boolean allowLazyPersist,
      final boolean pinning,
      final String storageId) throws IOException {
    return new BlockReceiver(block, storageType, in,
        inAddr, myAddr, stage, newGs, minBytesRcvd, maxBytesRcvd,
        clientname, srcDataNode, dn, requestedChecksum,
        cachingStrategy, allowLazyPersist, pinning, storageId);
}

BlockReceiver(final ExtendedBlock block, final StorageType storageType,
      final DataInputStream in,
      final String inAddr, final String myAddr,
      final BlockConstructionStage stage, 
      final long newGs, final long minBytesRcvd, final long maxBytesRcvd, 
      final String clientname, final DatanodeInfo srcDataNode,
      final DataNode datanode, DataChecksum requestedChecksum,
      CachingStrategy cachingStrategy,
      final boolean allowLazyPersist,
      final boolean pinning,
      final String storageId) throws IOException {
      ...
      this.isDatanode = clientname.length() == 0;
      // Open local disk out
      if (isDatanode) { //replication or move
        replicaHandler =
            datanode.data.createTemporary(storageType, storageId, block, false);
      } else {
        switch (stage) {
        // 第一次进入为PIPELINE_SETUP_CREATE阶段
        case PIPELINE_SETUP_CREATE:
          // 写磁盘
          replicaHandler = datanode.data.createRbw(storageType, storageId,
              block, allowLazyPersist);
          datanode.notifyNamenodeReceivingBlock(
              block, replicaHandler.getReplica().getStorageUuid());
          break;
        case PIPELINE_SETUP_STREAMING_RECOVERY:
        }
}

// FsDatasetImpl.java
public ReplicaHandler createRbw(
      StorageType storageType, String storageId, ExtendedBlock b,
      boolean allowLazyPersist) throws IOException {
      // create a new block
      FsVolumeReference ref = null;

      // Use ramdisk only if block size is a multiple of OS page size.
      // This simplifies reservation for partially used replicas
      // significantly.
      if (allowLazyPersist &&
          lazyWriter != null &&
          b.getNumBytes() % cacheManager.getOsPageSize() == 0 &&
          reserveLockedMemory(b.getNumBytes())) {
        try {
          // First try to place the block on a transient volume.
          ref = volumes.getNextTransientVolume(b.getNumBytes());
          datanode.getMetrics().incrRamDiskBlocksWrite();
        } catch (DiskOutOfSpaceException de) {
          // Ignore the exception since we just fall back to persistent storage.
          LOG.warn("Insufficient space for placing the block on a transient "
              + "volume, fall back to persistent storage: "
              + de.getMessage());
        } finally {
          if (ref == null) {
            cacheManager.release(b.getNumBytes());
          }
        }
      }
}
```
#### 1.2.4.4 DataStreamer 写 Packet
```java
// run()
private void sendPacket(DFSPacket packet) throws IOException {
    // write out data to remote datanode
    try {
      packet.writeTo(blockStream);
      blockStream.flush();
    } catch (IOException e) {
      // HDFS-3398 treat primary DN is down since client is unable to
      // write to primary DN. If a failed or restarting node has already
      // been recorded by the responder, the following call will have no
      // effect. Pipeline recovery can handle only one node error at a
      // time. If the primary node fails again during the recovery, it
      // will be taken out then.
      errorState.markFirstNodeIfNotMarked();
      throw e;
    }
    lastPacket = Time.monotonicNow();
}

// DFSPacket.java
public synchronized void writeTo(DataOutputStream stm) throws IOException {
    checkBuffer();

    final int dataLen = dataPos - dataStart;
    final int checksumLen = checksumPos - checksumStart;
    final int pktLen = HdfsConstants.BYTES_IN_INTEGER + dataLen + checksumLen;

    PacketHeader header = new PacketHeader(
        pktLen, offsetInBlock, seqno, lastPacketInBlock, dataLen, syncBlock);

    if (checksumPos != dataStart) {
      // Move the checksum to cover the gap. This can happen for the last
      // packet or during an hflush/hsync call.
      System.arraycopy(buf, checksumStart, buf,
          dataStart - checksumLen , checksumLen);
      checksumPos = dataStart;
      checksumStart = checksumPos - checksumLen;
    }

    final int headerStart = checksumStart - header.getSerializedSize();
    assert checksumStart + 1 >= header.getSerializedSize();
    assert headerStart >= 0;
    assert headerStart + header.getSerializedSize() == checksumStart;

    // Copy the header data into the buffer immediately preceding the checksum
    // data.
    System.arraycopy(header.getBytes(), 0, buf, headerStart,
        header.getSerializedSize());

    // corrupt the data for testing.
    if (DFSClientFaultInjector.get().corruptPacket()) {
      buf[headerStart+header.getSerializedSize() + checksumLen + dataLen-1] ^=
          0xff;
    }

    // Write the now contiguous full packet to the output stream.
    // 写数据
    stm.write(buf, headerStart,
        header.getSerializedSize() + checksumLen + dataLen);
}
```
## 1.3 读流程
### 1.3.1 开启 FSDataInputStream
```java
public FSDataInputStream open(Path f) throws IOException {
    return open(f, getConf().getInt(IO_FILE_BUFFER_SIZE_KEY,
        IO_FILE_BUFFER_SIZE_DEFAULT));
}

// DistributedFileSystem.java
public FSDataInputStream open(Path f, final int bufferSize)
      throws IOException {
    statistics.incrementReadOps(1);
    storageStatistics.incrementOpCounter(OpType.OPEN);
    Path absF = fixRelativePart(f);
    return new FileSystemLinkResolver<FSDataInputStream>() {
      @Override
      public FSDataInputStream doCall(final Path p) throws IOException {
        final DFSInputStream dfsis =
            dfs.open(getPathName(p), bufferSize, verifyChecksum);
        try {
          return dfs.createWrappedInputStream(dfsis);
        }
      }
   }
}
    
// DFSClient.java
public DFSInputStream open(String src, int buffersize, boolean verifyChecksum)
      throws IOException {
    checkOpen();
    //    Get block info from namenode
    try (TraceScope ignored = newPathTraceScope("newDFSInputStream", src)) {
      LocatedBlocks locatedBlocks = getLocatedBlocks(src, 0);
      return openInternal(locatedBlocks, src, verifyChecksum);
    }
}
```
### 1.3.2 向 NN 建立连接，获取块地址
```java
public LocatedBlocks getLocatedBlocks(String src, long start)
      throws IOException {
    return getLocatedBlocks(src, start, dfsClientConf.getPrefetchSize());
}

public LocatedBlocks getLocatedBlocks(String src, long start, long length)
      throws IOException {
    try (TraceScope ignored = newPathTraceScope("getBlockLocations", src)) {
      return callGetBlockLocations(namenode, src, start, length);
    }
}

static LocatedBlocks callGetBlockLocations(ClientProtocol namenode,
      String src, long start, long length)
      throws IOException {
    try {
      return namenode.getBlockLocations(src, start, length);
    } 
}

// 通过NNRPCServer获取块位置
public LocatedBlocks getBlockLocations(String src, 
                                          long offset, 
                                          long length) 
      throws IOException {
    checkNNStartup();
    metrics.incrGetBlockLocations();
    LocatedBlocks locatedBlocks =
        namesystem.getBlockLocations(getClientMachine(), src, offset, length);
    return locatedBlocks;
}

// FSNamesystem.java
LocatedBlocks getBlockLocations(String clientMachine, String srcArg,
      long offset, long length) throws IOException {
    final String operationName = "open";
    checkOperation(OperationCategory.READ);
    GetBlockLocationsResult res = null;
    // FSPermissionChecker检查权限
    final FSPermissionChecker pc = getPermissionChecker();
    FSPermissionChecker.setOperationType(operationName);
    final INode inode;
    try {
      readLock();
      try {
        checkOperation(OperationCategory.READ);
        // 获取块位置
        res = FSDirStatAndListingOp.getBlockLocations(
            dir, pc, srcArg, offset, length, true);
        inode = res.getIIp().getLastINode();
    ...  
    LocatedBlocks blocks = res.blocks;
    sortLocatedBlocks(clientMachine, blocks);
    return blocks;
      }
    }
}

// FSDirStatAndListingOp.java        
static GetBlockLocationsResult getBlockLocations(
      FSDirectory fsd, FSPermissionChecker pc, String src, long offset,
      long length, boolean needBlockToken) throws IOException {
    Preconditions.checkArgument(offset >= 0,
        "Negative offset is not supported. File: " + src);
    Preconditions.checkArgument(length >= 0,
        "Negative length is not supported. File: " + src);
    BlockManager bm = fsd.getBlockManager();
    fsd.readLock();
    try {
      // 检查权限以及路径
      final INodesInPath iip = fsd.resolvePath(pc, src, DirOp.READ);
      src = iip.getPath();
      final INodeFile inode = INodeFile.valueOf(iip.getLastINode(), src);
      if (fsd.isPermissionEnabled()) {
        fsd.checkPathAccess(pc, iip, FsAction.READ);
        fsd.checkUnreadableBySuperuser(pc, iip);
      }
      
      final LocatedBlocks blocks = bm.createLocatedBlocks(
          inode.getBlocks(iip.getPathSnapshotId()), fileSize, isUc, offset,
          length, needBlockToken, iip.isSnapshot(), feInfo, ecPolicy);
}

// BlockManager.java
public LocatedBlocks createLocatedBlocks(final BlockInfo[] blocks,
      final long fileSizeExcludeBlocksUnderConstruction,
      final boolean isFileUnderConstruction, final long offset,
      final long length, final boolean needBlockToken,
      final boolean inSnapshot, FileEncryptionInfo feInfo,
      ErasureCodingPolicy ecPolicy)
      throws IOException {
    assert namesystem.hasReadLock();
    if (blocks == null) {...}
    else {
      // blocks不是快照，长度也不是0
      if (LOG.isDebugEnabled()) {
        LOG.debug("blocks = {}", java.util.Arrays.asList(blocks));
      }

	  // 创建块列表
      createLocatedBlockList(locatedBlocks, blocks, offset, length, mode);
      return locations;
    }
}

private void createLocatedBlockList(
      LocatedBlockBuilder locatedBlocks,
      final BlockInfo[] blocks,
      final long offset, final long length,
      final AccessMode mode) throws IOException {
    
    do {
      locatedBlocks.addBlock(
          createLocatedBlock(locatedBlocks, blocks[curBlk], curPos, mode));
      curPos += blocks[curBlk].getNumBytes();
      curBlk++;
    }
}

private LocatedBlock createLocatedBlock(LocatedBlockBuilder locatedBlocks,  
    final BlockInfo blk, final long pos, final AccessMode mode)  
        throws IOException {  
  // 从DatanodeStorageInfo中获取到块信息
  final LocatedBlock lb = createLocatedBlock(locatedBlocks, blk, pos);  
  if (mode != null) {  
    setBlockToken(lb, mode);  
  }  
  return lb;  
}
```
### 1.3.3 读取数据
```java
// 获取块位置后，返回DFSClient中的open()继续执行
private DFSInputStream openInternal(LocatedBlocks locatedBlocks, String src,
      boolean verifyChecksum) throws IOException {
    if (locatedBlocks != null) {
      // 使用了EC策略时，返回DFSStripedInputStream，否则返回DFSInputStream
      // 在 HDFS 中，EC 文件是以 stripe 的形式进行存储
      ErasureCodingPolicy ecPolicy = locatedBlocks.getErasureCodingPolicy();
      if (ecPolicy != null) {
        return new DFSStripedInputStream(this, src, verifyChecksum, ecPolicy,
            locatedBlocks);
      }
      return new DFSInputStream(this, src, verifyChecksum, locatedBlocks);
    }
}

// 通过DFSInputStream读取数据
public int read(long position, byte[] buffer, int offset, int length)
      throws IOException {
    validatePositionedReadArgs(position, buffer, offset, length);
    if (length == 0) {
      return 0;
    }
    ByteBuffer bb = ByteBuffer.wrap(buffer, offset, length);
    return pread(position, bb);
}

private int pread(long position, ByteBuffer buffer)
      throws IOException {
    failures = 0;
    // 获取LocatedBlocks的长度
    long filelen = getFileLength();

    // 指定范围获取数据
    List<LocatedBlock> blockRange = getBlockRange(position, realLen);
    int remaining = realLen;
    CorruptedBlocks corruptedBlocks = new CorruptedBlocks();
    // 遍历block列表，读取需要的block的数据
    for (LocatedBlock blk : blockRange) {
      long targetStart = position - blk.getStartOffset();
      int bytesToRead = (int) Math.min(remaining,
          blk.getBlockSize() - targetStart);
      long targetEnd = targetStart + bytesToRead - 1;
      try {
        if (dfsClient.isHedgedReadsEnabled() && !blk.isStriped()) {
          hedgedFetchBlockByteRange(blk, targetStart,
              targetEnd, buffer, corruptedBlocks);
        } else {
          // 读数据
          fetchBlockByteRange(blk, targetStart, targetEnd,
              buffer, corruptedBlocks);
        }
      }
}
```
#### 1.3.3.1 getBlockRange()
```java
private List<LocatedBlock> getBlockRange(long offset,
      long length)  throws IOException {
    // getFileLength(): returns total file length
    // locatedBlocks.getFileLength(): returns length of completed blocks
    synchronized(infoLock) {
      final List<LocatedBlock> blocks;
      // block两种状态：已完成以及未完成
      final long lengthOfCompleteBlk = locatedBlocks.getFileLength();
      final boolean readOffsetWithinCompleteBlk = offset < lengthOfCompleteBlk;
      final boolean readLengthPastCompleteBlk = offset + length > lengthOfCompleteBlk;

      if (readOffsetWithinCompleteBlk) {
        //get the blocks of finalized (completed) block range
        blocks = getFinalizedBlockRange(offset,
          Math.min(length, lengthOfCompleteBlk - offset));
      } else {
        blocks = new ArrayList<>(1);
      }

      // get the blocks from incomplete block range
      if (readLengthPastCompleteBlk) {
        blocks.add(locatedBlocks.getLastLocatedBlock());
      }

      return blocks;
    }
}

private List<LocatedBlock> getFinalizedBlockRange(
      long offset, long length) throws IOException {
    synchronized(infoLock) {
      List<LocatedBlock> blockRange = new ArrayList<>();
      // search cached blocks first
      long remaining = length;
      long curOff = offset;
      while(remaining > 0) {
        // 从NN中取一个块并缓存
        LocatedBlock blk = fetchBlockAt(curOff, remaining, true);
        blockRange.add(blk);
        long bytesRead = blk.getStartOffset() + blk.getBlockSize() - curOff;
        remaining -= bytesRead;
        curOff += bytesRead;
      }
      return blockRange;
    }
}

private LocatedBlock fetchBlockAt(long offset, long length, boolean useCache)
      throws IOException {
    synchronized(infoLock) {
      updateBlockLocationsStamp();
      // 从缓存的locatedBlocks查找offset所在block在链表中的位置
      int targetBlockIdx = locatedBlocks.findBlock(offset);
      if (targetBlockIdx < 0) { // block is not cached
        targetBlockIdx = LocatedBlocks.getInsertIndex(targetBlockIdx);
        useCache = false;
      }
      if (!useCache) { // fetch blocks
        final LocatedBlocks newBlocks = (length == 0)
            ? dfsClient.getLocatedBlocks(src, offset)
            : dfsClient.getLocatedBlocks(src, offset, length);
        if (newBlocks == null || newBlocks.locatedBlockCount() == 0) {
          throw new EOFException("Could not find target position " + offset);
        }
        // Update the LastLocatedBlock, if offset is for last block.
        if (offset >= locatedBlocks.getFileLength()) {
          locatedBlocks = newBlocks;
          lastBlockBeingWrittenLength = getLastBlockLength();
        } else {
          locatedBlocks.insertRange(targetBlockIdx,
              newBlocks.getLocatedBlocks());
        }
      }
      return locatedBlocks.get(targetBlockIdx);
    }
}
```
#### 1.3.3.2 fetchBlockByteRange()
```java
protected void fetchBlockByteRange(LocatedBlock block, long start, long end,
      ByteBuffer buf, CorruptedBlocks corruptedBlocks)
      throws IOException {
    while (true) {
      DNAddrPair addressPair = chooseDataNode(block, null);
      // Latest block, if refreshed internally
      block = addressPair.block;
      try {
        // 读数据
        actualGetFromOneDataNode(addressPair, start, end, buf,
            corruptedBlocks);
        return;
      }
    }
}

private DNAddrPair chooseDataNode(LocatedBlock block,
      Collection<DatanodeInfo> ignoredNodes) throws IOException {
    return chooseDataNode(block, ignoredNodes, true);
}

private DNAddrPair chooseDataNode(LocatedBlock block,
      Collection<DatanodeInfo> ignoredNodes, boolean refetchIfRequired)
      throws IOException {
    while (true) {
      // 选择一个最佳的node
      DNAddrPair result = getBestNodeDNAddrPair(block, ignoredNodes);
      if (result != null) {
        return result;
      } else if (refetchIfRequired) {
        block = refetchLocations(block, ignoredNodes);
      } else {
        return null;
      }
    }
}

void actualGetFromOneDataNode(final DNAddrPair datanode, final long startInBlk,
      final long endInBlk, ByteBuffer buf, CorruptedBlocks corruptedBlocks)
      throws IOException {
    DFSClientFaultInjector.get().startFetchFromDatanode();
    int refetchToken = 1; // only need to get a new access token once
    int refetchEncryptionKey = 1; // only need to get a new encryption key once
    final int len = (int) (endInBlk - startInBlk + 1);
    LocatedBlock block = datanode.block;
    while (true) {
      BlockReader reader = null;
      try {
        DFSClientFaultInjector.get().fetchFromDatanodeException();
        // 创建reader
        // new BlockReaderFactory(...).build();
        reader = getBlockReader(block, startInBlk, len, datanode.addr,
            datanode.storageType, datanode.info);

        //Behave exactly as the readAll() call
        ByteBuffer tmp = buf.duplicate();
        tmp.limit(tmp.position() + len);
        tmp = tmp.slice();
        int nread = 0;
        int ret;
        while (true) {
          // 读取逻辑
          // BlockReader有四个实现类
          // 1. BlockReaderLocalLegacy以及BlockReaderLocal适用于DFS客户端和NN在同一节点的情况
          // 2. BlockReaderRemote
          // 3. ExternalBlockReader
          ret = reader.read(tmp);
          if (ret <= 0) {
            break;
          }
          nread += ret;
        }
        buf.position(buf.position() + nread);
	  ...
      } finally {
        if (reader != null) {
          // 关闭reader
          reader.close();
        }
      }
    }
}
```
## 1.4 文件系统
- `NameNode` 会维护文件系统的命名空间，`hdfs` 文件系统的命名空间是以"/" 为根目录开始的整棵目录树，整棵目录树是通过 `FSDirectory` 类来管理的
- 在 `HDFS` 中，无论是目录还是文件，在文件系统目录树中都被看做是一个 `INode` 节点，如果是目录，则对应的类是 `INodeDirectory`，如果是文件，则对应的类是 `INodeFile`；`INodeDirectory` 类以及 `INodeFile` 类都是 `INode` 的子类。其中 `INodeDirectory` 中包含一个成员集合变量 `children`，如果该目录下有子目录或者文件，其子目录或文件的 `INode` 引用就会被保存在 `children` 集合中。`HDFS` 就是通过这种方式维护整个文件
### 1.4.1 INode
```java
// 基础抽象类，其内部保存了hdfs文件和目录共有的基本属性；包括当前节点的父节点的INode对象的引用、
// 文件、目录名、用户组、访问权限等, INode类中只有一个属性parent，表明当前INode的父目录
// 内部定义了公共属性字段对应的方法:
// userName：文件/目录所属用户名
// groupName：文件/目录所属用户组
// fsPermission：文件或者目录权限
// modificationTime：上次修改时间
// accessTime：上次访问时间
// 以及INode的元数据信息
// id：INode的id
// name：文件/目录的名称
// fullPathName：文件/目录的完整路径
// parent：文件/目录的父节点

public abstract class INode implements INodeAttributes, Diff.Element<byte[]> {
    public static final Logger LOG = LoggerFactory.getLogger(INode.class);

    /** parent is either an {@link INodeDirectory} or an {@link INodeReference}.*/
    private INode parent = null;
    ...
}
```
### 1.4.2 INodeFile
```java
// 在文件系统目录树中，使用INodeFile抽象成一个hdfs文件，它也是INode的子类。INodeFile中包含了两个文件特有的属性：文件信息头header和文件对应的数据块信息blocks

public class INodeFile extends INodeWithAdditionalFields
    implements INodeFileAttributes, BlockCollection {
    // 定义了两个属性文件信息头header和文件对应的数据块信息blocks
    // 使用了和INode.permission一样的方法，在一个长整形变量里保存了文件的副本系数和文件数据块的大小
    // 高16字节存放着副本系数，低48位存放了数据块大小
    private long header = 0L;
    
    // 保存了文件拥有的所有数据块信息，数组元素类型是BlockInfo。BlockInfo类继承自Block类
    // 保存了数据块与文件、数据块与数据节点的对应关系
  	private BlockInfo[] blocks;
    ...
}
```
### 1.4.3 INodeDirectory
```java
// INodeDirectory抽象了HDFS中的目录，目录里面保存了文件和其他子目录。
// 在INodeDirectory实现中的体现是其成员变量children，它是一个用于保存INode的列表。
// INodeDirectory中的大部分方法都是在操作这个列表，如创建子目录项、查询或遍历子目录项、替换子目录项
public class INodeDirectory extends INodeWithAdditionalFields
    implements INodeDirectoryAttributes {
    ...
    private List<INode> children = null;
   
}
```
### 1.4.4 FSNamesystem
`FSNamesystem` 是瞬态和持久`namespace`状态的容器，并在 `NameNode` 上完成所有记录工作
主要作用有：
- 是 `BlockManager`、`DatanodeManager`、`DelegationTokens`、`LeaseManager` 等服务的容器
- 委托处理修改或检查`namespace`的 `RPC`调用
- 任何只涉及块的东西（例如块报告），它都委托给 `BlockManager`
- 任何只涉及文件信息（例如权限、mkdirs）的操作，它都会委托给 `FSDirectory`
- 任何跨越上述两个组件的东西在这里协调
- 记录变动到`FsEditLog`
```java
public class FSNamesystem implements Namesystem, FSNamesystemMBean,
NameNodeMXBean, ReplicatedBlocksMBean, ECBlockGroupsMBean {
	// 操作文件时通过dir来调用
    FSDirectory dir;
    // 操作块时委托BlockManager
    private BlockManager blockManager;
    final LeaseManager leaseManager = new LeaseManager(this); 
}
```
## 1.5 块管理
### 1.5.1 Block
`Block`继承结构为基类`Block`,`BlockInfo`继承自`Block`，`BlocksMap`是`BlockInfo`的一个`hashmap`存储
```java
// Block需要在NameNode，DataNodes间进行传输，所以进行了序列化
public class Block implements Writable, Comparable<Block> {
    // 主要属性
    private long blockId;
    private long numBytes;
    private long generationStamp;
}
```
### 1.5.2 BlockInfo
```java
// BlockInfo维护了块所在的BlockCollection，以及该Block备份到了哪几个DataNodes中
public abstract class BlockInfo extends Block
    implements LightWeightGSet.LinkedElement {
    
    private short replication;
    private volatile long bcId;
    // triplets存放的是引用的三元组，表示BlockInfo被复制在多个DN中
    // triplets[3*i]存储DatanodeStorageInfo对象，表示Block所在的DN
    // triplets[3*i+1]以及triplets[3*i+2]存储BlockInfoContiguous对象，表示Block所在的DN前一个以及后一个Block
    // 通过triplets可以获取块信息
    protected Object[] triplets;
}
```
### 1.5.3 BlocksMap
```java
class BlocksMap {
  // 内部定义了StorageIterator用于遍历存储信息
  public static class StorageIterator implements Iterator<DatanodeStorageInfo> {...}
  // BlocksMap通过GSet来存储块和块信息的映射
  private GSet<Block, BlockInfo> blocks;
}
```
### 1.5.4 DatanodeStorageInfo
```java
/**
 * A Datanode has one or more storages. A storage in the Datanode is represented
 * by this class.
 */
public class DatanodeStorageInfo{
  // NN对DN的抽象
  // DatanodeDescriptor继承了DatanodeID，DatanodeID封装了DN相关的一些基本信息，如ip，host等
  private final DatanodeDescriptor dn;
  private final String storageID;
  private StorageType storageType; //存储类型，disk，ssd等
  private State state;

  private long capacity; 
  private long dfsUsed;
  private volatile long remaining;
  private long blockPoolUsed;

  private volatile BlockInfoContiguous blockList = null;
  private int numBlocks = 0;

  // The ID of the last full block report which updated this storage.
  private long lastBlockReportId = 0;

  /** The number of block reports received */
  private int blockReportCount = 0;
```
### 1.5.5 BlockManager
```java
// BlockManager功能之一用于维护NN中的数据块信息，存储的信息包括两个部分：
// Block与其所在DN的对应关系，通过BlockInfo中triplets保存的DatanodeStorageInfo进行保存
// 而BlockInfo存储在BlocksMap中，另一部分是DN与其所保存的Block之间的关系，通过DatanodeStorageInfo中的blockList进行保存
// BlockManager通过修改这两部分数据来修改数据块
public class BlockManager implements BlockStatsMXBean {
    
    // 几个重要属性
    private final Namesystem namesystem;
    private final DatanodeManager datanodeManager;
  	private final HeartbeatManager heartbeatManager;
	...
}

// FSNamesystem初始化时会创建BlockManager对象：
public BlockManager(final Namesystem namesystem, boolean haEnabled,
      final Configuration conf) throws IOException {
    // 默认副本数 -> 3
    this.defaultReplication = conf.getInt(DFSConfigKeys.DFS_REPLICATION_KEY,
        DFSConfigKeys.DFS_REPLICATION_DEFAULT);
    // 最大默认副本数 -> 512 最小默认副本数 -> 1
    final int maxR = conf.getInt(DFSConfigKeys.DFS_REPLICATION_MAX_KEY,
        DFSConfigKeys.DFS_REPLICATION_MAX_DEFAULT);
    final int minR = conf.getInt(DFSConfigKeys.DFS_NAMENODE_REPLICATION_MIN_KEY,
        DFSConfigKeys.DFS_NAMENODE_REPLICATION_MIN_DEFAULT);
    // 块报告期间要记录信息的最大块数 -> 1000l
    this.maxNumBlocksToLog =
        conf.getLong(DFSConfigKeys.DFS_MAX_NUM_BLOCKS_TO_LOG_KEY,
            DFSConfigKeys.DFS_MAX_NUM_BLOCKS_TO_LOG_DEFAULT);
}
```
## 1.6 契约机制
- `HDFS`中的契约机制，同一时间只允许一个客户端获取`NameNode`上一个文件的契约，然后才可以向获取契约的文件写入数据。其他客户端尝试获取文件契约的时候只能等待，从而保证同一时间只有一个客户端在写一个文件。客户端获取到契约后，开启一个线程向`NN`请求文件续约，NN后台也有监控契约的线程，如果没有续约，或是没有继续写入文件，契约就会过期，交给别的客户端来写
### 1.6.1 addLease()
```java
// 写操作时FSDirWriteFileOp的startFile方法中会调用addLease()
fsn.leaseManager.addLease(
        newNode.getFileUnderConstructionFeature().getClientName(),
        newNode.getId());
// LeaseManager.java
synchronized Lease addLease(String holder, long inodeId) {
    Lease lease = getLease(holder);
    if (lease == null) {
      lease = new Lease(holder);
      // 没有lease的话添加一个lease对象做绑定
      // leases通过HashMap保存leaseHolder -> Lease的映射
      leases.put(holder, lease);
    } else {
      renewLease(lease);
    }
    // 再将inodeId和lease对象放进TreeSet里
    leasesById.put(inodeId, lease);
    lease.files.add(inodeId);
    return lease;
}
```
### 1.6.2 Lease
- 文件契约就是将文件和`Lease`对象存在`HashMap`和`TreeMap`中，每个文件有只能有一个契约，就通过比较来选择最新的契约
```java
// Lease是LeaseManager的内部类
/*
 * A Lease governs all the locks held by a single client.
 * For each client there's a corresponding lease, whose
 * timestamp is updated when the client periodically
 * checks in.  If the client dies and allows its lease to
 * expire, all the corresponding locks can be released.
*/
class Lease {
    private final String holder;
    private long lastUpdate;
    private final HashSet<Long> files = new HashSet<>();

    /** Only LeaseManager object can create a lease */
    private Lease(String h) {
      this.holder = h;
      renew();
    }
    /** Only LeaseManager object can renew a lease */
    private void renew() {
      this.lastUpdate = monotonicNow();
    }

    /** Does this lease contain any path? */
    boolean hasFiles() {return !files.isEmpty();}

    boolean removeFile(long inodeId) {
      return files.remove(inodeId);
    }

    private Collection<Long> getFiles() {...}

    String getHolder() {...}
  }
```
### 1.6.3 beginFileLease()
```java
// FDSClient调用newStreamForCreate时，返回的DFSOutputStream传给beginFileLease方法，用于续约
private void beginFileLease(final long inodeId, final DFSOutputStream out)
      throws IOException {
    synchronized (filesBeingWritten) {
      putFileBeingWritten(inodeId, out);
      LeaseRenewer renewer = getLeaseRenewer();
      // 执行LeaseRenewer内部定义的run方法
      boolean result = renewer.put(this);
      if (!result) {
        // Existing LeaseRenewer cannot add another Daemon, so remove existing
        // and add new one.
        LeaseRenewer.remove(renewer);
        renewer = getLeaseRenewer();
        renewer.put(this);
      }
    }
}

// 通过renew续约
private void renew() throws IOException {
    final List<DFSClient> copies;
    synchronized(this) {
      copies = new ArrayList<>(dfsclients);
    }
    //sort the client names for finding out repeated names.
    Collections.sort(copies, new Comparator<DFSClient>() {
      @Override
      public int compare(final DFSClient left, final DFSClient right) {
        return left.getClientName().compareTo(right.getClientName());
      }
    });
    String previousName = "";
    for (final DFSClient c : copies) {
      //skip if current client name is the same as the previous name.
      if (!c.getClientName().equals(previousName)) {
        // 遍历Client续约
        // nameNodeRpcServer.renewLease()调用FSNamesystem的renewLease()再调用LeaseManager的renewLease()
        if (!c.renewLease()) {
          LOG.debug("Did not renew lease for client {}", c);
          continue;
        }
        previousName = c.getClientName();
        LOG.debug("Lease renewed for client {}", previousName);
      }
    }
  }
```
### 1.6.4 checkLeases()
```java
// LeaseManager中有一个内部类Monitor用于监控契约
class Monitor implements Runnable {
    final String name = getClass().getSimpleName();

    /** Check leases periodically. */
    @Override
    public void run() {
        for(; shouldRunMonitor && fsnamesystem.isRunning(); ) {
            boolean needSync = false;
			try {
				if (!fsnamesystem.isInSafeMode()) {
					needSync = checkLeases(candidates);
				}
        }
    }
}

private synchronized boolean checkLeases(Collection<Lease> leasesToCheck) {
    ...
    //检查契约 
    for (Lease leaseToCheck : leasesToCheck) {
      if (isMaxLockHoldToReleaseLease(start)) {
        break;
      }
      // 检查是否超时 
      if (!leaseToCheck.expiredHardLimit(Time.monotonicNow())) {
        continue;
      }
      // 将检查完需要移除的契约添加至ArrayList removing中
      for(Long id : removing) {
        removeLease(leaseToCheck, id);
      }
    }
    return needSync;
}
```
### 1.6.5 removeLease()
```java
private synchronized void removeLease(Lease lease, long inodeId) {
  leasesById.remove(inodeId);
  if (!lease.removeFile(inodeId)) {
    LOG.debug("inode {} not found in lease.files (={})", inodeId, lease);
  }
  if (!lease.hasFiles()) {
    if (leases.remove(lease.holder) == null) {
      LOG.error("{} not found", lease);
    }
  }
}
```
# 2 MR
## 2.1 提交 Job
### 2.1.1 waitForCompletion()
```java
public class Job extends JobContextImpl implements JobContext, AutoCloseable {

    public enum JobState {DEFINE, RUNNING}; //定义枚举变量JobState
    private JobState state = JobState.DEFINE; //创建的Job的JobState默认为DEFINE

    public static Job getInstance() throws IOException {
        return getInstance(new Configuration()); //public的构造器均为@Depracated，通过getInstance方法返回示例对象
    }

    public static Job getInstance(Configuration conf) throws IOException {
        JobConf jobConf = new JobConf(conf);
        return new Job(jobConf); //调用default的构造器
    }

    //提交时调用
    public boolean waitForCompletion(boolean verbose
                                    ) throws IOException, InterruptedException,
    ClassNotFoundException {
        if (state == JobState.DEFINE) { //RUNNING态的job不会提交第二次
            submit(); //提交Job
        }
        if (verbose) {
            monitorAndPrintJob(); //设置verbose打印日志信息
        } else {...}
        return isSuccessful();
    }
}
```
### 2.1.2 submit()
```java
public void submit() 
    throws IOException, InterruptedException, ClassNotFoundException {
    // 确保当前Job的状态为DEFINE
    ensureState(JobState.DEFINE);
    // 处理新旧API
    setUseNewAPI();
    // 返回XXXRunner对象用于提交Job
    connect();

    //getJobSubmitter()方法返回包含集群信息的JobSubmitter对象
    final JobSubmitter submitter = 
        getJobSubmitter(cluster.getFileSystem(), cluster.getClient());

    //private JobStatus status;
    //JobContextImpl.java -> protected UserGroupInformation ugi;
    //public <T> T doAs(PrivilegedExceptionAction<T> action) doAs对象返回的是泛型类型的值
    status = ugi.doAs(new PrivilegedExceptionAction<JobStatus>() { //返回JobStatus对象
        public JobStatus run() throws IOException, InterruptedException, 
        ClassNotFoundException {
            // 提交
            return submitter.submitJobInternal(Job.this, cluster);
        }
    });
    state = JobState.RUNNING;
    LOG.info("The url to track the job: " + getTrackingURL());
}

synchronized void connect()
    throws IOException, InterruptedException, ClassNotFoundException {
    if (cluster == null) {
        cluster = 
            ugi.doAs(new PrivilegedExceptionAction<Cluster>() {
                public Cluster run()
                    throws IOException, InterruptedException, 
                ClassNotFoundException {
                    return new Cluster(getConfiguration()); //返回Cluster对象, 传给doAs方法, 将Cluster对象赋给cluster
                }
            });
    }
}
```
### 2.1.3 new Cluster()
```java
public Cluster(Configuration conf) throws IOException {
    this(null, conf);
}

public Cluster(InetSocketAddress jobTrackAddr, Configuration conf) 
    throws IOException {
    this.conf = conf;
    this.ugi = UserGroupInformation.getCurrentUser();
    initialize(jobTrackAddr, conf);
}

private void initialize(InetSocketAddress jobTrackAddr, Configuration conf)
      throws IOException {

    for (ClientProtocolProvider provider : providerList) {
      try {
        if (jobTrackAddr == null) {
          clientProtocol = provider.create(conf); //create有两个实现类
        } else {
          clientProtocol = provider.create(jobTrackAddr, conf);
        }
    }
}

// 根据运行Job位置的不同，create有两个实现类:LocalClientProtocolProvider和YarnClientProtocolProvider
public ClientProtocol create(Configuration conf) throws IOException {
    String framework =
        /* MRConfig类中定义的FRAMEWORK_NAME = "mapreduce.framework.name"
                           LOCAL_FRAMEWORK_NAME = "local"
        */
        conf.get(MRConfig.FRAMEWORK_NAME, MRConfig.LOCAL_FRAMEWORK_NAME);
    if (!MRConfig.LOCAL_FRAMEWORK_NAME.equals(framework)) {
        return null;
    }
    //local的情况下设置map数为1
    conf.setInt(JobContext.NUM_MAPS, 1);
    //返回LocalJobRunner对象
    return new LocalJobRunner(conf);
}
```
### 2.1.4 submitJobInternal()
```java
// JobSubmitter.java
// 通过Cluster提交任务
JobStatus submitJobInternal(Job job, Cluster cluster) 
  throws ClassNotFoundException, InterruptedException, IOException {

    //检查输出路径是否设置，或者是否已经存在
    checkSpecs(job);

    Configuration conf = job.getConfiguration();
    addMRFrameworkToDistributedCache(conf);
	
    //通过Cluster对象和conf获取staging文件夹的路径
    Path jobStagingArea = JobSubmissionFiles.getStagingDir(cluster, conf);
   
   // 通过Inet获得ip
    InetAddress ip = InetAddress.getLocalHost();
    if (ip != null) {
      //通过ip得到提交的地址，hostname并通过conf设置
      submitHostAddress = ip.getHostAddress();
      submitHostName = ip.getHostName();
      conf.set(MRJobConfig.JOB_SUBMITHOST,submitHostName);
      conf.set(MRJobConfig.JOB_SUBMITHOSTADDR,submitHostAddress);
    }
    JobID jobId = submitClient.getNewJobID();
    job.setJobID(jobId);
    //将staging路径歌jobId拼接成最终存放切片文件, 配置文件等的目录
    Path submitJobDir = new Path(jobStagingArea, jobId.toString());
    
    try{
      ...
      // 切片
      int maps = writeSplits(job, submitJobDir);
      conf.setInt(MRJobConfig.NUM_MAPS, maps);
      LOG.info("number of splits:" + maps);

      int maxMaps = conf.getInt(MRJobConfig.JOB_MAX_MAP,
          MRJobConfig.DEFAULT_JOB_MAX_MAP);
      ...
      // 上传配置文件
      writeConf(conf, submitJobFile);
      
      //
      // Now, actually submit the job (using the submit name)
      //

      status = submitClient.submitJob(
          jobId, submitJobDir.toString(), job.getCredentials());
}
```
#### 2.1.4.1 checkSpecs()
```java
private void checkSpecs(Job job) throws ClassNotFoundException, 
      InterruptedException, IOException {
    JobConf jConf = (JobConf)job.getConfiguration();
          
    /*
        getUseNewMapper和getUseNewReducer返回boolean
       
        public boolean getUseNewMapper() {
            return getBoolean("mapred.mapper.new-api", false);
        } 
        */
    if (jConf.getNumReduceTasks() == 0 ? 
        jConf.getUseNewMapper() : jConf.getUseNewReducer()) {
      org.apache.hadoop.mapreduce.OutputFormat<?, ?> output =
        ReflectionUtils.newInstance(job.getOutputFormatClass(),
          job.getConfiguration()); //output默认是TextOutputFormat
      output.checkOutputSpecs(job); //通过output调用
    } else {
      jConf.getOutputFormat().checkOutputSpecs(jtFs, jConf);
    }
}

public Class<? extends OutputFormat<?,?>> getOutputFormatClass() 
     throws ClassNotFoundException {
    return (Class<? extends OutputFormat<?,?>>) 
      conf.getClass(OUTPUT_FORMAT_CLASS_ATTR, TextOutputFormat.class);
} //返回TextOutputFormat的Class对象

//TextOutputFormat没有重写checkOutputSpecs方法，直接调用抽象父类OutputFormat的方法
public void checkOutputSpecs(JobContext job
    ) throws FileAlreadyExistsException, IOException{

    Path outDir = getOutputPath(job);
    if (outDir == null) {
      throw new InvalidJobConfException("Output directory not set.");
    }

    TokenCache.obtainTokensForNamenodes(job.getCredentials(),
        new Path[] { outDir }, job.getConfiguration());

    if (outDir.getFileSystem(job.getConfiguration()).exists(outDir)) {
      throw new FileAlreadyExistsException("Output directory " + outDir + " already exists");
    }
}
```
#### 2.1.4.2 writeSplits()
```java
private int writeSplits(org.apache.hadoop.mapreduce.JobContext job,
      Path jobSubmitDir) throws IOException,
      InterruptedException, ClassNotFoundException {
    JobConf jConf = (JobConf)job.getConfiguration();
    int maps;
    if (jConf.getUseNewMapper()) {
      maps = writeNewSplits(job, jobSubmitDir); // 使用新API，调用writeNewSplits()
    } else {
      maps = writeOldSplits(jConf, jobSubmitDir);
    }
    return maps;	
}

private <T extends InputSplit>
  int writeNewSplits(JobContext job, Path jobSubmitDir) throws IOException,
      InterruptedException, ClassNotFoundException {
    Configuration conf = job.getConfiguration();
    
    //默认返回TextInputFormat
    InputFormat<?, ?> input =
      ReflectionUtils.newInstance(job.getInputFormatClass(), conf);
	//通过List存放切片对象
    List<InputSplit> splits = input.getSplits(job);
    T[] array = (T[]) splits.toArray(new InputSplit[splits.size()]);

    // sort the splits into order based on size, so that the biggest
    // go first
    Arrays.sort(array, new SplitComparator());
    JobSplitWriter.createSplitFiles(jobSubmitDir, conf, 
        jobSubmitDir.getFileSystem(conf), array);
    return array.length;
}
```
#### 2.1.4.3 getSplits()
```java
//TextInputFormat没有重写抽象父类的方法，调用的是FileInputFormat的方法
public List<InputSplit> getSplits(JobContext job) throws IOException {
    StopWatch sw = new StopWatch().start();
    
    long minSize = Math.max(getFormatMinSplitSize(), getMinSplitSize(job));
    long maxSize = getMaxSplitSize(job);

    // generate splits
    List<InputSplit> splits = new ArrayList<InputSplit>();
    List<FileStatus> files = listStatus(job);

    boolean ignoreDirs = !getInputDirRecursive(job)
      && job.getConfiguration().getBoolean(INPUT_DIR_NONRECURSIVE_IGNORE_SUBDIRS, false);
    for (FileStatus file: files) {
      	...
        // 可切，不是压缩文件    
        if (isSplitable(job, path)) {
          long blockSize = file.getBlockSize();
          // 计算切片大小
          long splitSize = computeSplitSize(blockSize, minSize, maxSize);

          long bytesRemaining = length;
          while (((double) bytesRemaining)/splitSize > SPLIT_SLOP) {
            int blkIndex = getBlockIndex(blkLocations, length-bytesRemaining);
            // 创建FileSplit对象添加至集合中
            splits.add(makeSplit(path, length-bytesRemaining, splitSize,
                        blkLocations[blkIndex].getHosts(),
                        blkLocations[blkIndex].getCachedHosts()));
            bytesRemaining -= splitSize;
          }

          if (bytesRemaining != 0) {
            int blkIndex = getBlockIndex(blkLocations, length-bytesRemaining);
            splits.add(makeSplit(path, length-bytesRemaining, bytesRemaining,
                       blkLocations[blkIndex].getHosts(),
                       blkLocations[blkIndex].getCachedHosts()));
          }
        } else { // not splitable
          splits.add(makeSplit(path, 0, length, blkLocations[0].getHosts(), blkLocations[0].getCachedHosts()));
        }
      } else { 
        //Create empty hosts array for zero length files
        splits.add(makeSplit(path, 0, length, new String[0]));
      }
    }
    // Save the number of input files for metrics/loadgen
    return splits;
}

// 切片最小和最大大小计算
public static long getMinSplitSize(JobContext job) {
    return job.getConfiguration().getLong(SPLIT_MINSIZE, 1L);
}
public static final String SPLIT_MINSIZE = 
    "mapreduce.input.fileinputformat.split.minsize";

protected long getFormatMinSplitSize() {
    return 1;
}

public static long getMaxSplitSize(JobContext context) {
    return context.getConfiguration().getLong(SPLIT_MAXSIZE, 
                                              Long.MAX_VALUE);
}

public static final String SPLIT_MAXSIZE = 
    "mapreduce.input.fileinputformat.split.maxsize";

// 计算切片大小
protected long computeSplitSize(long blockSize, long minSize,
                                  long maxSize) {
    return Math.max(minSize, Math.min(maxSize, blockSize));
}

protected FileSplit makeSplit(Path file, long start, long length, 
                                String[] hosts) {
    return new FileSplit(file, start, length, hosts);
}
```
#### 2.1.4.4 createSplitFiles()
```java
// 1.7中返回的splits集合按大小排序后，通过JobSplitWriter的createSplitFiles方法，生成最终的切片文件
public static <T extends InputSplit> void createSplitFiles(Path jobSubmitDir, 
      Configuration conf, FileSystem fs, T[] splits) 
  throws IOException, InterruptedException {
    // 创建切片文件
    FSDataOutputStream out = createFile(fs, 
        JobSubmissionFiles.getJobSplitFile(jobSubmitDir), conf);
    // 向out中写入切片信息
    SplitMetaInfo[] info = writeNewSplits(conf, splits, out);
    out.close();
    // 再写一次元数据
    writeJobSplitMetaInfo(fs,JobSubmissionFiles.getJobSplitMetaFile(jobSubmitDir), 
        new FsPermission(JobSubmissionFiles.JOB_FILE_PERMISSION), splitVersion,
        info);
}

private static <T extends InputSplit> 
    SplitMetaInfo[] writeNewSplits(Configuration conf, 
                                   T[] array, FSDataOutputStream out)
    throws IOException, InterruptedException {
	// 切片元数据
    SplitMetaInfo[] info = new SplitMetaInfo[array.length];
    if (array.length != 0) {
        SerializationFactory factory = new SerializationFactory(conf);
        int i = 0;
        int maxBlockLocations = conf.getInt(MRConfig.MAX_BLOCK_LOCATIONS_KEY,
                                            MRConfig.MAX_BLOCK_LOCATIONS_DEFAULT);
        long offset = out.getPos();
        for(T split: array) {
            long prevCount = out.getPos();
            Text.writeString(out, split.getClass().getName());
            // 通过工厂模式获取序列化器
            Serializer<T> serializer = 
                factory.getSerializer((Class<T>) split.getClass());
            serializer.open(out);
            // 序列化
            serializer.serialize(split);
            long currCount = out.getPos();
            String[] locations = split.getLocations();
            if (locations.length > maxBlockLocations) {
                LOG.warn("Max block location exceeded for split: "
                         + split + " splitsize: " + locations.length +
                         " maxsize: " + maxBlockLocations);
                locations = Arrays.copyOf(locations, maxBlockLocations);
            }
            info[i++] = 
                new JobSplit.SplitMetaInfo( 
                locations, offset,
                split.getLength());
            offset += currCount - prevCount;
        }
    }
    return info;
}
```
#### 2.1.4.5 writeConf()
```java
private void writeConf(Configuration conf, Path jobFile) 
      throws IOException {
    // Write job file to JobTracker's fs        
    FSDataOutputStream out = 
      FileSystem.create(jtFs, jobFile, 
                        new FsPermission(JobSubmissionFiles.JOB_FILE_PERMISSION));
    try {
      conf.writeXml(out);
    } finally {
      out.close();
    }
}
```
#### 2.1.4.6 submitJob()
```java
// 最终的提交也分Local和Yarn两种模式
// Local
public org.apache.hadoop.mapreduce.JobStatus submitJob(
      org.apache.hadoop.mapreduce.JobID jobid, String jobSubmitDir,
      Credentials credentials) throws IOException {
    // 此处的Job是LocalJobRunner的内部类
    Job job = new Job(JobID.downgrade(jobid), jobSubmitDir);
    job.job.setCredentials(credentials);
    return job.status;
}
```
### 2.1.5 准备开始执行 Job
```java
public Job(JobID jobid, String jobSubmitDir) throws IOException {
      this.systemJobDir = new Path(jobSubmitDir);
      this.systemJobFile = new Path(systemJobDir, "job.xml");
      this.id = jobid;
      // 通过job.xml生成JobConf对象
      JobConf conf = new JobConf(systemJobFile);
      this.localFs = FileSystem.getLocal(conf);
      String user = UserGroupInformation.getCurrentUser().getShortUserName();
      this.localJobDir = localFs.makeQualified(new Path(
          new Path(conf.getLocalPath(jobDir), user), jobid.toString()));
      this.localJobFile = new Path(this.localJobDir, id + ".xml");

      // Manage the distributed cache.  If there are files to be copied,
      // this will trigger localFile to be re-written again.
      localDistributedCacheManager = new LocalDistributedCacheManager();
      localDistributedCacheManager.setup(conf);
      
      // Write out configuration file.  Instead of copying it from
      // systemJobFile, we re-write it, since setup(), above, may have
      // updated it.
      OutputStream out = localFs.create(localJobFile);
      try {
        conf.writeXml(out);
      } finally {
        out.close();
      }
      this.job = new JobConf(localJobFile);

      // Job (the current object) is a Thread, so we wrap its class loader.
      if (localDistributedCacheManager.hasLocalClasspaths()) {
        setContextClassLoader(localDistributedCacheManager.makeClassLoader(
                getContextClassLoader()));
      }
      
      profile = new JobProfile(job.getUser(), id, systemJobFile.toString(), 
                               "http://localhost:8080/", job.getJobName());
      status = new JobStatus(id, 0.0f, 0.0f, JobStatus.RUNNING, 
          profile.getUser(), profile.getJobName(), profile.getJobFile(), 
          profile.getURL().toString());
	  // private HashMap<JobID, Job> jobs = new HashMap<JobID, Job>();
      jobs.put(id, this);
	  ...
      this.start();
}

public void run() {
	// 获取ReduceTask个数    
	int numReduceTasks = job.getNumReduceTasks();
	outputCommitter.setupJob(jContext);
	status.setSetupProgress(1.0f);
	// 
	List<RunnableWithThrowable> mapRunnables = getMapTaskRunnables(
		taskSplitMetaInfos, jobId, mapOutputFiles);
		  
	initCounters(mapRunnables.size(), numReduceTasks);
	ExecutorService mapService = createMapExecutor();
	runTasks(mapRunnables, mapService, "map");

	try {
	  if (numReduceTasks > 0) {
		List<RunnableWithThrowable> reduceRunnables = getReduceTaskRunnables(
			jobId, mapOutputFiles);
		ExecutorService reduceService = createReduceExecutor();
		runTasks(reduceRunnables, reduceService, "reduce");
	  }
	} finally {
	  for (MapOutputFile output : mapOutputFiles.values()) {
		output.removeAll();
	  }
	}
	// delete the temporary directory in output directory
	outputCommitter.commitJob(jContext);
	status.setCleanupProgress(1.0f);
}
```
### 2.1.6 getMapTaskRunnables()
```java
// 创建一个MapTaskRunnable对象
protected List<RunnableWithThrowable> getMapTaskRunnables(
        TaskSplitMetaInfo [] taskInfo, JobID jobId,
        Map<TaskAttemptID, MapOutputFile> mapOutputFiles) {

      int numTasks = 0;
      ArrayList<RunnableWithThrowable> list =
          new ArrayList<RunnableWithThrowable>();
      for (TaskSplitMetaInfo task : taskInfo) {
        list.add(new MapTaskRunnable(task, numTasks++, jobId,
            mapOutputFiles));
      }

      return list;
}

// 启动Task
private void runTasks(List<RunnableWithThrowable> runnables,
        ExecutorService service, String taskType) throws Exception {
      // Start populating the executor with work units.
      // They may begin running immediately (in other threads).
      // 遍历task的runnable对象
      for (Runnable r : runnables) {
        service.submit(r);
      }
      ...
}

//AbstractExecutorService.java
public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
    	// 返回一个FutureTask对象
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
}

//ThreadPoolExecutor.java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        // 线程池有资源，addWorker创建Worker对象来调度启动MapRunable线程，开始MapTask
        // Worker有一个thread属性，与线程绑定后，会调用线程的start
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
}

private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (int c = ctl.get();;) {
            // Check if queue empty only if necessary.
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP)
                    || firstTask != null
                    || workQueue.isEmpty()))
                return false;

            for (;;) {
                if (workerCountOf(c)
                    >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateAtLeast(c, SHUTDOWN))
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 创建worker
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int c = ctl.get();

                    if (isRunning(c) ||
                        (runStateLessThan(c, STOP) && firstTask == null)) {
                        if (t.getState() != Thread.State.NEW)
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        workerAdded = true;
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    // 启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
}
```
## 2.2 Map
### 2.2.1 run()
```java
public void run(final JobConf job, final TaskUmbilicalProtocol umbilical)
    throws IOException, ClassNotFoundException, InterruptedException {
    this.umbilical = umbilical;
	// 判断有没有ReduceTask, 调整Map阶段的比例
    if (isMapTask()) {
      // If there are no reducers then there won't be any sort. Hence the map 
      // phase will govern the entire attempt's progress.
      if (conf.getNumReduceTasks() == 0) {
        mapPhase = getProgress().addPhase("map", 1.0f);
      } else {
        // If there are reducers then the entire attempt's progress will be 
        // split between the map phase (67%) and the sort phase (33%).
        mapPhase = getProgress().addPhase("map", 0.667f);
        sortPhase  = getProgress().addPhase("sort", 0.333f);
      }
    }
    boolean useNewApi = job.getUseNewMapper();
    // 初始化
    initialize(job, getJobID(), reporter, useNewApi);

    if (useNewApi) {
      // 运行mapper
      runNewMapper(job, splitMetaInfo, umbilical, reporter);
    }
}
```
#### 2.2.1.1 initialize()
```java
public void initialize(JobConf job, JobID id, 
                         Reporter reporter,
                         boolean useNewApi) throws IOException, 
                                                   ClassNotFoundException,
                                                   InterruptedException {
    jobContext = new JobContextImpl(job, id, reporter);
    taskContext = new TaskAttemptContextImpl(job, taskId, reporter);
    if (getState() == TaskStatus.State.UNASSIGNED) {
      setState(TaskStatus.State.RUNNING);
    }
    if (useNewApi) {
      if (LOG.isDebugEnabled()) {
        LOG.debug("using new api for output committer");
      }
      // 设置outputFormat类型，默认TextOutputFormat
      outputFormat =
        ReflectionUtils.newInstance(taskContext.getOutputFormatClass(), job);
      committer = outputFormat.getOutputCommitter(taskContext);
    } else {
      committer = conf.getOutputCommitter();
    }
    // 输出路径                                                   
    Path outputPath = FileOutputFormat.getOutputPath(conf);
    if (outputPath != null) {
      if ((committer instanceof FileOutputCommitter)) {
        FileOutputFormat.setWorkOutputPath(conf, 
          ((FileOutputCommitter)committer).getTaskAttemptPath(taskContext));
      } else {
        FileOutputFormat.setWorkOutputPath(conf, outputPath);
      }
    }
    ...
}

// JobContextImpl.java
public Class<? extends OutputFormat<?,?>> getOutputFormatClass() 
     throws ClassNotFoundException {
    return (Class<? extends OutputFormat<?,?>>) 
      conf.getClass(OUTPUT_FORMAT_CLASS_ATTR, TextOutputFormat.class);
}

public static final String OUTPUT_FORMAT_CLASS_ATTR = "mapreduce.job.outputformat.class";

// getClass第一个参数为null时会返回默认值
public Class<?> getClass(String name, Class<?> defaultValue) {
    String valueString = getTrimmed(name);
    if (valueString == null)
      return defaultValue;
    try {
      return getClassByName(valueString);
    } catch (ClassNotFoundException e) {
      throw new RuntimeException(e);
    }
}
```
#### 2.2.1.2 runNewMapper()
```java
private <INKEY,INVALUE,OUTKEY,OUTVALUE>
  void runNewMapper(final JobConf job,
                    final TaskSplitIndex splitIndex,
                    final TaskUmbilicalProtocol umbilical,
                    TaskReporter reporter
                    ) throws IOException, ClassNotFoundException,
                             InterruptedException {
    // make a task context so we can get the classes
    org.apache.hadoop.mapreduce.TaskAttemptContext taskContext =
      new org.apache.hadoop.mapreduce.task.TaskAttemptContextImpl(job, 
                                                                  getTaskID(),
                                                                  reporter);
    // make a mapper
    // 同样通过getClass获取自定义的Mapper
    org.apache.hadoop.mapreduce.Mapper<INKEY,INVALUE,OUTKEY,OUTVALUE> mapper =
      (org.apache.hadoop.mapreduce.Mapper<INKEY,INVALUE,OUTKEY,OUTVALUE>)
        ReflectionUtils.newInstance(taskContext.getMapperClass(), job);
    // make the input format
    // 默认TextInputFormat
    org.apache.hadoop.mapreduce.InputFormat<INKEY,INVALUE> inputFormat =
      (org.apache.hadoop.mapreduce.InputFormat<INKEY,INVALUE>)
        ReflectionUtils.newInstance(taskContext.getInputFormatClass(), job);
 	...
    // 创建RecordReader读取数据
    org.apache.hadoop.mapreduce.RecordReader<INKEY,INVALUE> input =
      new NewTrackingRecordReader<INKEY,INVALUE>
        (split, inputFormat, reporter, taskContext);    
    // get an output object
    // 创建Collector接收Mapper的输出
    if (job.getNumReduceTasks() == 0) {
      output = 
        new NewDirectOutputCollector(taskContext, job, umbilical, reporter);
    } else {
      output = new NewOutputCollector(taskContext, job, umbilical, reporter);
    }

    org.apache.hadoop.mapreduce.MapContext<INKEY, INVALUE, OUTKEY, OUTVALUE> 
    mapContext = 
      // 将上下文进行封装, mapContext的key就是LineRecordReader读取到一行的偏移量
      new MapContextImpl<INKEY, INVALUE, OUTKEY, OUTVALUE>(job, getTaskID(), 
          input, output, 
          committer, 
          reporter, split);

org.apache.hadoop.mapreduce.Mapper<INKEY,INVALUE,OUTKEY,OUTVALUE>.Context 
        mapperContext = 
          new WrappedMapper<INKEY, INVALUE, OUTKEY, OUTVALUE>().getMapContext(
              mapContext);

    try {
      // LineRecordReader的initialize方法
      input.initialize(split, mapperContext);
      // 执行Map任务
      mapper.run(mapperContext);
      mapPhase.complete();
      // 写完buffer后会调用close()
      output.close(mapperContext);
      output = null;
    } 
}
```
### 2.2.2 new NewTrackingRecordReader()
```java
// MapTask的静态内部类
NewTrackingRecordReader(org.apache.hadoop.mapreduce.InputSplit split,
        org.apache.hadoop.mapreduce.InputFormat<K, V> inputFormat,
        TaskReporter reporter,
        org.apache.hadoop.mapreduce.TaskAttemptContext taskContext)
        throws InterruptedException, IOException {
        
      // 调用TextInputFormat的createRecordReader
      this.real = inputFormat.createRecordReader(split, taskContext);
      long bytesInCurr = getInputBytes(fsStats);
      fileInputByteCounter.increment(bytesInCurr - bytesInPrev);
}

public RecordReader<LongWritable, Text> 
    createRecordReader(InputSplit split,
                       TaskAttemptContext context) {
    String delimiter = context.getConfiguration().get(
        "textinputformat.record.delimiter");
    byte[] recordDelimiterBytes = null;
    if (null != delimiter)
      recordDelimiterBytes = delimiter.getBytes(Charsets.UTF_8);
    return new LineRecordReader(recordDelimiterBytes);
}

public LineRecordReader(byte[] recordDelimiter) {
    this.recordDelimiterBytes = recordDelimiter;
}
```
### 2.2.3 new NewOutputCollector()
```java
// ReduceTask个数不为0
NewOutputCollector(org.apache.hadoop.mapreduce.JobContext jobContext,
                       JobConf job,
                       TaskUmbilicalProtocol umbilical,
                       TaskReporter reporter
                       ) throws IOException, ClassNotFoundException {
      // 创建Collector
      collector = createSortingCollector(job, reporter);
      partitions = jobContext.getNumReduceTasks();
      if (partitions > 1) {
        // 分区器默认HashPartitioner
        partitioner = (org.apache.hadoop.mapreduce.Partitioner<K,V>)
          ReflectionUtils.newInstance(jobContext.getPartitionerClass(), job);
      }
}

private <KEY, VALUE> MapOutputCollector<KEY, VALUE>
          createSortingCollector(JobConf job, TaskReporter reporter)
    throws IOException, ClassNotFoundException {
    MapOutputCollector.Context context =
      new MapOutputCollector.Context(this, job, reporter);
	// 默认MapOutputBuffer
    Class<?>[] collectorClasses = job.getClasses(
      JobContext.MAP_OUTPUT_COLLECTOR_CLASS_ATTR, MapOutputBuffer.class);
        // 创建collector的对象
        MapOutputCollector<KEY, VALUE> collector =
          ReflectionUtils.newInstance(subclazz, job);
        // 初始化环形缓冲区，见Shuffle  
        collector.init(context);
       
        return collector;
      } 
    }
}
```
### 2.2.4 initialize()
```java
// 根据切片的压缩方式采用不同的Reader
// SplitLineReader和UncompressedSplitLineReader都继承自LineReader，都会调用父类的readLine方法
// 不指定分隔符，以CR, LF, CRLF结尾
public void initialize(InputSplit genericSplit,
                         TaskAttemptContext context) throws IOException {
    FileSplit split = (FileSplit) genericSplit;
    Configuration job = context.getConfiguration();
    this.maxLineLength = job.getInt(MAX_LINE_LENGTH, Integer.MAX_VALUE);
    start = split.getStart();
    end = start + split.getLength();
    final Path file = split.getPath();

    // open the file and seek to the start of the split
    CompressionCodec codec = new CompressionCodecFactory(job).getCodec(file);
    if (null!=codec) {
      isCompressedInput = true;
      decompressor = CodecPool.getDecompressor(codec);
      // 支持切片的压缩方式
      if (codec instanceof SplittableCompressionCodec) {
        final SplitCompressionInputStream cIn =
          ((SplittableCompressionCodec)codec).createInputStream(
            fileIn, decompressor, start, end,
            SplittableCompressionCodec.READ_MODE.BYBLOCK);
        in = new CompressedSplitLineReader(cIn, job,
            this.recordDelimiterBytes);
        start = cIn.getAdjustedStart();
        end = cIn.getAdjustedEnd();
        filePosition = cIn;
      } else {
        // 不支持切片的压缩
        in = new SplitLineReader(codec.createInputStream(fileIn,
            decompressor), job, this.recordDelimiterBytes);
        filePosition = fileIn;
      }
    } else {
      // 不压缩  
      fileIn.seek(start);
      in = new UncompressedSplitLineReader(
          fileIn, job, this.recordDelimiterBytes, split.getLength());
      filePosition = fileIn;
    }
}
```
### 2.2.5 Mapper.run()
```java
// 自定义的Mapper没有重写run方法，调用父类的run()
public void run(Context context) throws IOException, InterruptedException {
    setup(context);
    try {
      while (context.nextKeyValue()) {
        // 调用自定义的map方法  
        // 通过context获取K和V传给Mapper
        map(context.getCurrentKey(), context.getCurrentValue(), context);
      }
    } finally {
      cleanup(context);
    }
}

// 将数据写出
// 此处的Context是Mapper中的抽象类，在WrappedMapper中有一个内部实现类Context
context.write()

//MapContextImpl继承了TaskInputOutputContextImpl
public void write(KEYOUT key, VALUEOUT value
                    ) throws IOException, InterruptedException {
    output.write(key, value);
}

// 调用MapTask的write方法，送到环形缓冲区
public void write(K key, V value) throws IOException, InterruptedException {
      collector.collect(key, value,
                        partitioner.getPartition(key, value, partitions));
}
```
## 2.3 Shuffle
### 2.3.1 MapOutputBuffer 类的属性
- 环形缓冲区其实是一个数组，数组中存放着 kv 的序列化数据和 kv 的元数据信息
	- `kvmeta` 以 `int` 类型存储，每个 kv 对应一个元数据
	- 元数据由 4 个 `int` 组成
		- 第一个 `int` 存放 `value` 的起始位置
		- 第二个存放 `key` 的起始位置
		- 第三个存放 `partition`
		- 最后一个存放 `value` 的长度
- kv 序列化的数据和元数据在环形缓冲区中的存储是由 `equator` 分隔
- kv 按照索引递增的方向存储，`kvmeta` 则按照索引递减的方向存储
- 将其数组抽象为一个环形结构，以 `equator` 为界，kv 顺时针存储，`kvmeta` 逆时针存储
```java
public static class MapOutputBuffer<K extends Object, V extends Object>
      implements MapOutputCollector<K, V>, IndexedSortable {
    private int partitions;
    private JobConf job;
    private TaskReporter reporter;
    private Class<K> keyClass;
    private Class<V> valClass;
    private RawComparator<K> comparator;
    private SerializationFactory serializationFactory;
    private Serializer<K> keySerializer;
    private Serializer<V> valSerializer;
    private CombinerRunner<K,V> combinerRunner;
    private CombineOutputCollector<K, V> combineCollector;

    // Compression for map-outputs
    private CompressionCodec codec;

    // k/v accounting
    // IntBuffer用于存放kv元数据
    private IntBuffer kvmeta; // metadata overlay on backing store
    int kvstart;            // marks origin of spill metadata
    int kvend;              // marks end of spill metadata
    int kvindex;            // marks end of fully serialized records
	// 分割meta和kv数据的标识
    int equator;            // marks origin of meta/serialization
    int bufstart;           // marks beginning of spill
    int bufend;             // marks beginning of collectable
    int bufmark;            // marks end of record
    int bufindex;           // marks end of collected
    int bufvoid;            // marks the point where we should stop
                            // reading at the end of the buffer
	// 存放数据的byte数组
    byte[] kvbuffer;        // main output buffer
    private final byte[] b0 = new byte[0];
	// kv在kvbuffer中的地址存放位置与kvindex的距离
    private static final int VALSTART = 0;         // val offset in acct
    private static final int KEYSTART = 1;         // key offset in acct
    // kvmeta存放partition位置与kvindex的距离
    private static final int PARTITION = 2;        // partition offset in acct
    private static final int VALLEN = 3;           // length of value
    // 一对kvmeta占用4个int
    private static final int NMETA = 4;            // num meta ints
    // int占4个字节，每个meta数据占用byte数
    private static final int METASIZE = NMETA * 4; // size in bytes
}
```
### 2.3.2 init()
```java
public void init(MapOutputCollector.Context context
                    ) throws IOException, ClassNotFoundException {
      job = context.getJobConf();
      reporter = context.getReporter();
      mapTask = context.getMapTask();
      mapOutputFile = mapTask.getMapOutputFile();
      sortPhase = mapTask.getSortPhase();
      spilledRecordsCounter = reporter.getCounter(TaskCounter.SPILLED_RECORDS);
      partitions = job.getNumReduceTasks();
      rfs = ((LocalFileSystem)FileSystem.getLocal(job)).getRaw();

      //sanity checks
      // 溢写百分比，默认值0.8
      // String MAP_SORT_SPILL_PERCENT = "mapreduce.map.sort.spill.percent";
      final float spillper =
        job.getFloat(JobContext.MAP_SORT_SPILL_PERCENT, (float)0.8);
      // 缓冲区大小
      // String IO_SORT_MB = "mapreduce.task.io.sort.mb";
      // int DEFAULT_IO_SORT_MB = 100
      final int sortmb = job.getInt(MRJobConfig.IO_SORT_MB,
          MRJobConfig.DEFAULT_IO_SORT_MB);
      indexCacheMemoryLimit = job.getInt(JobContext.INDEX_CACHE_MEMORY_LIMIT,
                                         INDEX_CACHE_MEMORY_LIMIT_DEFAULT);
      ...
      // getClass(String name,Class<? extends U> defaultValue, Class<U> xface) 判断defaultValue是否为xface的子类或实现类
      // 创建快排实例对象
      sorter = ReflectionUtils.newInstance(job.getClass(
                   MRJobConfig.MAP_SORT_CLASS, QuickSort.class,
                   IndexedSorter.class), job);
      // buffers and accounting
      // mb * 1024 * 1024 转为byte
      int maxMemUsage = sortmb << 20;
      // maxMemUsage % METASIZE整除的话整个maxMemUsage都可以用来存储数据
      maxMemUsage -= maxMemUsage % METASIZE;
      kvbuffer = new byte[maxMemUsage];
      bufvoid = kvbuffer.length;
      // 将kvbuffer封装成IntBuffer
      kvmeta = ByteBuffer.wrap(kvbuffer)
         .order(ByteOrder.nativeOrder())
         .asIntBuffer();
      // 设置Equator的值
      setEquator(0);
      // 初始化参数
      bufstart = bufend = bufindex = equator;
      kvstart = kvend = kvindex;
	  // 最大存放kvmeta的个数
      maxRec = kvmeta.capacity() / NMETA;
      // buffer溢写的阈值
      softLimit = (int)(kvbuffer.length * spillper);
      bufferRemaining = softLimit;

      // compression
      if (job.getCompressMapOutput()) {
        Class<? extends CompressionCodec> codecClass =
          job.getMapOutputCompressorClass(DefaultCodec.class);
        codec = ReflectionUtils.newInstance(codecClass, job);
      } else {
        codec = null;
      }

      // combiner
      
      spillInProgress = false;
      minSpillsForCombine = job.getInt(JobContext.MAP_COMBINE_MIN_SPILLS, 3);
      spillThread.setDaemon(true);
      spillThread.setName("SpillThread");
      spillLock.lock();
      try {
        // 启动spill的线程  
        spillThread.start();
        while (!spillThreadRunning) {
          spillDone.await();
        }
}

private void setEquator(int pos) {
      equator = pos;
      // set index prior to first entry, aligned at meta boundary
      // 第一个entry的末尾位置，meta和kv数据的分界线
      final int aligned = pos - (pos % METASIZE);
      // Cast one of the operands to long to avoid integer overflow
      // meta中存放数据的起始位置(逆时针)
      kvindex = (int)
        (((long)aligned - METASIZE + kvbuffer.length) % kvbuffer.length) / 4;
      LOG.info("(EQUATOR) " + pos + " kvi " + kvindex +
          "(" + (kvindex * 4) + ")");
    }
```
- 初始化后的 Buffer
![[MapOutBuffer 1.svg]]
### 2.3.3 collect()
```java
// MapOutputBuffer实现了MapOutputCollector，重写了collect方法
public synchronized void collect(K key, V value, final int partition
                                     ) throws IOException {
      reporter.progress();
      ...
      // bufferRemaining = softLimit
      // 减去元数据长度(16byte)，未达到softLimit不进行下面的逻辑
      bufferRemaining -= METASIZE;
      if (bufferRemaining <= 0) {
        // start spill if the thread is not running and the soft limit has been
        // reached
        spillLock.lock();
        try {
          do {
            // init时spillInProgress默认为false
            if (!spillInProgress) {
              // 转byte
              final int kvbidx = 4 * kvindex;
              final int kvbend = 4 * kvend;
              // serialized, unspilled bytes always lie between kvindex and
              // bufindex, crossing the equator. Note that any void space
              // created by a reset must be included in "used" bytes
              final int bUsed = distanceTo(kvbidx, bufindex);
              final boolean bufsoftlimit = bUsed >= softLimit;
              ...
              } else if (bufsoftlimit && kvindex != kvend) {
                // spill records, if any collected; check latter, as it may
                // be possible for metadata alignment to hit spill pcnt
                startSpill();
                final int avgRec = (int)
                  (mapOutputByteCounter.getCounter() /
                  mapOutputRecordCounter.getCounter());
                // leave at least half the split buffer for serialization data
                // ensure that kvindex >= bufindex
                final int distkvi = distanceTo(bufindex, kvbidx);
                final int newPos = (bufindex +
                  Math.max(2 * METASIZE - 1,
                          Math.min(distkvi / 2,
                                   distkvi / (METASIZE + avgRec) * METASIZE)))
                  % kvbuffer.length;
                setEquator(newPos);
                bufmark = bufindex = newPos;
                final int serBound = 4 * kvend;
                // bytes remaining before the lock must be held and limits
                // checked is the minimum of three arcs: the metadata space, the
                // serialization space, and the soft limit
                bufferRemaining = Math.min(
                    // metadata max
                    distanceTo(bufend, newPos),
                    Math.min(
                      // serialization max
                      distanceTo(newPos, serBound),
                      // soft limit
                      softLimit)) - 2 * METASIZE;
              }
            }
          } while (false);
        } finally {
          spillLock.unlock();
        }
      }

      try {
        // serialize key bytes into buffer
        int keystart = bufindex;
        // 通过WritableSerializer对kv进行序列化
        keySerializer.serialize(key);
        // 如果上次write，key有一部分是重置bufindex后写入的，此时bufindex < keystart
        if (bufindex < keystart) {
          // wrapped the key; must make contiguous
          bb.shiftBufferedKey();
          keystart = 0;
        }
        // serialize value bytes into buffer
        final int valstart = bufindex;
        valSerializer.serialize(value);

        bb.write(b0, 0, 0);
        // 更新valend
        int valend = bb.markRecord();

        mapOutputRecordCounter.increment(1);
        mapOutputByteCounter.increment(
            distanceTo(keystart, valend, bufvoid));

        // 更新元数据
        kvmeta.put(kvindex + PARTITION, partition);
        kvmeta.put(kvindex + KEYSTART, keystart);
        kvmeta.put(kvindex + VALSTART, valstart);
        kvmeta.put(kvindex + VALLEN, distanceTo(valstart, valend));
        // advance kvindex
        kvindex = (kvindex - NMETA + kvmeta.capacity()) % kvmeta.capacity();
      } catch (MapBufferTooSmallException e) {
        LOG.info("Record too large for in-memory buffer: " + e.getMessage());
        spillSingleRecord(key, value, partition);
        mapOutputRecordCounter.increment(1);
        return;
      }
}
```
### 2.3.4 serialize()
```java
// 通过WritableSerialization的静态内部类WritableSerializer进行序列化
@Override
	// w是需要序列化的Writable对象，自定义的类需要序列化时会实现Writable接口通过write方法序列化
    public void serialize(Writable w) throws IOException {
      w.write(dataOut);
}

// k为Text类型时
public void write(DataOutput out) throws IOException {
    WritableUtils.writeVInt(out, length);
    out.write(bytes, 0, length);
}

// DataOutputStream.java
public synchronized void write(byte[] b, int off, int len)
    throws IOException
{
    out.write(b, off, len);
    incCount(len);
}

// 最后调用MapOutputBuffer.Buffer.write
public void write(byte b[], int off, int len)
          throws IOException {
        // must always verify the invariant that at least METASIZE bytes are
        // available beyond kvindex, even when len == 0
    	// buffer长度减少，同样第一次不进入下面的逻辑
        bufferRemaining -= len;
        if (bufferRemaining <= 0) {
          // bufferRemaining<=0需要溢写
          // writing these bytes could exhaust available buffer space or fill
          // the buffer to soft limit. check if spill or blocking are necessary
          boolean blockwrite = false;
          spillLock.lock();
          try {
            do {
              checkSpillException();
			  
              // 计算byte位置
              final int kvbidx = 4 * kvindex;
              final int kvbend = 4 * kvend;
              // ser distance to key index
              final int distkvi = distanceTo(bufindex, kvbidx);
              // ser distance to spill end index
              final int distkve = distanceTo(bufindex, kvbend);
              blockwrite = distkvi <= distkve
                ? distkvi <= len + 2 * METASIZE
                : distkve <= len || distanceTo(bufend, kvbidx) < 2 * METASIZE;

              if (!spillInProgress) {
                if (blockwrite) {
                  if ((kvbend + METASIZE) % kvbuffer.length !=
                      equator - (equator % METASIZE)) {
                    // spill finished, reclaim space
                    // need to use meta exclusively; zero-len rec & 100% spill
                    // pcnt would fail
                    resetSpill(); // resetSpill doesn't move bufindex, kvindex
                    bufferRemaining = Math.min(
                        distkvi - 2 * METASIZE,
                        softLimit - distanceTo(kvbidx, bufindex)) - len;
                    continue;
                  }
                  // we have records we can spill; only spill if blocked
                  if (kvindex != kvend) {
                    startSpill();
                    // Blocked on this write, waiting for the spill just
                    // initiated to finish. Instead of repositioning the marker
                    // and copying the partial record, we set the record start
                    // to be the new equator
                    setEquator(bufmark);
                  } else {
                    // We have no buffered records, and this record is too large
                    // to write into kvbuffer. We must spill it directly from
                    // collect
                    final int size = distanceTo(bufstart, bufindex) + len;
                    setEquator(0);
                    bufstart = bufend = bufindex = equator;
                    kvstart = kvend = kvindex;
                    bufvoid = kvbuffer.length;
                    throw new MapBufferTooSmallException(size + " bytes");
                  }
                }
              }

              if (blockwrite) {
                // wait for spill
            } while (blockwrite);
          } finally {
            spillLock.unlock();
          }
        }
        // here, we know that we have sufficient space to write
    	// 在溢写时，同时向缓冲区尾部写入数据时，如果key的长度超过了buffer的长度
    	// 从bufindex处先将buffer写满(gaplen)
        // len减去gaplen，off增加gaplen，将bufindex重置为0后，再从头写入剩余数据
        if (bufindex + len > bufvoid) {
          final int gaplen = bufvoid - bufindex;
          System.arraycopy(b, off, kvbuffer, bufindex, gaplen);
          len -= gaplen;
          off += gaplen;
          bufindex = 0;
        }
    	// 向kvbuffer中bufindex的位置写入b中off处长度为len的数据
        System.arraycopy(b, off, kvbuffer, bufindex, len);
    	// buffindex移动
        bufindex += len;
}
```
### 2.3.5 写入 kv 序列化数据后的操作
```java
// MapOutputBuffer定义了属性，内部类BlockingBuffer:
final BlockingBuffer bb = new BlockingBuffer();
// 更新valend，此时还没写入meta
// 将写完后的bufindex赋给bufmark并返回bufindex
public int markRecord() {
        bufmark = bufindex;
        return bufindex;
}
// 再更新元数据，移动kvindex
```

![[MapOutBufferInit 1.svg]]

### 2.3.6 shiftBufferedKey
```java
protected void shiftBufferedKey() throws IOException {
        // spillLock unnecessary; both kvend and kvindex are current
        int headbytelen = bufvoid - bufmark;
        bufvoid = bufmark;
        final int kvbidx = 4 * kvindex;
        final int kvbend = 4 * kvend;
        final int avail =
          Math.min(distanceTo(0, kvbidx), distanceTo(0, kvbend));
        // 判断空间是否足够
        if (bufindex + headbytelen < avail) {
          // 空间足够，复制两次，移动bufindex，调整bufferRemaining
          // 先复制尾部
          System.arraycopy(kvbuffer, 0, kvbuffer, headbytelen, bufindex);
          // 再复制头部
          System.arraycopy(kvbuffer, bufvoid, kvbuffer, 0, headbytelen);
          bufindex += headbytelen;
          bufferRemaining -= kvbuffer.length - bufvoid;
        } else {
          // 空间不够
          byte[] keytmp = new byte[bufindex];
          System.arraycopy(kvbuffer, 0, keytmp, 0, bufindex);
          bufindex = 0;
          out.write(kvbuffer, bufmark, headbytelen);
          out.write(keytmp);
        }
}
```
### 2.3.7 写元数据时 spill
```java
// 缓冲区在初始化时会启动spill线程
public void run() {
        spillLock.lock();
        spillThreadRunning = true;
        try {
          while (true) {
            spillDone.signal();
            while (!spillInProgress) {
              spillReady.await();
            }
            try {
              spillLock.unlock();
              sortAndSpill();
            }
          }
        }
}

private void startSpill() {
      assert !spillInProgress;
      kvend = (kvindex + NMETA) % kvmeta.capacity();
      bufend = bufmark;
      spillInProgress = true;
      LOG.info("Spilling map output");
      LOG.info("bufstart = " + bufstart + "; bufend = " + bufmark +
               "; bufvoid = " + bufvoid);
      LOG.info("kvstart = " + kvstart + "(" + (kvstart * 4) +
               "); kvend = " + kvend + "(" + (kvend * 4) +
               "); length = " + (distanceTo(kvend, kvstart,
                     kvmeta.capacity()) + 1) + "/" + maxRec);
      // 唤醒线程
      spillReady.signal();
}

private void sortAndSpill() throws IOException, ClassNotFoundException,
                                       InterruptedException {
      //approximate the length of the output file to be the length of the
      //buffer + header lengths for the partitions
      final long size = distanceTo(bufstart, bufend, bufvoid) +
                  partitions * APPROX_HEADER_LENGTH;
      FSDataOutputStream out = null;
      FSDataOutputStream partitionOut = null;
      try {
        // create spill file
        final SpillRecord spillRec = new SpillRecord(partitions);
        final Path filename =
            mapOutputFile.getSpillFileForWrite(numSpills, size);
        out = rfs.create(filename);
        // 根据K排序
        sorter.sort(MapOutputBuffer.this, mstart, mend, reporter);
        int spindex = mstart;
        final IndexRecord rec = new IndexRecord();
        final InMemValBytes value = new InMemValBytes();
        for (int i = 0; i < partitions; ++i) {
          IFile.Writer<K, V> writer = null;
          try {
            long segmentStart = out.getPos();
            partitionOut =
                IntermediateEncryptedStream.wrapIfNecessary(job, out, false,
                    filename);
            // 创建writer
            writer = new Writer<K, V>(job, partitionOut, keyClass, valClass, codec,
                                      spilledRecordsCounter);
            if (combinerRunner == null) {
              // no combiner, spill directly
              DataInputBuffer key = new DataInputBuffer();
              while (spindex < mend &&
                  kvmeta.get(offsetFor(spindex % maxRec) + PARTITION) == i) {
                final int kvoff = offsetFor(spindex % maxRec);
                int keystart = kvmeta.get(kvoff + KEYSTART);
                int valstart = kvmeta.get(kvoff + VALSTART);
                key.reset(kvbuffer, keystart, valstart - keystart);
                getVBytesForOffset(kvoff, value);
                // 通过append写  
                writer.append(key, value);
                ++spindex;
              }
            } else {...}

            // close the writer
            // record offsets
            rec.startOffset = segmentStart;
            rec.rawLength = writer.getRawLength() + CryptoUtils.cryptoPadding(job);
            rec.partLength = writer.getCompressedLength() + CryptoUtils.cryptoPadding(job);
            spillRec.putIndex(rec, i);
            ...
        }

        if (totalIndexCacheMemory >= indexCacheMemoryLimit) {
          // create spill index file
          Path indexFilename =
              mapOutputFile.getSpillIndexFileForWrite(numSpills, partitions
                  * MAP_OUTPUT_INDEX_RECORD_LENGTH);
          IntermediateEncryptedStream.addSpillIndexFile(indexFilename, job);
          // 写磁盘
          spillRec.writeToFile(indexFilename, job);
        } else {...}
        LOG.info("Finished spill " + numSpills);
        ++numSpills;
      }
    }
}

// Writer是IFile的内部静态类
// IFile.java
public void append(DataInputBuffer key, DataInputBuffer value)
    throws IOException {
      int keyLength = key.getLength() - key.getPosition();
      if (keyLength < 0) {
        throw new IOException("Negative key-length not allowed: " + keyLength + 
                              " for " + key);
      }
      
      int valueLength = value.getLength() - value.getPosition();
      if (valueLength < 0) {
        throw new IOException("Negative value-length not allowed: " + 
                              valueLength + " for " + value);
      }

      WritableUtils.writeVInt(out, keyLength);
      WritableUtils.writeVInt(out, valueLength);
      // 向FSDataOutputStream中写数据
      out.write(key.getData(), key.getPosition(), keyLength); 
      out.write(value.getData(), value.getPosition(), valueLength); 

      // Update bytes written
      decompressedBytesWritten += keyLength + valueLength + 
                      WritableUtils.getVIntSize(keyLength) + 
                      WritableUtils.getVIntSize(valueLength);
      ++numRecordsWritten;
}

// SpillRecord.java写磁盘
// 用crc做校验
public void writeToFile(Path loc, JobConf job)
      throws IOException {
    writeToFile(loc, job, new PureJavaCrc32());
}

public void writeToFile(Path loc, JobConf job, Checksum crc)
      throws IOException {
    final FileSystem rfs = FileSystem.getLocal(job).getRaw();
    CheckedOutputStream chk = null;
    final FSDataOutputStream out = rfs.create(loc);
    try {
      if (crc != null) {
        crc.reset();
        chk = new CheckedOutputStream(out, crc);
        // 写crc文件
        chk.write(buf.array());
        // spill文件
        out.writeLong(chk.getChecksum().getValue());
      } else {
        out.write(buf.array());
      }
    } finally {
      if (chk != null) {
        chk.close();
      } else {
        out.close();
      }
    }
}
```
### 2.3.8 写 kv 数据时的 spill
```java
// 在write方法中，当bufferRemaining不够写时，开始spill
public void write(byte b[], int off, int len)
          throws IOException {
        bufferRemaining -= len;
        if (bufferRemaining <= 0) {
          boolean blockwrite = false;
          spillLock.lock();
          try {
            do {
            ...
              if (!spillInProgress) {
                // spill时需要阻塞写
                if (blockwrite) {
                  ...
                  if (kvindex != kvend) {
                    startSpill();
                    // Blocked on this write, waiting for the spill just
                    // initiated to finish. Instead of repositioning the marker
                    // and copying the partial record, we set the record start
                    // to be the new equator
                    setEquator(bufmark);
                  } else {
                    // 数据过大，直接抛MapBufferTooSmallException异常，并重置buffer的参数
                    // We have no buffered records, and this record is too large
                    // to write into kvbuffer. We must spill it directly from
                    // collect
                    final int size = distanceTo(bufstart, bufindex) + len;
                    setEquator(0);
                    bufstart = bufend = bufindex = equator;
                    kvstart = kvend = kvindex;
                    bufvoid = kvbuffer.length;
                    throw new MapBufferTooSmallException(size + " bytes");
                  }
                }
              }
			...
}
              
// collect()中catch到MapBufferTooSmallException异常，通过spillSingleRecord将整个文件溢写到磁盘
// spillSingleRecord不排序，不循环处理多个spill文件
catch (MapBufferTooSmallException e) {
        LOG.info("Record too large for in-memory buffer: " + e.getMessage());
        spillSingleRecord(key, value, partition);
        mapOutputRecordCounter.increment(1);
        return;
}
```
### 2.3.9 close()
```java
// NewOutputCollector是MapTask的私有内部类
// 写完Buffer后2.2.1.2中runNewMapper继续执行NewOutputCollector.close(mapperContext)
output.close(mapperContext);

public void close(TaskAttemptContext context
                      ) throws IOException,InterruptedException {
      try {
        collector.flush();
      } catch (ClassNotFoundException cnf) {
        throw new IOException("can't find class ", cnf);
      }
      collector.close();
}

public void flush() throws IOException, ClassNotFoundException,
           InterruptedException {
      // 没有spill结束时再次spill    
      // release sort buffer before the merge
      kvbuffer = null;
      // 将spill文件合并成大文件
      mergeParts();
      Path outputPath = mapOutputFile.getOutputFile();
}
```
### 2.3.10 合并切片文件
```java
private void mergeParts() throws IOException, InterruptedException, 
                                     ClassNotFoundException {
      // get the approximate size of the final output/index files
      long finalOutFileSize = 0;
      long finalIndexFileSize = 0;
      final Path[] filename = new Path[numSpills];
      final TaskAttemptID mapId = getTaskID();

      for(int i = 0; i < numSpills; i++) {
        filename[i] = mapOutputFile.getSpillFile(i);
        finalOutFileSize += rfs.getFileStatus(filename[i]).getLen();
      }
      // 只有一个切片，不用分区直接产生最终输出                                   
      if (numSpills == 1) { //the spill is the final output
        Path indexFileOutput =
            mapOutputFile.getOutputIndexFileForWriteInVolume(filename[0]);
        sameVolRename(filename[0],
            mapOutputFile.getOutputFileForWriteInVolume(filename[0]));
        if (indexCacheList.size() == 0) {
          Path indexFilePath = mapOutputFile.getSpillIndexFile(0);
          IntermediateEncryptedStream.validateSpillIndexFile(
              indexFilePath, job);
          sameVolRename(indexFilePath, indexFileOutput);
        } else {
          indexCacheList.get(0).writeToFile(indexFileOutput, job);
        }
        IntermediateEncryptedStream.addSpillIndexFile(indexFileOutput, job);
        sortPhase.complete();
        return;
      }
		
      // 遍历添加到indexCacheList
      for (int i = indexCacheList.size(); i < numSpills; ++i) {
        Path indexFileName = mapOutputFile.getSpillIndexFile(i);
        IntermediateEncryptedStream.validateSpillIndexFile(indexFileName, job);
        indexCacheList.add(new SpillRecord(indexFileName, job));
      }
	  ...
      if (numSpills == 0) {...}
      {
        sortPhase.addPhases(partitions); // Divide sort phase into sub-phases
        
        IndexRecord rec = new IndexRecord();
        final SpillRecord spillRec = new SpillRecord(partitions);
        // 遍历分区
        for (int parts = 0; parts < partitions; parts++) {
          //create the segments to be merged
          List<Segment<K,V>> segmentList =
            new ArrayList<Segment<K, V>>(numSpills);
          for(int i = 0; i < numSpills; i++) {
            IndexRecord indexRecord = indexCacheList.get(i).getIndex(parts);

            Segment<K,V> s =
              new Segment<K,V>(job, rfs, filename[i], indexRecord.startOffset,
                               indexRecord.partLength, codec, true);
            segmentList.add(i, s);

            if (LOG.isDebugEnabled()) {
              LOG.debug("MapId=" + mapId + " Reducer=" + parts +
                  "Spill =" + i + "(" + indexRecord.startOffset + "," +
                  indexRecord.rawLength + ", " + indexRecord.partLength + ")");
            }
          }

          int mergeFactor = job.getInt(MRJobConfig.IO_SORT_FACTOR,
              MRJobConfig.DEFAULT_IO_SORT_FACTOR);
          // sort the segments only if there are intermediate merges
          boolean sortSegments = segmentList.size() > mergeFactor;
          // 按分区合并
          @SuppressWarnings("unchecked")
          RawKeyValueIterator kvIter = Merger.merge(job, rfs,
                         keyClass, valClass, codec,
                         segmentList, mergeFactor,
                         new Path(mapId.toString()),
                         job.getOutputKeyComparator(), reporter, sortSegments,
                         null, spilledRecordsCounter, sortPhase.phase(),
                         TaskType.MAP);

          //write merged output to disk
          long segmentStart = finalOut.getPos();
          finalPartitionOut = IntermediateEncryptedStream.wrapIfNecessary(job,
              finalOut, false, finalOutputFile);
          Writer<K, V> writer =
              new Writer<K, V>(job, finalPartitionOut, keyClass, valClass, codec,
                               spilledRecordsCounter);
          // 没有combiner
          if (combinerRunner == null || numSpills < minSpillsForCombine) {
            Merger.writeFile(kvIter, writer, reporter, job);
          } else {
            combineCollector.setWriter(writer);
            combinerRunner.combine(kvIter, combineCollector);
          }

          //close
          writer.close();
          if (finalPartitionOut != finalOut) {
            finalPartitionOut.close();
            finalPartitionOut = null;
          }

          sortPhase.startNextPhase();
      }
}

public static <K extends Object, V extends Object>
  void writeFile(RawKeyValueIterator records, Writer<K, V> writer, 
                 Progressable progressable, Configuration conf) 
  throws IOException {
    long progressBar = conf.getLong(JobContext.RECORDS_BEFORE_PROGRESS,
        10000);
    long recordCtr = 0;
    while(records.next()) {
      writer.append(records.getKey(), records.getValue());
      
      if (((recordCtr++) % progressBar) == 0) {
        progressable.progress();
      }
    }
}
```
## 2.4 Reduce
- Reduce 分为 copy，merge，和 reduce
### 2.4.1 run()
```java
// 与1.12中的步骤类似，通过LocalRunner创建ReduceTask对象
public void run() {
        try {
          TaskAttemptID reduceId = new TaskAttemptID(new TaskID(
              jobId, TaskType.REDUCE, taskId), 0);
          LOG.info("Starting task: " + reduceId);

          ReduceTask reduce = new ReduceTask(systemJobFile.toString(),
              reduceId, taskId, mapIds.size(), 1);
          if (!Job.this.isInterrupted()) {
            try {
              reduce_tasks.getAndIncrement();
              myMetrics.launchReduce(reduce.getTaskID());
              // 执行run
              reduce.run(localConf, Job.this);
              myMetrics.completeReduce(reduce.getTaskID());
            } 
        }
}

public void run(JobConf job, final TaskUmbilicalProtocol umbilical)
    throws IOException, InterruptedException, ClassNotFoundException {
    job.setBoolean(JobContext.SKIP_RECORDS, isSkipping());

    if (isMapOrReduce()) {
      copyPhase = getProgress().addPhase("copy");
      sortPhase  = getProgress().addPhase("sort");
      reducePhase = getProgress().addPhase("reduce");
    }
    // start thread that will handle communication with parent
    TaskReporter reporter = startReporter(umbilical);
    
    boolean useNewApi = job.getUseNewReducer();
    // 与Map类似，设置输出类为TextOutputFormat
    initialize(job, getJobID(), reporter, useNewApi);

    // Initialize the codec
    codec = initCodec();
    RawKeyValueIterator rIter = null;
    ShuffleConsumerPlugin shuffleConsumerPlugin = null;
    
    Class combinerClass = conf.getCombinerClass();
    CombineOutputCollector combineCollector = 
      (null != combinerClass) ? 
     new CombineOutputCollector(reduceCombineOutputCounter, reporter, conf) : null;
	// clazz = Shuffle.class
    Class<? extends ShuffleConsumerPlugin> clazz =
          job.getClass(MRConfig.SHUFFLE_CONSUMER_PLUGIN, Shuffle.class, ShuffleConsumerPlugin.class);
	 
    shuffleConsumerPlugin = ReflectionUtils.newInstance(clazz, job);
    LOG.info("Using ShuffleConsumerPlugin: " + shuffleConsumerPlugin);
	// 创建shuffleContext
    ShuffleConsumerPlugin.Context shuffleContext = 
      new ShuffleConsumerPlugin.Context(getTaskID(), job, FileSystem.getLocal(job), umbilical, 
                  super.lDirAlloc, reporter, codec, 
                  combinerClass, combineCollector, 
                  spilledRecordsCounter, reduceCombineInputCounter,
                  shuffledMapsCounter,
                  reduceShuffleBytes, failedShuffleCounter,
                  mergedMapOutputsCounter,
                  taskStatus, copyPhase, sortPhase, this,
                  mapOutputFile, localMapFiles);
    // 初始化Shuffle
    shuffleConsumerPlugin.init(shuffleContext);
	// 启动Shuffle
    rIter = shuffleConsumerPlugin.run();

    // free up the data structures
    mapOutputFilesOnDisk.clear();
    
    sortPhase.complete(); // sort is complete
    setPhase(TaskStatus.Phase.REDUCE); 
    statusUpdate(umbilical);
    Class keyClass = job.getMapOutputKeyClass();
    Class valueClass = job.getMapOutputValueClass();
    RawComparator comparator = job.getOutputValueGroupingComparator();

    if (useNewApi) {
      runNewReducer(job, umbilical, reporter, rIter, comparator, 
                    keyClass, valueClass);
    } else {
      runOldReducer(job, umbilical, reporter, rIter, comparator, 
                    keyClass, valueClass);
    }
}
```
### 2.4.2 Shuffle.init()
```java
public void init(ShuffleConsumerPlugin.Context context) {
    this.context = context;

    this.reduceId = context.getReduceId();
    this.jobConf = context.getJobConf();
    this.umbilical = context.getUmbilical();
    this.reporter = context.getReporter();
    this.metrics = ShuffleClientMetrics.create(context.getReduceId(),
        this.jobConf);
    this.copyPhase = context.getCopyPhase();
    this.taskStatus = context.getStatus();
    this.reduceTask = context.getReduceTask();
    this.localMapFiles = context.getLocalMapFiles();
    
    scheduler = new ShuffleSchedulerImpl<K, V>(jobConf, taskStatus, reduceId,
        this, copyPhase, context.getShuffledMapsCounter(),
        context.getReduceShuffleBytes(), context.getFailedShuffleCounter());
    // 创建MergeManager, Merger用于溢写数据
    merger = createMergeManager(context);
}

protected MergeManager<K, V> createMergeManager(
      ShuffleConsumerPlugin.Context context) {
    return new MergeManagerImpl<K, V>(...);
}

public MergeManagerImpl(...) {

    // maxMemory JVM能使用的最大内存
    this.memoryLimit = (long)(jobConf.getLong(
        MRJobConfig.REDUCE_MEMORY_TOTAL_BYTES,
        Runtime.getRuntime().maxMemory()) * maxInMemCopyUse);
	
    // int DEFAULT_IO_SORT_FACTOR = 10;
    this.ioSortFactor = jobConf.getInt(MRJobConfig.IO_SORT_FACTOR,
        MRJobConfig.DEFAULT_IO_SORT_FACTOR);

    final float singleShuffleMemoryLimitPercent =
        // float DEFAULT_SHUFFLE_MEMORY_LIMIT_PERCENT = 0.25f
        jobConf.getFloat(MRJobConfig.SHUFFLE_MEMORY_LIMIT_PERCENT,
            DEFAULT_SHUFFLE_MEMORY_LIMIT_PERCENT);
    this.mergeThreshold = (long)(this.memoryLimit * 
                          jobConf.getFloat(
                            MRJobConfig.SHUFFLE_MERGE_PERCENT,
                            // float DEFAULT_SHUFFLE_MERGE_PERCENT = 0.66f;
                            MRJobConfig.DEFAULT_SHUFFLE_MERGE_PERCENT));

    // 创建inMemoryMerger和onDiskMerger线程并启动
    this.inMemoryMerger = createInMemoryMerger();
    this.inMemoryMerger.start();
    
    this.onDiskMerger = new OnDiskMerger(this);
    this.onDiskMerger.start();
    
    this.mergePhase = mergePhase;
}

protected MergeThread<InMemoryMapOutput<K,V>, K,V> createInMemoryMerger() {
    return new InMemoryMerger(this);
}

  protected MergeThread<CompressAwarePath,K,V> createOnDiskMerger() {
    return new OnDiskMerger(this);
}

// 守护线程
public InMemoryMerger(MergeManagerImpl<K, V> manager) {
      super(manager, Integer.MAX_VALUE, exceptionReporter);
      setName
      ("InMemoryMerger - Thread to merge in-memory shuffled map-outputs");
      setDaemon(true);
}

public OnDiskMerger(MergeManagerImpl<K, V> manager) {
      super(manager, ioSortFactor, exceptionReporter);
      setName("OnDiskMerger - Thread to merge on-disk map-outputs");
      setDaemon(true);
}
```
### 2.4.3 Shuffle.run()
```java
public RawKeyValueIterator run() throws IOException, InterruptedException {
	// int MIN_EVENTS_TO_FETCH = 100;
    int eventsPerReducer = Math.max(MIN_EVENTS_TO_FETCH,
        MAX_RPC_OUTSTANDING_EVENTS / jobConf.getNumReduceTasks());
    // int MAX_EVENTS_TO_FETCH = 10000;
    int maxEventsToFetch = Math.min(MAX_EVENTS_TO_FETCH, eventsPerReducer);

    // Start the map-completion events fetcher thread
    // 通过EventFetcher获取已完成的mapEvent信息
    final EventFetcher<K, V> eventFetcher =
        new EventFetcher<K, V>(reduceId, umbilical, scheduler, this,
            maxEventsToFetch);
    eventFetcher.start();
    
    // Start the map-output fetcher threads
    boolean isLocal = localMapFiles != null;
    // shuffle默认并行度5
    final int numFetchers = isLocal ? 1 :
        jobConf.getInt(MRJobConfig.SHUFFLE_PARALLEL_COPIES, 5);
    Fetcher<K, V>[] fetchers = new Fetcher[numFetchers];
    if (isLocal) {
      // 开启LocalFetcher线程取数据
      fetchers[0] = new LocalFetcher<K, V>(jobConf, reduceId, scheduler,
          merger, reporter, metrics, this, reduceTask.getShuffleSecret(),
          localMapFiles);
      fetchers[0].start();
    } else {...}
    
    // stop the scheduler
    scheduler.close();

    copyPhase.complete(); // copy is already complete
    taskStatus.setPhase(TaskStatus.Phase.SORT);
    reduceTask.statusUpdate(umbilical);

    // Finish the on-going merges...
    RawKeyValueIterator kvIter = null;
    try {
      // 最终合并
      kvIter = merger.close();
    } 
    return kvIter;
}
```
#### 2.4.3.1 EventFetcher.run()
```java
public void run() {
    int failures = 0;
    LOG.info(reduce + " Thread started: " + getName());
    
    try {
      while (!stopped && !Thread.currentThread().isInterrupted()) {
        try {
          // 获取MapCompletionEvents
          int numNewMaps = getMapCompletionEvents();
}
    
// 向TaskTracker查询已经完成的maptask        
protected int getMapCompletionEvents()
      throws IOException, InterruptedException {
    
    int numNewMaps = 0;
    TaskCompletionEvent events[] = null;

    do {
      MapTaskCompletionEventsUpdate update =
          umbilical.getMapCompletionEvents(
              (org.apache.hadoop.mapred.JobID)reduce.getJobID(),
              fromEventIdx,
              maxEventsToFetch,
              (org.apache.hadoop.mapred.TaskAttemptID)reduce);
      events = update.getMapTaskCompletionEvents();
      LOG.debug("Got " + events.length + " map completion events from " +
               fromEventIdx);

      assert !update.shouldReset() : "Unexpected legacy state";

      // Update the last seen event ID
      fromEventIdx += events.length;

      // Process the TaskCompletionEvents:
      // 1. Save the SUCCEEDED maps in knownOutputs to fetch the outputs.
      // 2. Save the OBSOLETE/FAILED/KILLED maps in obsoleteOutputs to stop
      //    fetching from those maps.
      // 3. Remove TIPFAILED maps from neededOutputs since we don't need their
      //    outputs at all.
      for (TaskCompletionEvent event : events) {
        scheduler.resolve(event);
        if (TaskCompletionEvent.Status.SUCCEEDED == event.getTaskStatus()) {
          ++numNewMaps;
        }
      }
    } while (events.length == maxEventsToFetch);

    return numNewMaps;
}
```
#### 2.4.3.2 new LocalFetcher()
```java
// local模式下，通过LocalFetcher抓取数据
public LocalFetcher(JobConf job, TaskAttemptID reduceId,
                 ShuffleSchedulerImpl<K, V> scheduler,
                 MergeManager<K,V> merger,
                 Reporter reporter, ShuffleClientMetrics metrics,
                 ExceptionReporter exceptionReporter,
                 SecretKey shuffleKey,
                 Map<TaskAttemptID, MapOutputFile> localMapFiles) {
    super(job, reduceId, scheduler, merger, reporter, metrics,
        exceptionReporter, shuffleKey);

    this.job = job;
    this.localMapFiles = localMapFiles;

    setName("localfetcher#" + id);
    setDaemon(true);
}

public void run() {
    // Create a worklist of task attempts to work over.
    Set<TaskAttemptID> maps = new HashSet<TaskAttemptID>();
    for (TaskAttemptID map : localMapFiles.keySet()) {
      maps.add(map);
    }

    while (maps.size() > 0) {
      try {
        // If merge is on, block
        merger.waitForResource();
        metrics.threadBusy();

        // Copy as much as is possible.
        // copy阶段
        doCopy(maps);
        metrics.threadFree();
      } catch (InterruptedException ie) {
      } catch (Throwable t) {
        exceptionReporter.reportException(t);
      }
    }
}

private void doCopy(Set<TaskAttemptID> maps) throws IOException {
    Iterator<TaskAttemptID> iter = maps.iterator();
    while (iter.hasNext()) {
      TaskAttemptID map = iter.next();
      LOG.debug("LocalFetcher " + id + " going to fetch: " + map);
      if (copyMapOutput(map)) {
        // Successful copy. Remove this from our worklist.
        iter.remove();
      } else {
        // We got back a WAIT command; go back to the outer loop
        // and block for InMemoryMerge.
        break;
      }
    }
}

private boolean copyMapOutput(TaskAttemptID mapTaskId) throws IOException {
    // Figure out where the map task stored its output.
    Path mapOutputFileName = localMapFiles.get(mapTaskId).getOutputFile();
    Path indexFileName = mapOutputFileName.suffix(".index");

    // Read its index to determine the location of our split
    // and its size.
    SpillRecord sr = new SpillRecord(indexFileName, job);
    IndexRecord ir = sr.getIndex(reduce);

    long compressedLength = ir.partLength;
    long decompressedLength = ir.rawLength;

    compressedLength -= CryptoUtils.cryptoPadding(job);
    decompressedLength -= CryptoUtils.cryptoPadding(job);

    // 判断在内存还是在磁盘merge
    MapOutput<K, V> mapOutput = merger.reserve(mapTaskId, decompressedLength,
        id);

    // Check if we can shuffle *now* ...
    if (mapOutput == null) {
      LOG.info("fetcher#" + id + " - MergeManager returned Status.WAIT ...");
      return false;
    }

    // Go!
    LOG.info("localfetcher#" + id + " about to shuffle output of map " + 
             mapOutput.getMapId() + " decomp: " +
             decompressedLength + " len: " + compressedLength + " to " +
             mapOutput.getDescription());

    // now read the file, seek to the appropriate section, and send it.
    FileSystem localFs = FileSystem.getLocal(job).getRaw();
    // 通过localFs打开mapOutputFileName，用于后面写数据
    FSDataInputStream inStream = localFs.open(mapOutputFileName);
    try {
      inStream.seek(ir.startOffset);
      inStream =
          IntermediateEncryptedStream.wrapIfNecessary(job, inStream,
              mapOutputFileName);
      // 将数据存在LOCALHOST
      mapOutput.shuffle(LOCALHOST, inStream, compressedLength,
          decompressedLength, metrics, reporter);
    } finally {
      IOUtils.cleanupWithLogger(LOG, inStream);
    }

    scheduler.copySucceeded(mapTaskId, LOCALHOST, compressedLength, 0, 0,
        mapOutput);
    return true; // successful fetch.
}

public synchronized MapOutput<K,V> reserve(TaskAttemptID mapId, 
                                             long requestedSize,
                                             int fetcher
                                             ) throws IOException {
    // 超过单条shuffle大小上限
    if (requestedSize > maxSingleShuffleLimit) {
      LOG.info(mapId + ": Shuffling to disk since " + requestedSize + 
               " is greater than maxSingleShuffleLimit (" + 
               maxSingleShuffleLimit + ")");
      // 创建OnDiskMapOutput
      return new OnDiskMapOutput<K,V>(mapId, this, requestedSize, jobConf,
         fetcher, true, FileSystem.getLocal(jobConf).getRaw(),
         mapOutputFile.getInputFileForWrite(mapId.getTaskID(), requestedSize));
    }
    
    // 在内存执行后续操作
    LOG.debug(mapId + ": Proceeding with shuffle since usedMemory ("
        + usedMemory + ") is lesser than memoryLimit (" + memoryLimit + ")."
        + "CommitMemory is (" + commitMemory + ")");
    return unconditionalReserve(mapId, requestedSize, true);
}

private synchronized InMemoryMapOutput<K, V> unconditionalReserve(
      TaskAttemptID mapId, long requestedSize, boolean primaryMapOutput) {
    usedMemory += requestedSize;
    return new InMemoryMapOutput<K,V>(jobConf, mapId, this, (int)requestedSize,
                                      codec, primaryMapOutput);
}
```
### 2.4.4 IFileWrappedMapOutput.shuffle()
```java
// copyMapOutput()中
public void shuffle(MapHost host, InputStream input,
                      long compressedLength, long decompressedLength,
                      ShuffleClientMetrics metrics,
                      Reporter reporter) throws IOException {
    // 对之前的copyMapOutput()中创建的FSDataInputStream进行包装得到IFileInputStream
    doShuffle(host, new IFileInputStream(input, compressedLength, conf),
        compressedLength, decompressedLength, metrics, reporter);
}

// 根据磁盘或是内存，有两个实现类
// InMemoryMapOutput.doshuffle
protected void doShuffle(MapHost host, IFileInputStream iFin,
                      long compressedLength, long decompressedLength,
                      ShuffleClientMetrics metrics,
                      Reporter reporter) throws IOException {
    InputStream input = iFin;

    // Are map-outputs compressed?
    if (codec != null) {
      decompressor.reset();
      input = codec.createInputStream(input, decompressor);
    }
  
    try {
      // 将内存中的数据读入IFileInputStream中
      IOUtils.readFully(input, memory, 0, memory.length);
      metrics.inputBytes(memory.length);
      reporter.progress();
      LOG.info("Read " + memory.length + " bytes from map-output for " +
                getMapId());

      /**
       * We've gotten the amount of data we were expecting. Verify the
       * decompressor has nothing more to offer. This action also forces the
       * decompressor to read any trailing bytes that weren't critical
       * for decompression, which is necessary to keep the stream
       * in sync.
       */
      if (input.read() >= 0 ) {
        throw new IOException("Unexpected extra bytes from input stream for " + getMapId());
      }
    } finally {
      CodecPool.returnDecompressor(decompressor);
    }
}

// OnDiskMapOutput.doshuffle
protected void doShuffle(MapHost host, IFileInputStream input,
                      long compressedLength, long decompressedLength,
                      ShuffleClientMetrics metrics,
                      Reporter reporter) throws IOException {
    // Copy data to local-disk
    long bytesLeft = compressedLength;
    try {
      final int BYTES_TO_READ = 64 * 1024;
      byte[] buf = new byte[BYTES_TO_READ];
      while (bytesLeft > 0) {
        int n = input.readWithChecksum(buf, 0,
                                      (int) Math.min(bytesLeft, BYTES_TO_READ));
        if (n < 0) {
          throw new IOException("read past end of stream reading " + 
                                getMapId());
        }
        // 向磁盘写
        disk.write(buf, 0, n);
        bytesLeft -= n;
        metrics.inputBytes(n);
        reporter.progress();
      }

      LOG.info("Read " + (compressedLength - bytesLeft) + 
               " bytes from map-output for " + getMapId());
		
      disk.close();
    } catch (IOException ioe) {
      // Close the streams
      IOUtils.cleanupWithLogger(LOG, disk);

      // Re-throw
      throw ioe;
    }

    // Sanity check
    if (bytesLeft != 0) {
      throw new IOException("Incomplete map output received for " +
                            getMapId() + " from " +
                            host.getHostName() + " (" + 
                            bytesLeft + " bytes missing of " + 
                            compressedLength + ")");
    }
    this.compressedSize = compressedLength;
}
```
### 2.4.5 merger 线程
#### 2.4.5.1 启动
```java
// MergeThread.java
// 2.4.2中创建
// InMemoryMerger和OnDiskMerger继承了MergeThread
public void run() {
    while (true) {
      List<T> inputs = null;
      try {
        // Wait for notification to start the merge...
        // private LinkedList<List<T>> pendingToBeMerged;
        synchronized (pendingToBeMerged) {
          while(pendingToBeMerged.size() <= 0) {
            // 等待
            pendingToBeMerged.wait();
          }
          // Pickup the inputs to merge.
          inputs = pendingToBeMerged.removeFirst();
        }

        // Merge
        merge(inputs);
      } catch (InterruptedException ie) {
        numPending.set(0);
        return;
      } catch(Throwable t) {
        numPending.set(0);
        reporter.reportException(t);
        return;
      } finally {
        synchronized (this) {
          numPending.decrementAndGet();
          notifyAll();
        }
      }
    }
}
```
#### 2.4.5.2 InMemoryMerger.merge()
```java
// reduce的merge分为mem2mem(不启用), mem2disk，disk2disk
// 内存充足时，首先是mem2disk
public void merge(List<InMemoryMapOutput<K,V>> inputs) throws IOException {
        ...
        // 获取kv迭代器
        // merge的同时通过OutputKeyComparator排序
        rIter = Merger.merge(jobConf, rfs,
                             (Class<K>)jobConf.getMapOutputKeyClass(),
                             (Class<V>)jobConf.getMapOutputValueClass(),
                             inMemorySegments, inMemorySegments.size(),
                             new Path(reduceId.toString()),
                             (RawComparator<K>)jobConf.getOutputKeyComparator(),
                             reporter, spilledRecordsCounter, null, null);
        
        if (null == combinerClass) {
          // 写文件
          Merger.writeFile(rIter, writer, reporter, jobConf);
        } else {
          combineCollector.setWriter(writer);
          combineAndSpill(rIter, reduceCombineInputCounter);
        }
        writer.close();
      // 关闭磁盘文件
      closeOnDiskFile(compressAwarePath);
}
```
#### 2.4.5.3 onDiskMerger.merge()
```java
public void merge(List<CompressAwarePath> inputs) throws IOException {    
      ...
      Path outputPath = 
        localDirAllocator.getLocalPathForWrite(inputs.get(0).toString(), 
            approxOutputSize, jobConf).suffix(Task.MERGED_OUTPUT_PREFIX);

      FSDataOutputStream out =
          IntermediateEncryptedStream.wrapIfNecessary(jobConf,
              rfs.create(outputPath), outputPath);
      Writer<K, V> writer = new Writer<K, V>(jobConf, out,
          (Class<K>) jobConf.getMapOutputKeyClass(),
          (Class<V>) jobConf.getMapOutputValueClass(), codec, null, true);

      RawKeyValueIterator iter  = null;
      CompressAwarePath compressAwarePath;
      Path tmpDir = new Path(reduceId.toString());
      try {
        // merge的同时通过OutputKeyComparator排序
        iter = Merger.merge(jobConf, rfs,
                            (Class<K>) jobConf.getMapOutputKeyClass(),
                            (Class<V>) jobConf.getMapOutputValueClass(),
                            codec, inputs.toArray(new Path[inputs.size()]), 
                            true, ioSortFactor, tmpDir, 
                            (RawComparator<K>) jobConf.getOutputKeyComparator(), 
                            reporter, spilledRecordsCounter, null, 
                            mergedMapOutputsCounter, null);

        Merger.writeFile(iter, writer, reporter, jobConf);
        writer.close();
       ...
      closeOnDiskFile(compressAwarePath);
    }
```
### 2.4.6 关闭 merger，准备执行 reduce
```java
public RawKeyValueIterator close() throws Throwable {
    // Wait for on-going merges to complete
    if (memToMemMerger != null) { 
      memToMemMerger.close();
    }
    // 关闭线程
    inMemoryMerger.close();
    onDiskMerger.close();
    
    List<InMemoryMapOutput<K, V>> memory = 
      new ArrayList<InMemoryMapOutput<K, V>>(inMemoryMergedMapOutputs);
    inMemoryMergedMapOutputs.clear();
    memory.addAll(inMemoryMapOutputs);
    inMemoryMapOutputs.clear();
    List<CompressAwarePath> disk = new ArrayList<CompressAwarePath>(onDiskMapOutputs);
    onDiskMapOutputs.clear();
    return finalMerge(jobConf, rfs, memory, disk);
}

private RawKeyValueIterator finalMerge(JobConf job, FileSystem fs,
                                       List<InMemoryMapOutput<K,V>> inMemoryMapOutputs,
                                       List<CompressAwarePath> onDiskMapOutputs
                                       ) throws IOException {
    // 先处理内存中的数据
    if (inMemoryMapOutputs.size() > 0) {
        ... 
        // 
        try {
          Merger.writeFile(rIter, writer, reporter, job);
          writer.close();
          // 添加到List DiskMapOutputs中
          onDiskMapOutputs.add(new CompressAwarePath(outputPath,
              writer.getRawLength(), writer.getCompressedLength()));
          writer = null;
          // add to list of final disk outputs.
        } ...

    // 处理磁盘中的数据
    List<Segment<K,V>> diskSegments = new ArrayList<Segment<K,V>>();
    long onDiskBytes = inMemToDiskBytes;
    long rawBytes = inMemToDiskBytes;
    CompressAwarePath[] onDisk = onDiskMapOutputs.toArray(
        new CompressAwarePath[onDiskMapOutputs.size()]);
    for (CompressAwarePath file : onDisk) {
      long fileLength = fs.getFileStatus(file).getLen();
      onDiskBytes += fileLength;
      rawBytes += (file.getRawDataLength() > 0) ? file.getRawDataLength() : fileLength;

      LOG.debug("Disk file: " + file + " Length is " + fileLength);
      diskSegments.add(new Segment<K, V>(job, fs, file, codec, keepInputs,
                                         (file.toString().endsWith(
                                             Task.MERGED_OUTPUT_PREFIX) ?
                                          null : mergedMapOutputsCounter), file.getRawDataLength()
                                        ));
    }
    // 排序
    Collections.sort(diskSegments, new Comparator<Segment<K,V>>() {
      public int compare(Segment<K, V> o1, Segment<K, V> o2) {
        if (o1.getLength() == o2.getLength()) {
          return 0;
        }
        return o1.getLength() < o2.getLength() ? -1 : 1;
      }
    });

    // build final list of segments from merged backed by disk + in-mem
    List<Segment<K,V>> finalSegments = new ArrayList<Segment<K,V>>();
    long inMemBytes = createInMemorySegments(inMemoryMapOutputs, 
                                             finalSegments, 0);
    LOG.info("Merging " + finalSegments.size() + " segments, " +
             inMemBytes + " bytes from memory into reduce");
    if (0 != onDiskBytes) {...}
    // 再merge一次
    return Merger.merge(job, fs, keyClass, valueClass,
                 finalSegments, finalSegments.size(), tmpDir,
                 comparator, reporter, spilledRecordsCounter, null,
                 null);
}
// 拉取数据结束，回到ReduceTask的run()中
```
### 2.4.7 runNewReducer()
```java
private <INKEY,INVALUE,OUTKEY,OUTVALUE>
  void runNewReducer(JobConf job,
                     final TaskUmbilicalProtocol umbilical,
                     final TaskReporter reporter,
                     RawKeyValueIterator rIter,
                     RawComparator<INKEY> comparator,
                     Class<INKEY> keyClass,
                     Class<INVALUE> valueClass
                     ) throws IOException,InterruptedException, 
                              ClassNotFoundException {

    // make a task context so we can get the classes
    org.apache.hadoop.mapreduce.TaskAttemptContext taskContext =
      new org.apache.hadoop.mapreduce.task.TaskAttemptContextImpl(job,
          getTaskID(), reporter);
    // make a reducer
    // 获取自定义的reducer                              
    org.apache.hadoop.mapreduce.Reducer<INKEY,INVALUE,OUTKEY,OUTVALUE> reducer =
      (org.apache.hadoop.mapreduce.Reducer<INKEY,INVALUE,OUTKEY,OUTVALUE>)
        ReflectionUtils.newInstance(taskContext.getReducerClass(), job);
    // 指定输出时的RecordWriter                          
    org.apache.hadoop.mapreduce.RecordWriter<OUTKEY,OUTVALUE> trackedRW = 
      new NewTrackingRecordWriter<OUTKEY, OUTVALUE>(this, taskContext);
    job.setBoolean("mapred.skip.on", isSkipping());
    job.setBoolean(JobContext.SKIP_RECORDS, isSkipping());
    org.apache.hadoop.mapreduce.Reducer.Context
         // 创建上下文
         reducerContext = createReduceContext(reducer, job, getTaskID(),
                                               rIter, reduceInputKeyCounter, 
                                               reduceInputValueCounter, 
                                               trackedRW,
                                               committer,
                                               reporter, comparator, keyClass,
                                               valueClass);
    try {
      // 运行reducer  
      reducer.run(reducerContext);
    } finally {
      trackedRW.close(reducerContext);
    }
}

public void run(Context context) throws IOException, InterruptedException {
    setup(context);
    try {
      // createReduceContext创建的上下文
      while (context.nextKey()) {
        reduce(context.getCurrentKey(), context.getValues(), context);
        // If a back up store is used, reset it
        Iterator<VALUEIN> iter = context.getValues().iterator();
        if(iter instanceof ReduceContext.ValueIterator) {
          ((ReduceContext.ValueIterator<VALUEIN>)iter).resetBackupStore();        
        }
      }
    } finally {
      cleanup(context);
    }
}

NewTrackingRecordWriter(ReduceTask reduce,
        org.apache.hadoop.mapreduce.TaskAttemptContext taskContext)
        throws InterruptedException, IOException {

      long bytesOutPrev = getOutputBytes(fsStats);
      // 初始化时reduce.outputFormat是TextOutputFormat
      // 通过TextOutputFormat的getRecordWriter方法，返回一个LineRecordWriter
      this.real = (org.apache.hadoop.mapreduce.RecordWriter<K, V>) reduce.outputFormat
          .getRecordWriter(taskContext);
}
```
### 2.4.8 执行 reduce 任务
```java
// 通过reduce()中写出
context.write(key, outV);

// 与map类似，WrappedReducer中也定义了reduceContext
public void write(KEYOUT key, VALUEOUT value) throws IOException,
        InterruptedException {
      reduceContext.write(key, value);
}
//ReduceContextImpl继承了TaskInputOutputContextImpl
public void write(KEYOUT key, VALUEOUT value
                    ) throws IOException, InterruptedException {
    output.write(key, value);
}
// 类似的，调用ReduceTask的write()
public void write(K key, V value) throws IOException, InterruptedException {
      long bytesOutPrev = getOutputBytes(fsStats);
      // private final org.apache.hadoop.mapreduce.RecordWriter<K,V> real
      real.write(key,value);
      long bytesOutCurr = getOutputBytes(fsStats);
      fileOutputByteCounter.increment(bytesOutCurr - bytesOutPrev);
      outputRecordCounter.increment(1);
}
```
# 3 Yarn
- 基本架构
- `Yarn` 主要由 `ResourceManager`，`NodeManager`，`AppMaster` 以及 `Container` 组成
-  `ResourceManager` 是全局的资源管理器，负责整个系统的资源管理和分配
- 用户提交的每一个程序包含一个 `AppMaster`
	- 与 `ResourceManager` 调度器协商以获取资源，用 `Container` 表示
	- 将得到的任务进一步分配给内部的任务
	- 与 `NodeManager` 通信以启动/停止任务
	- 监控所有任务运行状态，并在任务运行失败时重新为任务申请资源以重启任务
- `NodeManager` 是每个节点上的资源和任务管理器
	- 定时地向 `ResourceManager` 汇报本节点上的资源使用情况和各个 `Container` 的运行状态
	- 接收并处理来自 `AppMaster` 的 `Container` 启动/停止等各种请求
- `Container` 是 Yarn 中的资源抽象
	- 封装了某个节点上的多维度资源，如内存、CPU、磁盘、网络等
	- `ResourceManager` 为 `AppMaster` 返回的资源用 `Container` 表示
	- `Yarn` 会为每个任务分配一个 `Container`
	- `Container` 是一个动态资源划分单位
## 3.1 启动
### 3.1.1 脚本
#### 3.1.1.1 start-yarn.sh
```shell
# 与hdfs启动类似，首先通过start-yarn.sh进行启动
#!/usr/bin/env bash
...
# start resourceManager
HARM=$("${HADOOP_HDFS_HOME}/bin/hdfs" getconf -confKey yarn.resourcemanager.ha.enabled 2>&-)
if [[ ${HARM} = "false" ]]; then
  echo "Starting resourcemanager"
  hadoop_uservar_su yarn resourcemanager "${HADOOP_YARN_HOME}/bin/yarn" \
      --config "${HADOOP_CONF_DIR}" \
      --daemon start \
      resourcemanager
  (( HADOOP_JUMBO_RETCOUNTER=HADOOP_JUMBO_RETCOUNTER + $? ))
else
  # 高可用逻辑
fi

# start nodemanager
echo "Starting nodemanagers"
hadoop_uservar_su yarn nodemanager "${HADOOP_YARN_HOME}/bin/yarn" \
    --config "${HADOOP_CONF_DIR}" \
    --workers \
    --daemon start \
    nodemanager
(( HADOOP_JUMBO_RETCOUNTER=HADOOP_JUMBO_RETCOUNTER + $? ))

# start proxyserver
```
#### 3.1.1.2 yarn
```shell
#!/usr/bin/env bash
...
## @description  Default command handler for yarn command
## @audience     public
## @stability    stable
## @replaceable  no
## @param        CLI arguments
function yarncmd_case
{
  subcmd=$1
  shift

  case ${subcmd} in
    app|application|applicationattempt|container)
      HADOOP_CLASSNAME=org.apache.hadoop.yarn.client.cli.ApplicationCLI
      set -- "${subcmd}" "$@"
      HADOOP_SUBCMD_ARGS=("$@")
      local sld="${HADOOP_YARN_HOME}/${YARN_DIR},\
${HADOOP_YARN_HOME}/${YARN_LIB_JARS_DIR},\
${HADOOP_HDFS_HOME}/${HDFS_DIR},\
${HADOOP_HDFS_HOME}/${HDFS_LIB_JARS_DIR},\
${HADOOP_COMMON_HOME}/${HADOOP_COMMON_DIR},\
${HADOOP_COMMON_HOME}/${HADOOP_COMMON_LIB_JARS_DIR}"
      hadoop_translate_cygwin_path sld
      hadoop_add_param HADOOP_OPTS service.libdir "-Dservice.libdir=${sld}"
    ;;
    # 主类
    nodemanager)
    hadoop_add_classpath 
HADOOP_CLASSNAME='org.apache.hadoop.yarn.server.nodemanager.NodeManager'
    ;;
    resourcemanager)
      hadoop_add_classpath HADOOP_CLASSNAME='org.apache.hadoop.yarn.server.resourcemanager.ResourceManager'
# 与hdfs类似，通过java启动进程
# everything is in globals at this point, so call the generic handler
hadoop_generic_java_subcmd_handler
```
### 3.1.2 ResourceManager

![[ResourceManager.svg]]

- 通信
	- `ResourceTracker`: `NM` 通过该 `RPC` 协议向 `RM` 注册，汇报节点健康状况和 `Container` 的运行状态，并获取 `RM` 的命令，`NM` 周期向 `RM` 发起请求，领取发给自己的命令
	- `ApplicationMasterProtocol`: `AppMaster` 通过该 `RPC` 协议向 `RM` 注册，申请资源和释放资源
	- `ApplicationClientProtocol`: 应用程序客户端通过该 `RPC` 协议向 `RM``提交应用程序`，查询程序状态等
- 架构
	- 交互模块
		- `RM` 针对普通用户，管理员和 web 提供了三种对外的服务 `ClientRMService`，`AdminService` 以及 `RMWebApp`
	- `NM` 管理模块
		- `NMLivelinessMonitor` 监控 `NM` 的存活状态
		- `NodesListManager` 维护正常节点和异常节点列表
		- `ResourceTrackerService` 处理来自 `NM` 的注册请求以及心跳请求
	- `AM` 管理模块
		- `AMLivelinessMonitor` 监控 `AM` 存活状态，`AM` 未汇报心跳信息超时后，上面正在运行的 `Container` 会被置为失败
		- `ApplicationMasterLuncher` 在与 `NM` 通信时启动 `AppMaster`
		- `ApplicationMasterService` 处理来自 `AppMaster` 的注册和心跳请求
	- `App` 管理模块
		- `ApplicationAClsManage`：管理应用程序的访问权限
		- `RMAppManager`：管理应用程序的启动和关闭
		- `ContainerAllocationExpirer`：当 `AM` 收到 `RM` 新分配的一个 `Container` 后，必须在一定的时间(默认 10min )内在对应的 `NM` 上启动该 `Container`，否则 `RM` 将强制回收该 `Container`，而一个已经分配的 `Container` 是否该被回收则是由 `ContainerAllocationExpirer` 决定和执行的
	- 状态机管理模块
	    `RM`**使用有限状态机**维护有状态对象的声明周期，`RM`维护了四个状态机(接口，对应的实现类为接口名+`Impl`)
		- `RMApp`：维护了 `APP` 整个运行周期
		- `RMAppAttempt`：一个实例运行失败后，可能再次启动一个重新运行，而每次启动称为一次运行尝试，`RMAppAttempt` 维护了一次运行尝试的整个生命周期
		- `RMContainer`： `RMContainer` 维护了一个 `Container` 的运行周期，包括从创建到运行结束整个过程
		- `RMNode`： `RMNode` 维护了一个 `NodeManager` 的生命周期，包括启动到运行结束整个过程
	- 安全管理模块
	- 资源分配模块
		- `ResourceScheduler` 是一个插拔式模块，`Yarn` 自带了一个批处理资源调度器 `FIFO` 和两个多用户调度器 `FairScheduler` 和 `CapacityScheduler`
#### 3.1.2.1 main()
```java
public static void main(String argv[]) {
    Thread.setDefaultUncaughtExceptionHandler(new YarnUncaughtExceptionHandler());
    StringUtils.startupShutdownMessage(ResourceManager.class, argv, LOG);
    try {
      Configuration conf = new YarnConfiguration();
      GenericOptionsParser hParser = new GenericOptionsParser(conf, argv);
      argv = hParser.getRemainingArgs();
      // 启动脚本中没有参数(非HA)
      // If -format-state-store, then delete RMStateStore; else startup normally
      if (argv.length >= 1) {...} 
        else {
        // 创建ResourceManager对象
        ResourceManager resourceManager = new ResourceManager();
        ShutdownHookManager.get().addShutdownHook(
          new CompositeServiceShutdownHook(resourceManager),
          SHUTDOWN_HOOK_PRIORITY);
        // 初始化后启动
        resourceManager.init(conf);
        resourceManager.start();
      }
    } catch (Throwable t) {
      LOG.error(FATAL, "Error starting ResourceManager", t);
      System.exit(-1);
    }
}

// ResourceManager继承自CompositeService,CompositeService又继承自AbstractService
// 构造器向上调用
public ResourceManager() {
    super("ResourceManager");
}

public CompositeService(String name) {
    super(name);
}

// 建立name为ResourceManager的服务
public AbstractService(String name) {
    this.name = name;
    stateModel = new ServiceStateModel(name);
}
```
#### 3.1.2.2 RM 的初始化
```java
// 先调用父类AbstractService的init方法，它实现了Service接口，重写了Service中的init()
@override
public void init(Configuration conf) {
    synchronized (stateChangeLock) {
      if (enterState(STATE.INITED) != STATE.INITED) {
        setConfig(conf);
        try {
          // 初始化服务
          serviceInit(config);
          if (isInState(STATE.INITED)) {
            //if the service ended up here during init,
            //notify the listeners
            notifyListeners();
          }
        }
      }
    }
}

// RM重写的serviceInit
protected void serviceInit(Configuration conf) throws Exception {
    this.conf = conf;
    UserGroupInformation.setConfiguration(conf);
    // 创建RM的上下文
    this.rmContext = new RMContextImpl();
    rmContext.setResourceManager(this);
    rmContext.setYarnConfiguration(conf);

    rmStatusInfoBean = new RMInfo(this);
    rmStatusInfoBean.register();

    // Set HA configuration should be done before login
    ...
    // Set UGI and do login
    // If security is enabled, use login user
    // If security is not enabled, use current user
    ...
    // load core-site.xml
    loadConfigurationXml(YarnConfiguration.CORE_SITE_CONFIGURATION_FILE);
	// 从已加载的 core-site.xml文件中获取 用户和组之间 的映射表
    // Do refreshSuperUserGroupsConfiguration with loaded core-site.xml
    // Or use RM specific configurations to overwrite the common ones first
    // if they exist
    
    RMServerUtils.processRMProxyUsersConf(conf);
    ProxyUsers.refreshSuperUserGroupsConfiguration(this.conf);

    // load yarn-site.xml
    loadConfigurationXml(YarnConfiguration.YARN_SITE_CONFIGURATION_FILE);

    validateConfigs(this.conf);

    // register the handlers for all AlwaysOn services using setupDispatcher().
    // 注册异步dispatcher后加入上下文
    rmDispatcher = setupDispatcher();
    //通过CompositeService将dispatcher加入serviceList
    addIfService(rmDispatcher);
    rmContext.setDispatcher(rmDispatcher);
	// 管理员以及HA配置
    ...
    // 创建ActiveServices    
    createAndInitActiveServices(false);
	// 8088端口
    webAppAddress = WebAppUtils.getWebAppBindURL(this.conf,
                      YarnConfiguration.RM_BIND_HOST,
                     WebAppUtils.getRMWebAppURLWithoutScheme(this.conf));
	// 持久化RMApp, RMAppAttempt, RMContainer的信息
    RMApplicationHistoryWriter rmApplicationHistoryWriter =
        createRMApplicationHistoryWriter();
    addService(rmApplicationHistoryWriter);
    rmContext.setRMApplicationHistoryWriter(rmApplicationHistoryWriter);
    ...
	//调用父类方法，初始化全部服务
    super.serviceInit(this.conf);
}
```
##### 3.1.2.2.1 new RMContextImpl()
```java
// RMContextImpl.java
public RMContextImpl() {
    this.serviceContext = new RMServiceContext();
    this.activeServiceContext = new RMActiveServiceContext();
}

// RMServiceContext维护了始终运行的服务
public class RMServiceContext {

  private Dispatcher rmDispatcher;
  private boolean isHAEnabled;
  private HAServiceState haServiceState =
      HAServiceProtocol.HAServiceState.INITIALIZING;
  private AdminService adminService;
  private ConfigurationProvider configurationProvider;
  private Configuration yarnConfiguration;
  private RMApplicationHistoryWriter rmApplicationHistoryWriter;
  private SystemMetricsPublisher systemMetricsPublisher;
  private EmbeddedElector elector;
  private final Object haServiceStateLock = new Object();
  private ResourceManager resourceManager;
  private RMTimelineCollectorManager timelineCollectorManager;
    ...
}

// RMActiveServiceContext维护了存活的服务

public RMActiveServiceContext() {
    queuePlacementManager = new PlacementManager();
}
// 实际创建了一个锁对象
public PlacementManager() {
    ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    readLock = lock.readLock();
    writeLock = lock.writeLock();
}
```
##### 3.1.2.2.2 setupDispatcher()
```java
private Dispatcher setupDispatcher() {
    Dispatcher dispatcher = createDispatcher();
    // 将dispatcher放在eventDispatchers中
    dispatcher.register(RMFatalEventType.class,
        new ResourceManager.RMFatalEventDispatcher());
    return dispatcher;
}

protected Dispatcher createDispatcher() {
    // 创建一个异步的Dispatcher，将线程名定义为RM Event dispatcher
    AsyncDispatcher dispatcher = new AsyncDispatcher("RM Event dispatcher");
    return dispatcher;
}

public AsyncDispatcher(String dispatcherName) {
    this();
    dispatcherThreadName = dispatcherName;
}

public AsyncDispatcher() {
    this(new LinkedBlockingQueue<Event>());
}

// 向上调用含参数的构造器
public AsyncDispatcher(BlockingQueue<Event> eventQueue) {
    // AsyncDispatcher也是服务，继承了AbstrcatService类
    // stateModel = new ServiceStateModel(name);
    // public ServiceStateModel(String name) {
    // this(name, Service.STATE.NOTINITED) }
    super("Dispatcher");
    // LinkedBlockingQueue
    this.eventQueue = eventQueue;
    this.eventDispatchers = new HashMap<Class<? extends Enum>, EventHandler>();
}
```
##### 3.1.2.2.3 createAndInitActiveServices()
```java
// 创建启动RMActiveServices
protected void createAndInitActiveServices(boolean fromActive) {
    activeServices = new RMActiveServices(this);
    activeServices.fromActive = fromActive;
    activeServices.init(conf);
}

// RMActiveServices是RM的内部类，继承了CompositeService
public class RMActiveServices extends CompositeService {

    private DelegationTokenRenewer delegationTokenRenewer;
    private EventHandler<SchedulerEvent> schedulerDispatcher;
    private ApplicationMasterLauncher applicationMasterLauncher;
    private ContainerAllocationExpirer containerAllocationExpirer;
    private ResourceManager rm;
    private boolean fromActive = false;
    private StandByTransitionRunnable standByTransitionRunnable;
    private RMNMInfo rmnmInfo;

}
// RMActiveServices也是服务，向上调用
RMActiveServices(ResourceManager rm) {
      super("RMActiveServices");
      this.rm = rm;
}

// 初始化服务
protected void serviceInit(Configuration configuration) throws Exception {
      standByTransitionRunnable = new StandByTransitionRunnable();

      rmSecretManagerService = createRMSecretManagerService();
      addService(rmSecretManagerService);
	  // 创建ContainerAllocationExpirer
      // 继承自AbstractLivelinessMonitor同样也继承了抽象服务类
      containerAllocationExpirer = new ContainerAllocationExpirer(rmDispatcher);
      addService(containerAllocationExpirer);
      rmContext.setContainerAllocationExpirer(containerAllocationExpirer);
      // 创建AMLivelinessMonitor监控AM
      AMLivelinessMonitor amLivelinessMonitor = createAMLivelinessMonitor();
      addService(amLivelinessMonitor);
      rmContext.setAMLivelinessMonitor(amLivelinessMonitor);

      // Register event handler for NodesListManager
      // 创建NodesListManager管理NM
      nodesListManager = new NodesListManager(rmContext);
      rmDispatcher.register(NodesListManagerEventType.class, nodesListManager);
      addService(nodesListManager);
      rmContext.setNodesListManager(nodesListManager);

      // Initialize the scheduler
      // 默认容量调度器
      scheduler = createScheduler();
      scheduler.setRMContext(rmContext);
      addIfService(scheduler);
      rmContext.setScheduler(scheduler);

	  // 创建NMLivelinessMonitor
      nmLivelinessMonitor = createNMLivelinessMonitor();
      addService(nmLivelinessMonitor);
	  // 创建ResourceTrackerService
      resourceTracker = createResourceTrackerService();
      addService(resourceTracker);
      rmContext.setResourceTrackerService(resourceTracker);
      
      // 创建ApplicationMasterService
      masterService = createApplicationMasterService();
      createAndRegisterOpportunisticDispatcher(masterService);
      addService(masterService) ;
      rmContext.setApplicationMasterService(masterService);

      applicationACLsManager = new ApplicationACLsManager(conf);

      queueACLsManager = createQueueACLsManager(scheduler, conf);
	  // 创建RMAppManager
      rmAppManager = createRMAppManager();
      // Register event handler for RMAppManagerEvents
      rmDispatcher.register(RMAppManagerEventType.class, rmAppManager);
	  // 创建ClientRMService
      clientRM = createClientRMService();
      addService(clientRM);
      rmContext.setClientRMService(clientRM);
	  // 创建AMLauncher
      applicationMasterLauncher = createAMLauncher();
      rmDispatcher.register(AMLauncherEventType.class,
          applicationMasterLauncher);
      super.serviceInit(conf);
}
```
##### 3.1.2.2.4 getRMWebAppURLWithoutScheme()
```java
// WebAppUtils.java
public static String getRMWebAppURLWithoutScheme(Configuration conf) {
    return getRMWebAppURLWithoutScheme(conf, false, 0);
}

public static String getRMWebAppURLWithoutScheme(Configuration conf,
      boolean isHAEnabled, int haIdIndex)  {
    YarnConfiguration yarnConfig = new YarnConfiguration(conf);
    // 是否设置了HTTP策略
    if (YarnConfiguration.useHttps(yarnConfig)) {...}
    else {
      if (isHAEnabled) {...}
      // 未设置HA
      // public static final int DEFAULT_RM_WEBAPP_PORT = 8088;
      // public static final String DEFAULT_RM_WEBAPP_ADDRESS = "0.0.0.0:" + DEFAULT_RM_WEBAPP_PORT;
      return yarnConfig.get(YarnConfiguration.RM_WEBAPP_ADDRESS,
          YarnConfiguration.DEFAULT_RM_WEBAPP_ADDRESS);
    }
}
```
#### 3.1.2.3 RM 的启动
```java
// 首先调用父类AbstractService的start()
public void start() {
    if (isInState(STATE.STARTED)) {
      return;
    }
    //enter the started state
    synchronized (stateChangeLock) {
      if (stateModel.enterState(STATE.STARTED) != STATE.STARTED) {
        try {
          startTime = System.currentTimeMillis();
          serviceStart();
          if (isInState(STATE.STARTED)) {
            //if the service started (and isn't now in a later state), notify
            LOG.debug("Service {} is started", getName());
            notifyListeners();
          }
        }
      }
    }
}

protected void serviceStart() throws Exception {
    if (this.rmContext.isHAEnabled()) {
      transitionToStandby(false);
    }
	// 开启webApp
	// 通过Builder创建HttpServer2服务
    startWepApp();
    if (getConfig().getBoolean(YarnConfiguration.IS_MINI_YARN_CLUSTER,
        false)) {
      int port = webApp.port();
      WebAppUtils.setRMWebAppPort(conf, port);
    }
    super.serviceStart();

    // Non HA case, start after RM services are started.
    if (!this.rmContext.isHAEnabled()) {
      transitionToActive();
    }
}
```

### 3.1.3 NodeManager

![[nodeManager 1.svg]]

- 主要功能
	- 启动后向 `RM` 注册，然后保持通信，通过心跳汇报自己的状态，接受来自 `RM` 的指令
	- 监控节点的健康状态，并与 `RM` 同步
	- 管理节点上所有 `Container` 的生命周期，监控 `Container` 的资源使用情况，向 `RM` 汇报 `Container` 的状态信息
	- 管理分布式缓存(缓存 jar 包，配置文件等)
- 核心组件
	- `NodeStatusUpdater`
		- `NN` 和 `RM` 通信的唯一渠道，主要用于 `NM` 刚启动或重启后向 `RM` 注册
		- `NM` 启动注册后，汇报本节点资源情况，后面周期性地汇报节点的健康状态，`Containers` 运行情况
		- `RM` 会在 `NM` 注册时给其发送一个令牌，`NM` 保存这个令牌后，用这个令牌为 `AM` 请求的 `Container` 做认证
	- `ContainerManager`: 负责 `Container` 的分配与管理
		- `ContainerLuncher` 维护了一个线程池，用于杀死或启动 `Container`
			- `RM` 通过 `NodeStatusUpdater` 或者 `AM` 通过 `RPC` 服务要求清理某个 `Container` 时，它就会杀死 `Container`
			- `AM` 通过 `RPCServer` 发送启动请求后，`ContainerLauncher` 就会启动 `Container`
		- `ContainerMonitor` 管理 `NM` 运行中的 `Container` 的资源使用情况，超过申请资源的容器就会被杀死
		- `ResourceLocaliazationService` 在启动 `Container` 前将其需要的所有的资源安全地下载到本地磁盘，比如从 `HDFS` 上下载的文件
	- `NodeHealthCheckService`:节点的健康检查通过`YARN`提供的脚本定期运行，将检查结果通过`NodeStatusUpdater`汇报给`RM`,以方便`RM`做节点的监控管理
#### 3.1.3.1 main()
```java
public static void main(String[] args) throws IOException {
    Thread.setDefaultUncaughtExceptionHandler(new YarnUncaughtExceptionHandler());
    StringUtils.startupShutdownMessage(NodeManager.class, args, LOG);
    @SuppressWarnings("resource")
    // 创建NodeManager实例，服务名为NodeManager，服务状态为NOTINITED
    // public NodeManager() {
    // super(NodeManager.class.getName());}
    NodeManager nodeManager = new NodeManager();
    Configuration conf = new YarnConfiguration();
    new GenericOptionsParser(conf, args);
    // 初始化和启动NM
    nodeManager.initAndStartNodeManager(conf, false);
}

private void initAndStartNodeManager(Configuration conf, boolean hasToReboot) {
      ...  
      this.init(conf);
      this.start();
    } catch (Throwable t) {
      LOG.error("Error starting NodeManager", t);
      System.exit(-1);
    }
}
```
#### 3.1.3.2 serviceInit()
```java
// 调用父类的init()，再调用serviceInit()
protected void serviceInit(Configuration conf) throws Exception {
    UserGroupInformation.setConfiguration(conf);
    rmWorkPreservingRestartEnabled = conf.getBoolean(YarnConfiguration
            .RM_WORK_PRESERVING_RECOVERY_ENABLED,
        YarnConfiguration.DEFAULT_RM_WORK_PRESERVING_RECOVERY_ENABLED);

    try {
      initAndStartRecoveryStore(conf);
    } 
	
    // 容器令牌管理器
    NMContainerTokenSecretManager containerTokenSecretManager =
        new NMContainerTokenSecretManager(conf, nmStore);
	 // NM令牌管理器
    NMTokenSecretManagerInNM nmTokenSecretManager =
        new NMTokenSecretManagerInNM(nmStore);

    recoverTokens(nmTokenSecretManager, containerTokenSecretManager);
    // App权限管理器
    this.aclsManager = new ApplicationACLsManager(conf);

    this.dirsHandler = new LocalDirsHandlerService(metrics);

    boolean isDistSchedulingEnabled =
        conf.getBoolean(YarnConfiguration.DIST_SCHEDULING_ENABLED,
            YarnConfiguration.DEFAULT_DIST_SCHEDULING_ENABLED);
	// NM上下文
    this.context = createNMContext(containerTokenSecretManager,
        nmTokenSecretManager, nmStore, isDistSchedulingEnabled, conf);

    ResourcePluginManager pluginManager = createResourcePluginManager();
    pluginManager.initialize(context);
    ((NMContext)context).setResourcePluginManager(pluginManager);
	// 容器启动器
    ContainerExecutor exec = createContainerExecutor(conf);
    try {
      exec.init(context);
    } catch (IOException e) {
      throw new YarnRuntimeException("Failed to initialize container executor", e);
    }
    DeletionService del = createDeletionService(exec);
    addService(del);

    // NodeManager level dispatcher
    this.dispatcher = createNMDispatcher();

    this.nodeHealthChecker = new NodeHealthCheckerService(dirsHandler);
    addService(nodeHealthChecker);

    ((NMContext)context).setContainerExecutor(exec);
    ((NMContext)context).setDeletionService(del);

	// 节点状态更新服务
	// 涉及到和RM通信
    nodeAttributesProvider = createNodeAttributesProvider(conf);
    if (nodeAttributesProvider != null) {
      addIfService(nodeAttributesProvider);
      nodeStatusUpdater.setNodeAttributesProvider(nodeAttributesProvider);
    }
	// 节点资源监控服务
    nodeResourceMonitor = createNodeResourceMonitor();
    addService(nodeResourceMonitor);
    ((NMContext) context).setNodeResourceMonitor(nodeResourceMonitor);
	// 容器管理器服务
    containerManager =
        createContainerManager(context, exec, del, nodeStatusUpdater,
        this.aclsManager, dirsHandler);
    addService(containerManager);
    ((NMContext) context).setContainerManager(containerManager);

    this.nmLogAggregationStatusTracker = createNMLogAggregationStatusTracker(
        context);
    addService(nmLogAggregationStatusTracker);
    ((NMContext)context).setNMLogAggregationStatusTracker(
        this.nmLogAggregationStatusTracker);
	// web服务
    WebServer webServer = createWebServer(context, containerManager
        .getContainersMonitor(), this.aclsManager, dirsHandler);
    addService(webServer);
    ((NMContext) context).setWebServer(webServer);

    dispatcher.register(ContainerManagerEventType.class, containerManager);
    dispatcher.register(NodeManagerEventType.class, this);
    addService(dispatcher);

    // Do secure login before calling init for added services.
    try {
      doSecureLogin();
    } catch (IOException e) {
      throw new YarnRuntimeException("Failed NodeManager login", e);
    }

    registerMXBean();

    context.getContainerExecutor().start();
    // 循环初始化上面的服务
    super.serviceInit(conf);
    // TODO add local dirs to del
}
```
### 3.1.4 NodeStatusUpdater 相关
```java
// 创建
protected NodeStatusUpdater createNodeStatusUpdater(Context context,  
    Dispatcher dispatcher, NodeHealthCheckerService healthChecker) {  
  return new NodeStatusUpdaterImpl(context, dispatcher, healthChecker,  
      metrics);  
}

// 启动服务
protected void serviceStart() throws Exception {

    // NodeManager is the last service to start, so NodeId is available.
    this.nodeId = this.context.getNodeId();
    LOG.info("Node ID assigned is : " + this.nodeId);
    this.httpPort = this.context.getHttpPort();
    this.nodeManagerVersionId = YarnVersionInfo.getVersion();
    try {
	  // 创建RMClient，用于和RM进行通信
      this.resourceTracker = getRMClient();
      //配置必要配置信息，和安全认证操作利用Hadoop RPC远程调用RM端
      //ResourcesTrackerService.registerNodeManager()方法  
      registerWithRM();
      super.serviceStart();
      /*
       * 创建一个线程，然后启动，所有操作都在运行while的循环中
       * 获取本地Container和本地Node的状态，以供后面的nodeHeartbeat()方法使用
       * 设置、获取和输出必要配置信息，其中比较重要的调用getNodeStatus()方法
       * 通过Hadoop RPC远程调用RM端ResourcesTrackerService下的nodeHeartbeat()函数
       * 用while循环以一定时间间隔向RM发送心跳信息，心跳操作通过ResourcesTrackerService下的nodeHeartbeat()函数
       * nodeHeartbeat()将返回给NM信息，根据返回的response，标记不需要的Container和Application发送相应的FINISH_CONTAINERS和 FINISH_APPS给ContainerManager，进行清理操作
      */
      startStatusUpdater();
    } catch (Exception e) {
      String errorMessage = "Unexpected error starting NodeStatusUpdater";
      LOG.error(errorMessage, e);
      throw new YarnRuntimeException(e);
    }
}
```
## 3.2 Service 框架
- Yarn 中的所有组件都统一实现了 Service 接口，用具体的 Service 接口表示抽象的服务概念
```java
public interface Service extends Closeable {
	// 内部定义的枚举类表示服务的四种状态
	public enum STATE {  
	  /** Constructed but not initialized */  
	  NOTINITED(0, "NOTINITED"),  
  
	  /** Initialized but not started or stopped */  
	  INITED(1, "INITED"),  
  
	  /** started and not stopped */  
	  STARTED(2, "STARTED"),  
  
	  /** stopped. No further state transitions are permitted */  
	  STOPPED(3, "STOPPED");}

// 初始化方法 NOINITED -> INITED
void init(Configuration config);
// 启动方法 INITED -> STARTED
void start();
// 停止方法 STARTED -> STOPPED
void stop();
```
- `Yarn` 使用抽象类 `AbstractService` 实现 `Service` 接口的 `init()`等方法，组件只需要重写 `AbstractService` 中的 `serviceInit`, `serviceStart` 等方法即可
### 3.2.1 AbstractService 中状态的转换
```java
public void init(Configuration conf) {  
  synchronized (stateChangeLock) {  
	// enterState方法传入新状态，返回旧状态
	// 从NOINITED转为INITED
    if (enterState(STATE.INITED) != STATE.INITED) {  
      setConfig(conf);  
      try {  
        // 组件调用自己的初始化方法
        serviceInit(config);  
      }  
    }  
  }  
}

public void start() {   
  //enter the started state  
  synchronized (stateChangeLock) { 
	// 状态转为 STARTED
    if (stateModel.enterState(STATE.STARTED) != STATE.STARTED) {  
      try {  
        startTime = System.currentTimeMillis();  
        serviceStart();  
          notifyListeners();  
        }  
      }
    }  
  }  
}

// stop方法也类似，将状态转换为STOPPED

// ServiceStateModel.java, 是AbstractService的成员变量
// 每个服务对象包含一个ServiceStateModel成员，服务状态用ServiceStateModel中的state成员表示
public synchronized Service.STATE enterState(Service.STATE proposed) {  
  checkStateTransition(name, state, proposed);  
  Service.STATE oldState = state;  
  //atomic write of the new state  
  state = proposed;  
  return oldState;  
}

// 防止状态的不合法转换，ServiceStateModel定义了一个statemap，来限制状态的转换，在checkStateTransition中进行合法性判断
private static final boolean[][] statemap =  
  {  
    //                uninited inited started stopped  
    /* uninited  */    {false, true,  false,  true},  
    /* inited    */    {false, true,  true,   true},  
    /* started   */    {false, false, true,   true},  
    /* stopped   */    {false, false, false,  true},  
  };

public static void checkStateTransition(String name,  
                                        Service.STATE state,  
                                        Service.STATE proposed) {  
  if (!isValidStateTransition(state, proposed)) {...
  }  
}

public static boolean isValidStateTransition(Service.STATE current,  
                                             Service.STATE proposed) {  
  boolean[] row = statemap[current.getValue()];  
  return row[proposed.getValue()];  
}
```
### 3.2.2 组合服务 CompositeService
- 一个 `AbstractService` 只能表示一个独立的 `Service`，而类似 `RM` 这些服务本身包含许多子服务，定义为一个 `CompositeService`，它是 `AbstractService` 的子类
```java
// CompositeService通过serviceList集合来保存每个子服务
private final List<Service> serviceList = new ArrayList<Service>();

// CompositeService的启动或停止，会遍历子服务来进行启动或停止
protected void serviceStart() throws Exception {  
  List<Service> services = getServices();  
  if (LOG.isDebugEnabled()) {  
    LOG.debug(getName() + ": starting services, size=" + services.size());  
  }  
  for (Service service : services) {  
    // start the service. If this fails that service  
    // will be stopped and an exception raised    service.start();  
  }  
  super.serviceStart();  
}
```
## 3.3 事件驱动模型框架
### 3.3.1 事件
- Yarn 中的事件通过是实现 Event 接口来表示一种行为状态
```java
public interface Event<TYPE extends Enum<TYPE>> {  
  
  TYPE getType(); // 获取事件状态
  long getTimestamp();   
}
```
- `Yarn` 基于生产者-消费者模式处理事件，使用 `EventQueue` 存放事件，生产者向 `EventQueue` 中存放事件数据，消费者处理 `EventQueue` 中的时间数据，`Yarn` 通过各种事件的 `EventHandler` 来处理事件
```java
public interface EventHandler<T extends Event> {  
  void handle(T event);   
}

// Yarn通过调度器来让handler处理对应的event
public interface Dispatcher {  
  EventHandler<Event> getEventHandler(); 
  // 注册不同类型事件Event对应的Handler
  void register(Class<? extends Enum> eventType, EventHandler handler);  
}
```
### 3.3.2 异步事件调度 AsyncDispatcher
```java
// AsyncDispatcher继承了AbstractService，实现了Dispatcher接口
public class AsyncDispatcher extends AbstractService implements Dispatcher {  

  // 存放事件的队列
  private final BlockingQueue<Event> eventQueue;  
  // 通用的handler
  // For drainEventsOnStop enabled only, block newly coming events into the  
  // queue while stopping.  private volatile boolean blockNewEvents = false;  
  private final EventHandler<Event> handlerInstance = new GenericEventHandler();  
  // 分发并处理事件的线程
  private Thread eventHandlingThread;  
  // 存放不同类型事件对应的handler
  protected final Map<Class<? extends Enum>, EventHandler> eventDispatchers;  
}
```
#### 3.3.2.1 服务创建过程
```java
// 3.1.2.2中初始化ResourceManager时，会创建AsyncDispatcher服务
rmDispatcher = setupDispatcher();  
addIfService(rmDispatcher);  
rmContext.setDispatcher(rmDispatcher);

// 创建Dispatcher
private Dispatcher setupDispatcher() {  
  Dispatcher dispatcher = createDispatcher();  
  dispatcher.register(RMFatalEventType.class,  
      new ResourceManager.RMFatalEventDispatcher());  
  return dispatcher;  
}

protected Dispatcher createDispatcher() {  
  // 创建AsyncDispatcher
  AsyncDispatcher dispatcher = new AsyncDispatcher("RM Event dispatcher");  
  return dispatcher;  
}

// 注册事件类型对应的handler，使用eventDispatchers存储Type->handler的映射
public void register(Class<? extends Enum> eventType,  
    EventHandler handler) {  
  /* check to see if we have a listener registered */  
  EventHandler<Event> registeredHandler = (EventHandler<Event>)  
  eventDispatchers.get(eventType);  
  if (registeredHandler == null) {  
    eventDispatchers.put(eventType, handler);  
  }  
}
```
#### 3.3.2.2 服务初始化启动过程
```java
protected void serviceInit(Configuration conf) throws Exception{  
  super.serviceInit(conf);  
  this.detailsInterval = getConfig().getInt(YarnConfiguration.  
                  YARN_DISPATCHER_PRINT_EVENTS_INFO_THRESHOLD,  
          YarnConfiguration.  
                  DEFAULT_YARN_DISPATCHER_PRINT_EVENTS_INFO_THRESHOLD);  
}

protected void serviceStart() throws Exception {  
  //start all the components  
  super.serviceStart();  
  eventHandlingThread = new Thread(createThread());  
  eventHandlingThread.setName(dispatcherThreadName);  
  eventHandlingThread.start();  
}

// 创建并启动eventHandlingThread
// 从队列中取到event，进行分发交给handler处理
Runnable createThread() {  
  return new Runnable() {  
    @Override  
    public void run() {  
      while (!stopped && !Thread.currentThread().isInterrupted()) {  
        Event event;  
        try {  
          event = eventQueue.take();  
        }
        if (event != null) {  
          if (eventTypeMetricsMap.  
              get(event.getType().getDeclaringClass()) != null) {  
            long startTime = clock.getTime();  
            dispatch(event);  
            eventTypeMetricsMap.get(event.getType().getDeclaringClass())  
                .increment(event.getType(),  
                    clock.getTime() - startTime);  
          } else {  
            dispatch(event);  
          }  
        }  
      }  
    }  
  };  
}

protected void dispatch(Event event) {    
  // 获取到事件类型
  Class<? extends Enum> type = event.getType().getDeclaringClass();  
  
  try{  
   // 获取到handler
    EventHandler handler = eventDispatchers.get(type);  
    if(handler != null) {  
      handler.handle(event);  
    }
  }  
}
```
## 3.4 状态机
- `Yarn` 中主要使用四个状态机
	- `RMApp`：维护一个 `Application` 的生命周期
	- `RMAppAttempt`：维护一次 `attempt` 的生命周期
	- `RMContainer`：维护 `Container` 的生命周期
	- `RMNode`：维护 `NM` 的生命周期 
### 3.4.1 状态机初始化
#### 3.4.1.1 StateMachineFactory
```java
// StateMachineFactory通过installTopology方法创建状态机
private static final StateMachineFactory<RMAppImpl,  
                                         RMAppState,  
                                         RMAppEventType,  
                                         RMAppEvent> stateMachineFactory  
	 = new StateMachineFactory<RMAppImpl,  
								 RMAppState,  
								 RMAppEventType,  
								 RMAppEvent>(RMAppState.NEW)
		 .addTransition()...
		 .installTopology();

// StateMachineFactory有两个属性
final public class StateMachineFactory  
             <OPERAND, STATE extends Enum<STATE>,  
              EVENTTYPE extends Enum<EVENTTYPE>, EVENT> {  
  // 暂存Transition，最终存储在stateMachineTable中
  private final TransitionsListNode transitionsListNode;  
  
  private Map<STATE, Map<EVENTTYPE,  
    Transition<OPERAND, STATE, EVENTTYPE, EVENT>>> stateMachineTable;
}
```
#### 3.4.1.2 TransitionsListNode
```java
// TransitionsListNode是链表结构，存放
private class TransitionsListNode {  
  final ApplicableTransition<OPERAND, STATE, EVENTTYPE, EVENT> transition;  
  final TransitionsListNode next;
}
// ApplicableTransition接口
/*
 * 四个泛型
 *   OPERAND -> 操作对象
 *   STATE -> 目的状态
 *   EVENTTYPE -> 事件类型
 *   EVENT -> 事件
*/ 
```
#### 3.4.1.3 ApplicableTransition
```java
private interface ApplicableTransition  
           <OPERAND, STATE extends Enum<STATE>,  
            EVENTTYPE extends Enum<EVENTTYPE>, EVENT> {  
  // 将TransitionsListNode节点存放至stateMachineTable中
  void apply(StateMachineFactory<OPERAND, STATE, EVENTTYPE, EVENT> subject);  
}

// ApplicableTransition的实现类ApplicableSingleOrMultipleTransition
public void apply  
         (StateMachineFactory<OPERAND, STATE, EVENTTYPE, EVENT> subject) {  
 // 获取preState的对应表
  Map<EVENTTYPE, Transition<OPERAND, STATE, EVENTTYPE, EVENT>> transitionMap  
    = subject.stateMachineTable.get(preState);
  // 放入事件类型和处理方法  
  transitionMap.put(eventType, transition);  
}
```
#### 3.4.1.4 Transition
```java
// Transition接口执行状态转换
private interface Transition<OPERAND, STATE extends Enum<STATE>,  
        EVENTTYPE extends Enum<EVENTTYPE>, EVENT> {  
  STATE doTransition(OPERAND operand, STATE oldState,  
                     EVENT event, EVENTTYPE eventType);  
}

// Transition接口有两个实现类
// SingleInternalArc表示初始状态转换后，只有一种结束状态
public interface SingleArcTransition<OPERAND, EVENT> {  
	public void transition(OPERAND operand, EVENT event);
}
// MultipleInternalArc表示状态转换后，根据Transition的执行结果返回结束状态
 public interface MultipleArcTransition  
        <OPERAND, EVENT, STATE extends Enum<STATE>> {  
	public STATE transition(OPERAND operand, EVENT event);  
	}
}
```
#### 3.4.1.5 stateMachineTable
- `stateMachineTable` 就是状态机，由两层 Map 组成
```java
/*
 * 外层Map的STATE表示旧状态，内层Map的EVENTTYPE表示事件类型
 * 内层Map的value是Transition
 * stateMachineTable的作用是：
 *     组件可能有多种旧状态，每种旧状态对应多种事件类型
 *     根据旧状态和要处理事件的类型，映射到状态转换方法和目的状态
*/
Map<STATE, Map<EVENTTYPE,  
  Transition<OPERAND, STATE, EVENTTYPE, EVENT>>> stateMachineTable;
```
### 3.4.2 构建状态机
#### 3.4.2.1 addTransition()
```java
// SigleArcTransition和MultipleArcTransition注册的方法不同
// SigleArcTransition
public StateMachineFactory  
           <OPERAND, STATE, EVENTTYPE, EVENT>  
        addTransition(STATE preState, STATE postState,  
                      EVENTTYPE eventType,  
                      SingleArcTransition<OPERAND, EVENT> hook){  
  return new StateMachineFactory<OPERAND, STATE, EVENTTYPE, EVENT>  
      (this, new ApplicableSingleOrMultipleTransition<OPERAND, STATE, EVENTTYPE, EVENT>  
         (preState, eventType, new SingleInternalArc(postState, hook)));  
}

// MultipleArcTransition 
// 第二个传入的是EnumSet.of(...)
public StateMachineFactory  
           <OPERAND, STATE, EVENTTYPE, EVENT>  
        addTransition(STATE preState, Set<STATE> postStates,  
                      EVENTTYPE eventType,  
                      MultipleArcTransition<OPERAND, EVENT, STATE> hook){  
  return new StateMachineFactory<OPERAND, STATE, EVENTTYPE, EVENT>  
      (this,  
       new ApplicableSingleOrMultipleTransition<OPERAND, STATE, EVENTTYPE, EVENT>  
         (preState, eventType, new MultipleInternalArc(postStates, hook)));  
}
```
#### 3.4.2.2 new StateMachineFactory()
```java
// addTransiton只完成对TransitionsListNode的构建，最终通过installTopology()方法构建映射表
private StateMachineFactory  
    (StateMachineFactory<OPERAND, STATE, EVENTTYPE, EVENT> that,  
     ApplicableTransition<OPERAND, STATE, EVENTTYPE, EVENT> t) {  
  this.defaultInitialState = that.defaultInitialState;  
  // 向旧的TransitionsListNode中添加preState的Node
  this.transitionsListNode = new TransitionsListNode(t, that.transitionsListNode);  
  this.optimized = false;  
  this.stateMachineTable = null;  
}

public StateMachineFactory  
           <OPERAND, STATE, EVENTTYPE, EVENT>  
        installTopology() {  
  return new StateMachineFactory<OPERAND, STATE, EVENTTYPE, EVENT>(this, true);  
}

private StateMachineFactory  
    (StateMachineFactory<OPERAND, STATE, EVENTTYPE, EVENT> that,  
     boolean optimized) {  
  this.defaultInitialState = that.defaultInitialState;  
  this.transitionsListNode = that.transitionsListNode;  
  this.optimized = optimized;  
  // 上一个方法传入的是true，构建映射表
  if (optimized) {  
    makeStateMachineTable();  
  }
}
```
#### 3.4.2.3 makeStateMachineTable()
```java
private void makeStateMachineTable() {  
  Stack<ApplicableTransition<OPERAND, STATE, EVENTTYPE, EVENT>> stack =  
    new Stack<ApplicableTransition<OPERAND, STATE, EVENTTYPE, EVENT>>();  
 // 入栈
  for (TransitionsListNode cursor = transitionsListNode;  
       cursor != null;  
       cursor = cursor.next) {  
    stack.push(cursor.transition);  
  }  
  // apply处理后出栈
  while (!stack.isEmpty()) {  
    stack.pop().apply(this);  
  }  
}

// 注册Transition到stateMachineTable中
public void apply  
         (StateMachineFactory<OPERAND, STATE, EVENTTYPE, EVENT> subject) {  
  Map<EVENTTYPE, Transition<OPERAND, STATE, EVENTTYPE, EVENT>> transitionMap  
    = subject.stateMachineTable.get(preState);  
  if (transitionMap == null) {  
    transitionMap = new HashMap<EVENTTYPE,  
      Transition<OPERAND, STATE, EVENTTYPE, EVENT>>();  
    subject.stateMachineTable.put(preState, transitionMap);  
  }  
  transitionMap.put(eventType, transition);  
}
```
### 3.4.3 使用状态机
- `EventHandler` 调用 `handle` 方法就会转换状态
```java
public void handle(RMAppEvent event) {  
  this.writeLock.lock();  
  try {   
    final RMAppState oldState = getState();  
    try {  
      this.stateMachine.doTransition(event.getType(), event); 
		} 
	} 
}  

public synchronized STATE doTransition(EVENTTYPE eventType, EVENT event)  
     throws InvalidStateTransitionException  {  
  listener.preTransition(operand, currentState, event);  
  previousState = currentState;  
  // 转换状态
  currentState = StateMachineFactory.this.doTransition  
      (operand, currentState, eventType, event);  
  listener.postTransition(operand, previousState, currentState, event);  
  return currentState;  
}

private STATE doTransition  
         (OPERAND operand, STATE oldState, EVENTTYPE eventType, EVENT event)  
    throws InvalidStateTransitionException {  
	Map<EVENTTYPE, Transition<OPERAND, STATE, EVENTTYPE, EVENT>> transitionMap  
    = stateMachineTable.get(oldState);  
  // 从transitionMap中取出Transition
  if (transitionMap != null) {  
    Transition<OPERAND, STATE, EVENTTYPE, EVENT> transition  
        = transitionMap.get(eventType);  
    if (transition != null) {  
      // 最终执行实现类的转换
      return transition.doTransition(operand, oldState, event, eventType);  
    }  
  }  
  throw new InvalidStateTransitionException(oldState, eventType);  
}
```
## 3.5 RMContainerImpl
- `RMContainerImpl` 实现了 `RMContainer`
- `RMContainerImpl` 中定义了 `Container` 的几种状态：
	- `NEW`：调度器初始化一个 `RMContainerImpl`
	- `RESERVED`：
		- 当调度器准备把某个 `Container` 分配给相应的 `NM`，资源不能满足需求时，调度器会让 `Container` 预订此 `NM`，然后创建一个 `RMContainerEventType.RESERVED` 事件，`RMContainerImpl` 会调用 `ContainerReservedTransition` 处理这个事件，把预订信息（资源，节点，优先级）保存下来，然后设置自己的状态为 `RESERVED`
	- `ALLOCATED`：
		- `RMContainerImpl` 已经处于分配状态，把相应的 `Container` 标记调度到 `NM` 上，并创建 `RMContainerEventType.START` 事件，`RMContainerImpl` 会调用 `ContainerStartedTransition`，创建 `RMAppAttemptEventType.CONTAINER_ALLOCATED` 事件，然后 `RMContainerImpl` 状态被设置为 `ALLOCATED`
	- `ACQUIRED`：
		- 已经分配资源的 `Container` 通知给 `AM`，`AM` 通过 `ApplicationMasterProtocol.allocate()` 向 `RM` 发起资源请求，`RM` 会调用调度器处理 `AM` 的请求，在调度器中首先会把请求资源保存下来，然后把已经分配的资源返回给 `AM`，这期间调度器会生成 `RMContainerEventType.ACQUIRED` 事件，`RMContainerImpl` 调用 `AcquiredTransition` 处理这个事件，生成 `RMAppAttemptEventType.CONTAINER_ACQUIRED` 事件，然后 `RMContainerImpl` 状态改为 `ACQUIRED` 状态
	- `RUNNING`：`RMContainerImpl` 已处于运行状态
		- 当 `NM` 发送心跳给 `RM`，`NM` 会把自己节点上运行的 `Container` 列表汇报给 `RM`，`RM` 让调度器负责处理，调度器生成 `RMContainerEventType.LAUNCHED` 事件，`RMContainerImpl` 会调用 `LaunchedTransition` 处理此事件，然后 `RMContainerImpl` 状态改为 `RUNNING`
	- `COMPLETED`：
		- 调度器生成 `RMContainerEventType.FINISHED` 事件，`RMContainerImpl` 会调用 `FinishedTransition` 处理此事件，生成 `RMAppAttemptEventType.CONTAINER_FINISHED` 事件，然后 `RMContainerImpl` 改为 `COMPLETED`
	- `EXPIRED`
	- `RELEASED` 
	- `KILLED`

![[RMContainer.gif]]
## 3.6 RMAppImpl
## 3.7 RMAppAttemptImpl
## 3.8 RMNodeImpl
## 3.9 提交执行任务
### 3.9.1 YARNRunner.submitJob()
```java
public JobStatus submitJob(JobID jobId, String jobSubmitDir, Credentials ts)
  throws IOException, InterruptedException {
    
    addHistoryToken(ts);
	// 创建提交任务的上下文
	// ApplicationSubmissionContext 包含了启动 AM 的所有信息
    ApplicationSubmissionContext appContext =
      createApplicationSubmissionContext(conf, jobSubmitDir, ts);

    // Submit to ResourceManager
    try {
      ApplicationId applicationId =
          // 提交任务
          resMgrDelegate.submitApplication(appContext);
          
      return clientCache.getClient(jobId).getJobStatus(jobId);
    } 
}

// Constructs all the necessary information to start the MR AM.
public ApplicationSubmissionContext createApplicationSubmissionContext(
      Configuration jobConf, String jobSubmitDir, Credentials ts)
      throws IOException {
    ApplicationId applicationId = resMgrDelegate.getApplicationId();

    // Setup LocalResources
    // 获取job需要的本地jar包，配置文件
    Map<String, LocalResource> localResources =
        setupLocalResources(jobConf, jobSubmitDir);

    // Setup ContainerLaunchContext for AM container
    // 封装启动AM的命令
    List<String> vargs = setupAMCommand(jobConf);
    // 创建启动Container的上下文
    ContainerLaunchContext amContainer = setupContainerLaunchContextForAM(
        jobConf, localResources, securityTokens, vargs);

    // Set up the ApplicationSubmissionContext
    // 创建ApplicationSubmissionContext
    ApplicationSubmissionContext appContext =
  recordFactory.newRecordInstance(ApplicationSubmissionContext.class);
    // ApplicationId
    appContext.setApplicationId(applicationId); 
    // Queue name
    appContext.setQueue(                                      
        jobConf.get(JobContext.QUEUE_NAME,
        YarnConfiguration.DEFAULT_QUEUE_NAME));
    // Job name
    appContext.setApplicationName(                             
        jobConf.get(JobContext.JOB_NAME,
        YarnConfiguration.DEFAULT_APPLICATION_NAME));
    // AM Container
    appContext.setAMContainerSpec(amContainer);  
    // 默认最大重试次数2
    appContext.setMaxAppAttempts(
        conf.getInt(MRJobConfig.MR_AM_MAX_ATTEMPTS,
            MRJobConfig.DEFAULT_MR_AM_MAX_ATTEMPTS));

    // Setup the AM ResourceRequests
    List<ResourceRequest> amResourceRequests = generateResourceRequests();
    appContext.setAMContainerResourceRequests(amResourceRequests);

    appContext.setApplicationType(MRJobConfig.MR_APPLICATION_TYPE);
    if (tagsFromConf != null && !tagsFromConf.isEmpty()) {
      appContext.setApplicationTags(new HashSet<>(tagsFromConf));
    }

    String jobPriority = jobConf.get(MRJobConfig.PRIORITY);
    if (jobPriority != null) {
      int iPriority;
      try {
        iPriority = TypeConverter.toYarnApplicationPriority(jobPriority);
      } catch (IllegalArgumentException e) {
        iPriority = Integer.parseInt(jobPriority);
      }
      appContext.setPriority(Priority.newInstance(iPriority));
    }

    return appContext;
  }
```
#### 3.9.1.1 setupLocalResources()
```java
private Map<String, LocalResource> setupLocalResources(Configuration jobConf,
      String jobSubmitDir) throws IOException {
    Map<String, LocalResource> localResources = new HashMap<>();
	
    // MRJobConfig.JOB_CONF_FILE="job.xml"
    Path jobConfPath = new Path(jobSubmitDir, MRJobConfig.JOB_CONF_FILE);

    URL yarnUrlForJobSubmitDir = URL.fromPath(defaultFileContext
        .getDefaultFileSystem().resolvePath(
            defaultFileContext.makeQualified(new Path(jobSubmitDir))));
    LOG.debug("Creating setup context, jobSubmitDir url is "
        + yarnUrlForJobSubmitDir);

    localResources.put(MRJobConfig.JOB_CONF_FILE,
        // 将path转为URL
        createApplicationResource(defaultFileContext,
            jobConfPath, LocalResourceType.FILE));
    // 类似的，获取jar包的URL
    if (jobConf.get(MRJobConfig.JAR) != null) {...}

    return localResources;
  }
```
#### 3.9.1.2 setupAMCommand()
```java
private List<String> setupAMCommand(Configuration jobConf) {
    List<String> vargs = new ArrayList<>(8);
    // 启动java的命令
    vargs.add(MRApps.crossPlatformifyMREnv(jobConf, Environment.JAVA_HOME) + "/bin/java");
    Path amTmpDir =
        new Path(MRApps.crossPlatformifyMREnv(conf, Environment.PWD),
            YarnConfiguration.DEFAULT_CONTAINER_TEMP_DIR);
    vargs.add("-Djava.io.tmpdir=" + amTmpDir);
    MRApps.addLog4jSystemProperties(null, vargs, conf);

    // Check for Java Lib Path usage in MAP and REDUCE configs
   ...
    // Add AM admin command opts before user command opts
    // so that it can be overridden by user
    ...
    // Add AM user command opts
    //\DEFAULT_MR_AM_COMMAND_OPTS = "-Xmx1024m";
    String mrAppMasterUserOptions = conf.get(MRJobConfig.MR_AM_COMMAND_OPTS,
        MRJobConfig.DEFAULT_MR_AM_COMMAND_OPTS);
    warnForJavaLibPath(mrAppMasterUserOptions, "app master",
        MRJobConfig.MR_AM_COMMAND_OPTS, MRJobConfig.MR_AM_ENV);
    vargs.add(mrAppMasterUserOptions);

    if (jobConf.getBoolean(MRJobConfig.MR_AM_PROFILE,
        MRJobConfig.DEFAULT_MR_AM_PROFILE)) {
      final String profileParams = jobConf.get(MRJobConfig.MR_AM_PROFILE_PARAMS,
          MRJobConfig.DEFAULT_TASK_PROFILE_PARAMS);
      if (profileParams != null) {
        vargs.add(String.format(profileParams,
            ApplicationConstants.LOG_DIR_EXPANSION_VAR + Path.SEPARATOR
                + TaskLog.LogName.PROFILE));
      }
    }
	// public static final String APPLICATION_MASTER_CLASS =
    // "org.apache.hadoop.mapreduce.v2.app.MRAppMaster";
    vargs.add(MRJobConfig.APPLICATION_MASTER_CLASS);
    vargs.add("1>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR +
        Path.SEPARATOR + ApplicationConstants.STDOUT);
    vargs.add("2>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR +
        Path.SEPARATOR + ApplicationConstants.STDERR);
    return vargs;
  }
```
### 3.9.2 submitApplication()
```java
// ResourceMgrDelegate.java
public ApplicationId submitApplication(ApplicationSubmissionContext appContext) throws YarnException, IOException {
    return client.submitApplication(appContext);
}

// YarnClientImpl.java
public ApplicationId submitApplication(ApplicationSubmissionContext appContext) throws YarnException, IOException {
    // 提交请求
    SubmitApplicationRequest request =
        Records.newRecord(SubmitApplicationRequest.class);
    request.setApplicationSubmissionContext(appContext);

    // rmClient是ApplicationClientProtocol
    // ClientRMService处理客户端的请求
    rmClient.submitApplication(request);

    return applicationId;
  }
}

// ClientRMService.java
public SubmitApplicationResponse submitApplication(
      SubmitApplicationRequest request) throws YarnException, IOException {
    try {
      // call RMAppManager to submit application directly
      rmAppManager.submitApplication(submissionContext,
          System.currentTimeMillis(), userUgi);
          
      // 创建default队列
      if (submissionContext.getQueue() == null) {
      submissionContext.setQueue(YarnConfiguration.DEFAULT_QUEUE_NAME);    
    } catch (YarnException e) {...}
	// 返回一个SubmitApplicationResponse对象
    return recordFactory
        .newRecordInstance(SubmitApplicationResponse.class);
}

// RMAppManager.java
protected void submitApplication(
      ApplicationSubmissionContext submissionContext, long submitTime,
      UserGroupInformation userUgi) throws YarnException {
    ApplicationId applicationId = submissionContext.getApplicationId();

    // Passing start time as -1. It will be eventually set in RMAppImpl
    // constructor.
    // 创建RMAppImpl的对象
    RMAppImpl application = createAndPopulateNewRMApp(
        submissionContext, submitTime, userUgi, false, -1, null);
    try {
      if (UserGroupInformation.isSecurityEnabled()) {...} 
        else {
 		// 创建一个START类型的RMAppEvent交给EventHandler处理
        this.rmContext.getDispatcher().getEventHandler()
            .handle(new RMAppEvent(applicationId, RMAppEventType.START));
      }
    } catch (Exception e) {
      this.rmContext.getDispatcher().getEventHandler()
          .handle(new RMAppEvent(applicationId,
              RMAppEventType.APP_REJECTED, e.getMessage()));
      throw RPCUtil.getRemoteException(e);
    }
}
```
### 3.9.3 状态转换
#### 3.9.3.1 RMAppEventType.START
```java
// RMAppImpl中的Transition
.addTransition(RMAppState.NEW, RMAppState.NEW_SAVING,  
    RMAppEventType.START, new RMAppNewlySavingTransition())

private static final class RMAppNewlySavingTransition extends RMAppTransition {  
  @Override  
  public void transition(RMAppImpl app, RMAppEvent event) {  
    app.rmContext.getStateStore().storeNewApplication(app);  
  }  
}

public void storeNewApplication(RMApp app) {  
  ApplicationStateData appState =  ApplicationStateData.newInstance(...); 
  getRMStateStoreEventHandler().handle(new RMStateStoreAppEvent(appState));  
}

// 产生一个RMStateStoreEventType.STORE_APP事件
public RMStateStoreAppEvent(ApplicationStateData appState) {  
  super(RMStateStoreEventType.STORE_APP);  
  this.appState = appState;  
}
```
#### 3.9.3.2 RMStateStoreEventType.STORE_APP
```java
// RMStateStore.java中定义了状态机，注册了处理RMStateStoreEvent的Transition
.addTransition(RMStateStoreState.ACTIVE,  
    EnumSet.of(RMStateStoreState.ACTIVE, RMStateStoreState.FENCED),  
    RMStateStoreEventType.STORE_APP, new StoreAppTransition())
    
private static class StoreAppTransition  
    implements MultipleArcTransition<RMStateStore, RMStateStoreEvent,  
        RMStateStoreState> {  
  @Override  
  public RMStateStoreState transition(RMStateStore store,  
      RMStateStoreEvent event) {  
    try {  
      store.storeApplicationStateInternal(appId, appState);  
      store.notifyApplication(  
          new RMAppEvent(appId, RMAppEventType.APP_NEW_SAVED));  
    }
}
```
#### 3.9.3.3 RMAppEventType.APP_NEW_SAVED
```java
// RMAppImpl转为RMAppState.SUBMITTED状态
.addTransition(RMAppState.NEW_SAVING, RMAppState.SUBMITTED,  
    RMAppEventType.APP_NEW_SAVED, new AddApplicationToSchedulerTransition())

private static final class AddApplicationToSchedulerTransition extends  
    RMAppTransition {  
  @Override  
  public void transition(RMAppImpl app, RMAppEvent event) {  
    app.handler.handle(  
        new AppAddedSchedulerEvent(app.user, app.submissionContext, false, app.applicationPriority, app.placementContext));  
  }  
}

// 产生一个SchedulerEventType.APP_ADDED事件, 会被队列的调度器接收处理
public AppAddedSchedulerEvent(ApplicationId applicationId, String queue,  
    String user, boolean isAppRecovering, ReservationId reservationID,  
    Priority appPriority, ApplicationPlacementContext placementContext) { 
  super(SchedulerEventType.APP_ADDED);
}

// CapacityScheduler.java
public void handle(SchedulerEvent event) {  
  switch(event.getType()) {
  case APP_ADDED:  
{  
  if (queueName != null) {  
    if (!appAddedEvent.getIsAppRecovering()) {  
      addApplication(...);  
    } 
  }  
}  
break;

private void addApplication(ApplicationId applicationId, String queueName, String user, Priority priority,  
    ApplicationPlacementContext placementContext) {  
  writeLock.lock();  
  try {  
    // Submit to the queue  
    try {  
      queue.submitApplication(applicationId, user, queueName);  
    }  
    rmContext.getDispatcher().getEventHandler().handle(  
    new RMAppEvent(applicationId, RMAppEventType.APP_ACCEPTED));
  }
}
```
#### 3.9.3.4 RMAppEventType.APP_ACCEPTED
```java
// RMAppImpl
.addTransition(RMAppState.SUBMITTED, RMAppState.ACCEPTED,  
    RMAppEventType.APP_ACCEPTED, new StartAppAttemptTransition())

private static final class StartAppAttemptTransition extends RMAppTransition {  
  @Override  
  public void transition(RMAppImpl app, RMAppEvent event) {  
    app.createAndStartNewAttempt(false);  
  };  
}

private void  
    createAndStartNewAttempt(boolean transferStateFromPreviousAttempt) {  
  // 创建一个RMAppAttemptImpl对象
  createNewAttempt();  
  handler.handle(new RMAppStartAttemptEvent(currentAttempt.getAppAttemptId(),  
    transferStateFromPreviousAttempt));  
}

public RMAppStartAttemptEvent(ApplicationAttemptId appAttemptId,  
    boolean transferStateFromPreviousAttempt) {  
  super(appAttemptId, RMAppAttemptEventType.START);  
  this.transferStateFromPreviousAttempt = transferStateFromPreviousAttempt;  
}
```
#### 3.9.3.5 RMAppAttemptEventType.START
```java
// RMAppAttemptImpl
.addTransition(RMAppAttemptState.NEW, RMAppAttemptState.SUBMITTED,  
    RMAppAttemptEventType.START, new AttemptStartedTransition())

private static final class AttemptStartedTransition extends BaseTransition {  
@Override  
   public void transition(RMAppAttemptImpl appAttempt,  
       RMAppAttemptEvent event) {  
          
    appAttempt.eventHandler.handle(new AppAttemptAddedSchedulerEvent(  
       appAttempt.applicationAttemptId, transferStateFromPreviousAttempt));  
   }  
 }

public AppAttemptAddedSchedulerEvent(  
    ApplicationAttemptId applicationAttemptId,  
    boolean transferStateFromPreviousAttempt,  
    boolean isAttemptRecovering) {  
  super(SchedulerEventType.APP_ATTEMPT_ADDED); 
}
```
#### 3.9.3.6 SchedulerEventType.APP_ATTEMPT_ADDED
```java
// 容量调度器的switch：
case APP_ATTEMPT_ADDED:  
{  
  addApplicationAttempt(...);  
}  
break;

private void addApplicationAttempt(  
    ApplicationAttemptId applicationAttemptId,  
    boolean transferStateFromPreviousAttempt,  
    boolean isAttemptRecovering) {  
  writeLock.lock();  
  try {  
    SchedulerApplication<FiCaSchedulerApp> application = applications.get(  
        applicationAttemptId.getApplicationId());  
    
    CSQueue queue = (CSQueue) application.getQueue();  
  
    FiCaSchedulerApp attempt = new FiCaSchedulerApp(applicationAttemptId,  
        application.getUser(), queue, queue.getAbstractUsersManager(),  
        rmContext, application.getPriority(), isAttemptRecovering,  
        activitiesManager);  
 
    application.setCurrentAppAttempt(attempt);  

	// 提交attempt
    queue.submitApplicationAttempt(attempt, application.getUser());  
    if (isAttemptRecovering) {  
      LOG.debug("{} is recovering. Skipping notifying ATTEMPT_ADDED",  
          applicationAttemptId);  
    } else{  
      rmContext.getDispatcher().getEventHandler().handle(  
          new RMAppAttemptEvent(applicationAttemptId,  
              RMAppAttemptEventType.ATTEMPT_ADDED));  
    }  
  } finally {  
    writeLock.unlock();  
  }  
}
```
#### 3.9.3.7 RMAppAttemptEventType.ATTEMPT_ADDED
```java
// RMAppAttemptImpl
.addTransition(RMAppAttemptState.SUBMITTED,   
    EnumSet.of(RMAppAttemptState.LAUNCHED_UNMANAGED_SAVING,  
               RMAppAttemptState.SCHEDULED),  
    RMAppAttemptEventType.ATTEMPT_ADDED,  
    new ScheduleTransition())

public static final class ScheduleTransition  
    implements  
    MultipleArcTransition<RMAppAttemptImpl, RMAppAttemptEvent, RMAppAttemptState> {  
  @Override  
  public RMAppAttemptState transition(RMAppAttemptImpl appAttempt,  
      RMAppAttemptEvent event) {  
    ApplicationSubmissionContext subCtx = appAttempt.submissionContext;  
    if (!subCtx.getUnmanagedAM()) {  
      // 分配资源
      // AM resource has been checked when submission  
      Allocation amContainerAllocation =  
          appAttempt.scheduler.allocate(  
              appAttempt.applicationAttemptId,  
              appAttempt.amReqs, null, EMPTY_CONTAINER_RELEASE_LIST,  
              amBlacklist.getBlacklistAdditions(),  
              amBlacklist.getBlacklistRemovals(),  
              new ContainerUpdates());  
      return RMAppAttemptState.SCHEDULED;  
    } 
  }  
}
```
##### 3.9.3.7.1 RMContainerEventType.ACQUIRED
```java
// ResourceScheduler分配返回资源前，会向 RMContainerlmpl 发送一个 RMContainerEventType.ACQUIRED 事件

```
#### 3.9.3.8 RMAppAttemptState.SCHEDULED
```java
.addTransition(RMAppAttemptState.SCHEDULED,  
    EnumSet.of(RMAppAttemptState.ALLOCATED_SAVING,  
      RMAppAttemptState.SCHEDULED),  
    RMAppAttemptEventType.CONTAINER_ALLOCATED,  
    new AMContainerAllocatedTransition())

private static final class AMContainerAllocatedTransition  
    implements  
    MultipleArcTransition<RMAppAttemptImpl, RMAppAttemptEvent, RMAppAttemptState> {  
  @Override  
  public RMAppAttemptState transition(RMAppAttemptImpl appAttempt,  
      RMAppAttemptEvent event) {  
    // Acquire the AM container from the scheduler.  
    Allocation amContainerAllocation =  
        appAttempt.scheduler.allocate(appAttempt.applicationAttemptId,  
          EMPTY_CONTAINER_REQUEST_LIST, null, EMPTY_CONTAINER_RELEASE_LIST, null,  
          null, new ContainerUpdates());  
    // 还未获取到资源，保持SCHEDULED状态        
    if (amContainerAllocation.getContainers().size() == 0) {  
      appAttempt.retryFetchingAMContainer(appAttempt);  
      return RMAppAttemptState.SCHEDULED;  
    }  
  
    // Set the masterContainer  
    Container amContainer = amContainerAllocation.getContainers().get(0);  
    RMContainerImpl rmMasterContainer = (RMContainerImpl)appAttempt.scheduler  
        .getRMContainer(amContainer.getId());  

    if (rmMasterContainer == null) {  
      return RMAppAttemptState.SCHEDULED;  
    }  
    appAttempt.setMasterContainer(amContainer);  
    rmMasterContainer.setAMContainer(true); 
     
    return RMAppAttemptState.ALLOCATED_SAVING;  
  }  
}

public Allocation allocate(ApplicationAttemptId applicationAttemptId,
                           List<ResourceRequest> ask, List<SchedulingRequest> schedulingRequests, List<ContainerId> release, List<String> blacklistAdditions, List<String> blacklistRemovals, ContainerUpdates updateRequests) {
    synchronized (application) {
        Resource headroom = application.getHeadroom();
        application.setApplicationHeadroomForMetrics(headroom);
        return new Allocation(application.pullNewlyAllocatedContainers(),
           headroom, null, null, null, application.pullUpdatedNMTokens());
    }
}

// SchedulerApplicationAttempt.pullNewlyAllocatedContainers
public List<Container> pullNewlyAllocatedContainers() {
    writeLock.lock();
    try {
        List<Container> returnContainerList = new ArrayList<Container>(
            newlyAllocatedContainers.size());

        Iterator<RMContainer> i = newlyAllocatedContainers.iterator();
        while (i.hasNext()) {
            RMContainer rmContainer = i.next();
            // 为新分配空间的container更新状态
            Container updatedContainer =
            updateContainerAndNMToken(rmContainer, null);
        }
        return returnContainerList;
    } finally {
        writeLock.unlock();
    }
}

private Container updateContainerAndNMToken(RMContainer rmContainer,
      ContainerUpdateType updateType) {
    Container container = rmContainer.getContainer();
    ContainerType containerType = ContainerType.TASK;

    if (updateType == null) {
      // This is a newly allocated container
      rmContainer.handle(new RMContainerEvent(
          rmContainer.getContainerId(), RMContainerEventType.ACQUIRED));
    }
    return container;
}
```
##### 3.9.3.8.1 RMContainerEventType.ACQUIRED
```java
.addTransition(RMContainerState.ALLOCATED, RMContainerState.ACQUIRED,  
    RMContainerEventType.ACQUIRED, new AcquiredTransition())

private static final class AcquiredTransition extends BaseTransition {  
  
  @Override  
  public void transition(RMContainerImpl container, RMContainerEvent event) {  
    
    // Tell the app  
    // 添加到node后，返回RMAppAttemptState.ALLOCATED_SAVING
    container.eventHandler.handle(new RMAppRunningOnNodeEvent(container  
        .getApplicationAttemptId().getApplicationId(), container.nodeId));  
  }  
}
```
#### 3.9.3.9 RMAppAttemptState.ALLOCATED_SAVING
```java
.addTransition(RMAppAttemptState.ALLOCATED_SAVING,   
    RMAppAttemptState.ALLOCATED,  
    RMAppAttemptEventType.ATTEMPT_NEW_SAVED, new AttemptStoredTransition())

private static final class AttemptStoredTransition extends BaseTransition {  
  @Override  
  public void transition(RMAppAttemptImpl appAttempt,  
     RMAppAttemptEvent event) {  

    appAttempt.launchAttempt();  
  }  
}

private void launchAttempt(){  
  // Send event to launch the AM Container  
  eventHandler.handle(new AMLauncherEvent(AMLauncherEventType.LAUNCH, this));  
}
```
#### 3.9.3.10 AMLauncherEventType.LAUNCH
```java
// AMLauncher.java
public void run() {  
  switch (eventType) {  
  case LAUNCH:  
    try {  
      LOG.info("Launching master" + application.getAppAttemptId());  
      launch();  
      handler.handle(new RMAppAttemptEvent(application.getAppAttemptId(), 
          RMAppAttemptEventType.LAUNCHED, System.currentTimeMillis()));  
    } catch(Exception ie) {  
      onAMLaunchFailed(masterContainer.getId(), ie);  
    }  
    break;
}

// 启动AM
private void launch() throws IOException, YarnException {  
  // 获取AM和NM的通信协议
  connect();  
  // 创建启动AM的上下文
  ContainerLaunchContext launchContext =  
      createAMContainerLaunchContext(applicationContext, masterContainerID); 
  // 启动AMContainer
  StartContainersResponse response =  
      containerMgrProxy.startContainers(allRequests);  
}

// ContainerManagerImpl.java
public StartContainersResponse startContainers(  
    StartContainersRequest requests) throws YarnException, IOException {  

    for (StartContainerRequest request : requests  
        .getStartContainerRequests()) {  
      ContainerId containerId = null;  
      try {  
        startContainerInternal(containerTokenIdentifier, request,  
            remoteUser);  
        succeededContainers.add(containerId);  
      }  
    }  
    return StartContainersResponse  
        .newInstance(getAuxServiceMetaData(), succeededContainers,  
            failedContainers);  
  }
}

protected void startContainerInternal(  
    ContainerTokenIdentifier containerTokenIdentifier,  
    StartContainerRequest request, String remoteUser)  
    throws YarnException, IOException {  
  
  try {  
    if (!isServiceStopped()) {  
      if (!context.getApplications().containsKey(applicationID)) {  
        // 创建App
        Application application =  
            new ApplicationImpl(dispatcher, user, flowContext,  
                applicationID, credentials, context);    
          dispatcher.getEventHandler().handle(new ApplicationInitEvent(  
              applicationID, appAcls, logAggregationContext));  
        }  
      } 
	  // 启动容器
      this.context.getNMStateStore().storeContainer(containerId,  
          containerTokenIdentifier.getVersion(), containerStartTime, request);  
      dispatcher.getEventHandler().handle(  
        new ApplicationContainerInitEvent(container));  
 }
}
```
##### 3.9.3.10.1 RMAppAttemptEventType.LAUNCHED
```java
// run()方法调用完launch()方法后，发送RMAppAttemptEventType.LAUNCHED事件
.addTransition(RMAppAttemptState.ALLOCATED, RMAppAttemptState.LAUNCHED,  
    RMAppAttemptEventType.LAUNCHED, LAUNCHED_TRANSITION)
// 从ALLOCATED转为LAUNCHED
```
#### 3.9.3.11 ApplicationEventType.INIT_APPLICATION
```java
public ApplicationInitEvent(ApplicationId appId,  
    Map<ApplicationAccessType, String> acls,  
    LogAggregationContext logAggregationContext) {  
  super(appId, ApplicationEventType.INIT_APPLICATION);  
}

.addTransition(ApplicationState.NEW, ApplicationState.INITING,  
    ApplicationEventType.INIT_APPLICATION, new AppInitTransition())

static class AppInitTransition implements  
    SingleArcTransition<ApplicationImpl, ApplicationEvent> {  
  @Override  
  public void transition(ApplicationImpl app, ApplicationEvent event) {  
    app.dispatcher.getEventHandler().handle(  
        new LogHandlerAppStartedEvent(app.appId, app.user,  
            app.credentials, app.applicationACLs,  
            app.logAggregationContext, app.applicationLogInitedTimestamp));  
  }  
}
```
#### 3.9.3.12 LogHandlerEventType.APPLICATION_STARTED
```java
public LogHandlerAppStartedEvent(ApplicationId appId, String user,  
    Credentials credentials, Map<ApplicationAccessType, String> appAcls,  
    LogAggregationContext logAggregationContext, long appLogInitedTime) {  
  super(LogHandlerEventType.APPLICATION_STARTED);  
}

// LogAggregationService.java
public void handle(LogHandlerEvent event) {
    switch (event.getType()) {
      case APPLICATION_STARTED:
        initApp();
        break;
    }
}

private void initApp(final ApplicationId appId, String user,  
    Credentials credentials, Map<ApplicationAccessType, String> appAcls,  
    LogAggregationContext logAggregationContext,  
    long recoveredLogInitedTime) {  
  try {   
    eventResponse = new ApplicationEvent(appId,  
        ApplicationEventType.APPLICATION_LOG_HANDLING_INITED);  
  }
  this.dispatcher.getEventHandler().handle(eventResponse);  
}
```
#### 3.9.3.13 ApplicationEventType.APPLICATION_LOG_HANDLING_INITED
```java
.addTransition(ApplicationState.INITING, ApplicationState.INITING,  
    ApplicationEventType.APPLICATION_LOG_HANDLING_INITED,  
    new AppLogInitDoneTransition())

static class AppLogInitDoneTransition implements  
    SingleArcTransition<ApplicationImpl, ApplicationEvent> {  
  @Override  
  public void transition(ApplicationImpl app, ApplicationEvent event) {  
    app.dispatcher.getEventHandler().handle(  
        new ApplicationLocalizationEvent(  
            LocalizationEventType.INIT_APPLICATION_RESOURCES, app));  
  }  
}
```
#### 3.9.3.14 LocalizationEventType.INIT_APPLICATION_RESOURCES
```java
// ResourceLocalizationService.java
public void handle(LocalizationEvent event) {  
  // TODO: create log dir as $logdir/$user/$appId  
  switch (event.getType()) {  
  case INIT_APPLICATION_RESOURCES:  
    handleInitApplicationResources(  
        ((ApplicationLocalizationEvent)event).getApplication());  
    break;
    }
}

private void handleInitApplicationResources(Application app) {  
  dispatcher.getEventHandler().handle(new ApplicationInitedEvent(  
        app.getAppId()));  
}
```
#### 3.9.3.15 ApplicationEventType.APPLICATION_INITED
```java
public ApplicationInitedEvent(ApplicationId appID) {  
  super(appID, ApplicationEventType.APPLICATION_INITED);  
}

.addTransition(ApplicationState.INITING, ApplicationState.RUNNING,  
    ApplicationEventType.APPLICATION_INITED,  
    new AppInitDoneTransition())
    
static class AppInitDoneTransition implements  
    SingleArcTransition<ApplicationImpl, ApplicationEvent> {  
  @Override  
  public void transition(ApplicationImpl app, ApplicationEvent event) {  
    // Start all the containers waiting for ApplicationInit  
    for (Container container : app.containers.values()) {  
      app.dispatcher.getEventHandler().handle(new ContainerInitEvent(  
            container.getContainerId()));  
    }  
  }  
}
```
#### 3.9.3.16 ContainerEventType.INIT_CONTAINER
```java
public ContainerInitEvent(ContainerId c) {  
  super(c, ContainerEventType.INIT_CONTAINER);  
}

// containerImpl
.addTransition(ContainerState.NEW,  
    EnumSet.of(ContainerState.LOCALIZING,  
        ContainerState.SCHEDULED,  
        ContainerState.LOCALIZATION_FAILED,  
        ContainerState.DONE),  
    ContainerEventType.INIT_CONTAINER, new RequestResourcesTransition())

static class RequestResourcesTransition implements  
    MultipleArcTransition<ContainerImpl,ContainerEvent,ContainerState> {  
  @Override  
  public ContainerState transition(ContainerImpl container,  
      ContainerEvent event) {  

    Map<String, LocalResource> cntrRsrc;  
    try {  
      // 获取本地资源
      cntrRsrc = container.context  
          .getContainerExecutor().getLocalResources(container);  
      if (!cntrRsrc.isEmpty()) {  
        Map<LocalResourceVisibility, Collection<LocalResourceRequest>> req = container.resourceSet.addResources(ctxt.getLocalResources());  
        container.dispatcher.getEventHandler().handle(  
            new ContainerLocalizationRequestEvent(container, req));  
        return ContainerState.LOCALIZING;  
      }
    }  
  }  
}
```
##### 3.9.3.16.1 (LocalizationEventType.LOCALIZE_CONTAINER_RESOURCES
```java
public ContainerLocalizationRequestEvent(Container c,  
    Map<LocalResourceVisibility, Collection<LocalResourceRequest>> rsrc) {  
  super(LocalizationEventType.LOCALIZE_CONTAINER_RESOURCES, c);  
  this.rsrc = rsrc;  
}

public void handle(LocalizationEvent event) {  
  // TODO: create log dir as $logdir/$user/$appId  
  switch (event.getType()) {  
  case LOCALIZE_CONTAINER_RESOURCES:  
    handleInitContainerResources((ContainerLocalizationRequestEvent) event);  
    break;
}

// 通过LocalResourcesTrackerImpl分配资源
```
##### 3.9.3.16.2 ContainerState.LOCALIZING
```java
.addTransition(ContainerState.LOCALIZING,  
    EnumSet.of(ContainerState.LOCALIZING, ContainerState.SCHEDULED),  
    ContainerEventType.RESOURCE_LOCALIZED, new LocalizedTransition())

static class LocalizedTransition implements  
    MultipleArcTransition<ContainerImpl,ContainerEvent,ContainerState> {  

  public ContainerState transition(ContainerImpl container,  
      ContainerEvent event) {  

    container.dispatcher.getEventHandler().handle(  
        new ContainerLocalizationEvent(LocalizationEventType.  
            CONTAINER_RESOURCES_LOCALIZED, container));  
  
    container.sendScheduleEvent();  
    container.metrics.endInitingContainer();  
  
    return ContainerState.SCHEDULED;  
  }  
}
```
#### 3.9.3.17 ContainerSchedulerEventType.SCHEDULE_CONTAINER
```java
private void sendScheduleEvent() {  
  if (recoveredStatus == RecoveredContainerStatus.PAUSED) {  
  } else {  
    dispatcher.getEventHandler().handle(new ContainerSchedulerEvent(this, 
        ContainerSchedulerEventType.SCHEDULE_CONTAINER));  
  }  
}

// ContainerScheduler.java
public void handle(ContainerSchedulerEvent event) {  
  switch (event.getType()) {  
  case SCHEDULE_CONTAINER:  
    // 加入队列调用startPendingContainers()
    scheduleContainer(event.getContainer());  
    break;
}
```
#### 3.9.3.18 startPendingContainers()
```java
private void startPendingContainers(boolean forceStartGuaranteedContaieners) {  
  // Start guaranteed containers that are paused, if resources available.  
  boolean resourcesAvailable = startContainers(  
        queuedGuaranteedContainers.values(), forceStartGuaranteedContaieners);  
  // Start opportunistic containers, if resources available.  
  if (resourcesAvailable) {  
    startContainers(queuedOpportunisticContainers.values(), false);  
  }  
}

private boolean startContainers(  
    Collection<Container> containersToBeStarted, boolean force) {  
  Iterator<Container> cIter = containersToBeStarted.iterator();  
  boolean resourcesAvailable = true;  
  while (cIter.hasNext() && resourcesAvailable) {  
    Container container = cIter.next();  
    if (tryStartContainer(container, force)) {  
      cIter.remove();  
    } else {  
      resourcesAvailable = false;  
    }  
  }  
  return resourcesAvailable;  
}

private boolean tryStartContainer(Container container, boolean force) {  
  boolean containerStarted = false;  
  // call startContainer without checking available resource when force==true  
  if (force || resourceAvailableToStartContainer(  
      container)) {  
    startContainer(container);  
    containerStarted = true;  
  }  
  return containerStarted;  
}

private void startContainer(Container container) {  
  container.sendLaunchEvent();  
}

public void sendLaunchEvent() {  
  if (ContainerState.PAUSED == getContainerState()) {
  } else {  
    ContainersLauncherEventType launcherEvent =  
        ContainersLauncherEventType.LAUNCH_CONTAINER;  
    if (recoveredStatus == RecoveredContainerStatus.LAUNCHED) {  
      // try to recover a container that was previously launched  
      launcherEvent = ContainersLauncherEventType.RECOVER_CONTAINER;  
    }
  
    containerLaunchStartTime = clock.getTime();  
    dispatcher.getEventHandler().handle(  
        new ContainersLauncherEvent(this, launcherEvent));  
  }  
}
```
#### 3.9.3.19 ContainersLauncherEventType.LAUNCH_CONTAINER
```java
public void handle(ContainersLauncherEvent event) {  
  // TODO: ContainersLauncher launches containers one by one!!  
  Container container = event.getContainer();  
  ContainerId containerId = container.getContainerId();  
  switch (event.getType()) {  
    case LAUNCH_CONTAINER:   
	  // 新建一个ContainerLaunch对象，放入ContainerLaunch线程池对象中
      ContainerLaunch launch =  
          new ContainerLaunch(context, getConfig(), dispatcher, exec, app, event.getContainer(), dirsHandler, containerManager);  
      containerLauncher.submit(launch);  
      running.put(containerId, launch);  
      break;
}

// ContainerLaunch继承了Callable，运行方法在call中
public Integer call() {  
  
  final ContainerLaunchContext launchContext = container.getLaunchContext();  
  
  final List<String> command = launchContext.getCommands();  
  int ret = -1;  
  
  try {  
    ret = launchContainer(...);  
  } 
  return ret;  
}

protected int launchContainer(ContainerStartContext ctx)  
    throws IOException, ConfigurationException {  
  int launchPrep = prepareForLaunch(ctx);  
  if (launchPrep == 0) {  
    launchLock.lock();  
    try {  
      return exec.launchContainer(ctx);  
    } finally {  
      launchLock.unlock();  
    }  
  }  
  return launchPrep;  
}
```
### 3.9.4 启动 MRAppMaster
#### 3.9.4.1 main()
```java
public static void main(String[] args) {  
  try { 
    ContainerId containerId = ContainerId.fromString(containerIdStr);  
    ApplicationAttemptId applicationAttemptId =  
        containerId.getApplicationAttemptId();  
    if (applicationAttemptId != null) {  
      CallerContext.setCurrent(new CallerContext.Builder(  
          "mr_appmaster_" + applicationAttemptId.toString()).build());  
    }  
    long appSubmitTime = Long.parseLong(appSubmitTimeStr);  
      
    // 创建AM对象
    MRAppMaster appMaster =  
        new MRAppMaster(applicationAttemptId, containerId, nodeHostString, Integer.parseInt(nodePortString),  
		Integer.parseInt(nodeHttpPortString), appSubmitTime);  
	
    initAndStartAppMaster(appMaster, conf, jobUserName);  
  }   
}

protected static void initAndStartAppMaster(final MRAppMaster appMaster,  
    final JobConf conf, String jobUserName) throws IOException,  
    InterruptedException {  
  appMasterUgi.doAs(new PrivilegedExceptionAction<Object>() {  
    @Override  
    public Object run() throws Exception {  
      appMaster.init(conf);  
      appMaster.start();  
      if(appMaster.errorHappenedShutDown) {  
        throw new IOException("Was asked to shut down.");  
      }  
      return null;  
    }  
  });  
}
```
#### 3.9.4.2 ServiceInit()
```java
protected void serviceInit(final Configuration conf) throws Exception {  
  // create the job classloader if enabled  
  createJobClassLoader(conf);  

  dispatcher = createDispatcher();  
  jobId = MRBuilderUtils.newJobId(appAttemptID.getApplicationId(),  
      appAttemptID.getApplicationId().getId());  
  int numReduceTasks = conf.getInt(MRJobConfig.NUM_REDUCES, 0);  
  if ((numReduceTasks > 0 &&   
      conf.getBoolean("mapred.reducer.new-api", false)) ||  
        (numReduceTasks == 0 &&   
         conf.getBoolean("mapred.mapper.new-api", false)))  {  
    newApiCommitter = true;  
    LOG.info("Using mapred newApiCommitter.");  
  }  
    // service to allocate containers from RM (if non-uber) or to fake it (uber)  
    containerAllocator = createContainerAllocator(null, context);  
    addIfService(containerAllocator);  
    dispatcher.register(ContainerAllocator.EventType.class, containerAllocator);  
  
    //service to handle requests from JobClient  
    clientService = createClientService(context);  
    clientService.init(conf);  
      
    containerAllocator = createContainerAllocator(clientService, context);  
      
    this.jobEventDispatcher = new JobEventDispatcher();  
  
    //register the event dispatchers  
    dispatcher.register(JobEventType.class, jobEventDispatcher);  
    dispatcher.register(TaskEventType.class, new TaskEventDispatcher());  
    dispatcher.register(TaskAttemptEventType.class,   
        new TaskAttemptEventDispatcher());  
    dispatcher.register(CommitterEventType.class, committerEventHandler); 
    
    addIfService(containerAllocator);  
    dispatcher.register(ContainerAllocator.EventType.class, containerAllocator);  
  
    // corresponding service to launch allocated containers via NodeManager  
    containerLauncher = createContainerLauncher(context);  
    addIfService(containerLauncher);  
    dispatcher.register(ContainerLauncher.EventType.class, containerLauncher);  
  }  
  super.serviceInit(conf);  
} // end of init()
```
#### 3.9.4.3 serviceStart()
```java
protected void serviceStart() throws Exception {  
  
  // 创建job对象
  job = createJob(getConfig(), forcedState, shutDownMessage);  
  
  // 通知AM启动
    dispatcher.getEventHandler().handle(  
        new JobHistoryEvent(job.getID(), new AMStartedEvent(info  
            .getAppAttemptId(), info.getStartTime(), info.getContainerId(),  
            info.getNodeManagerHost(), info.getNodeManagerPort(), info  
                .getNodeManagerHttpPort(), appSubmitTime)));  
  }  
  
  // Send out an MR AM inited event for this AM.  
  dispatcher.getEventHandler().handle(  
      new JobHistoryEvent(job.getID(), new AMStartedEvent(amInfo  
          .getAppAttemptId(), amInfo.getStartTime(), amInfo.getContainerId(),  
          amInfo.getNodeManagerHost(), amInfo.getNodeManagerPort(), amInfo  
              .getNodeManagerHttpPort(), this.forcedState == null ? null  
                  : this.forcedState.toString(), appSubmitTime)));  
  amInfos.add(amInfo);  

  if (initFailed) {  
    JobEvent initFailedEvent = new JobEvent(job.getID(), JobEventType.JOB_INIT_FAILED);  
    jobEventDispatcher.handle(initFailedEvent);  
  } else {  
    // All components have started, start the job. 
    // 启动Job 
    startJobs();  
  }  
}

protected void startJobs() {  
  /** create a job-start event to get this ball rolling */  
  JobEvent startJobEvent = new JobStartEvent(job.getID(),  
      recoveredJobStartTime);  
  /** send the job-start event. this triggers the job execution. */  
  dispatcher.getEventHandler().handle(startJobEvent);  
}

// GenericEventHandler将JobEventType.JOB_START事件放入队列中
public JobStartEvent(JobId jobID, long recoveredJobStartTime) {  
  super(jobID, JobEventType.JOB_START);  
  this.recoveredJobStartTime = recoveredJobStartTime;  
}
```
#### 3.9.4.4 JobEventType.JOB_START
```java
// JobImpl中的状态机进行处理
.addTransition(JobStateInternal.INITED, JobStateInternal.SETUP,  
    JobEventType.JOB_START,  
    new StartTransition())

public static class StartTransition  
implements SingleArcTransition<JobImpl, JobEvent> {  
  /**  
   * This transition executes in the event-dispatcher thread, though it's   * triggered in MRAppMaster's startJobs() method.   */  
  @Override  
  public void transition(JobImpl job, JobEvent event) {  
    JobStartEvent jse = (JobStartEvent) event;  
    job.eventHandler.handle(new CommitterJobSetupEvent(  
            job.jobId, job.jobContext));  
  }  
}

public CommitterJobSetupEvent(JobId jobID, JobContext jobContext) {  
  super(CommitterEventType.JOB_SETUP);  
  this.jobID = jobID;  
  this.jobContext = jobContext;  
}
```
#### 3.9.4.5 CommitterEventType.JOB_SETUP
```java
// CommitterEventHandler.java
public void run() {  
  switch (event.getType()) {  
  case JOB_SETUP:  
    handleJobSetup((CommitterJobSetupEvent) event);  
    break;
}

protected void handleJobSetup(CommitterJobSetupEvent event) {  
  try {  
    committer.setupJob(event.getJobContext());  
    context.getEventHandler().handle(  
        new JobSetupCompletedEvent(event.getJobID()));  
  }  
}

public JobSetupCompletedEvent(JobId jobID) {  
  super(jobID, JobEventType.JOB_SETUP_COMPLETED);  
}
```
#### 3.9.4.6 obEventType.JOB_SETUP_COMPLETED
```java
.addTransition(JobStateInternal.SETUP, JobStateInternal.RUNNING,  
    JobEventType.JOB_SETUP_COMPLETED,  
    new SetupCompletedTransition())

private static class SetupCompletedTransition  
    implements SingleArcTransition<JobImpl, JobEvent> {  
  @Override  
  public void transition(JobImpl job, JobEvent event) {  
    job.setupProgress = 1.0f;  
    job.scheduleTasks(job.mapTasks, job.numReduceTasks == 0);  
    job.scheduleTasks(job.reduceTasks, true);  
  
    // If we have no tasks, just transition to job completed  
    if (job.numReduceTasks == 0 && job.numMapTasks == 0) {  
      job.eventHandler.handle(new JobEvent(job.jobId,  
          JobEventType.JOB_COMPLETED));  
    }  
  }  
}

protected void scheduleTasks(Set<TaskId> taskIDs,  
    boolean recoverTaskOutput) {  
  for (TaskId taskID : taskIDs) {  
    TaskInfo taskInfo = completedTasksFromPreviousRun.remove(taskID);  
    if (taskInfo != null) {} else {  
      eventHandler.handle(new TaskEvent(taskID, TaskEventType.T_SCHEDULE));  
    }  
  }  
}
```
#### 3.9.4.7 TaskEventType.T_SCHEDULE
```java
// TaskImpl
.addTransition(TaskStateInternal.NEW, TaskStateInternal.SCHEDULED,   
    TaskEventType.T_SCHEDULE, new InitialScheduleTransition())

private static class InitialScheduleTransition  
  implements SingleArcTransition<TaskImpl, TaskEvent> {  
  
  @Override  
  public void transition(TaskImpl task, TaskEvent event) {  
    task.addAndScheduleAttempt(Avataar.VIRGIN);  
    task.scheduledTime = task.clock.getTime();  
    task.sendTaskStartedEvent();  
  }  
}

private void addAndScheduleAttempt(Avataar avataar, boolean reschedule) {  
  TaskAttempt attempt = addAttempt(avataar);  
  inProgressAttempts.add(attempt.getID());  
  //schedule the nextAttemptNumber  
  if (failedAttempts.size() > 0 || reschedule) {} 
  else {  
    eventHandler.handle(new TaskAttemptEvent(attempt.getID(),  
        TaskAttemptEventType.TA_SCHEDULE));  
  }  
}
```
#### 3.9.4.8 TaskAttemptEventType.TA_SCHEDULE
```java
// TaskAttemptImpl
.addTransition(TaskAttemptStateInternal.NEW, TaskAttemptStateInternal.UNASSIGNED,  
    TaskAttemptEventType.TA_SCHEDULE, new RequestContainerTransition(false))

static class RequestContainerTransition implements  
    SingleArcTransition<TaskAttemptImpl, TaskAttemptEvent> {  
  private final boolean rescheduled;  
  public RequestContainerTransition(boolean rescheduled) {  
    this.rescheduled = rescheduled;  
  }  
  @SuppressWarnings("unchecked")  
  @Override  
  public void transition(TaskAttemptImpl taskAttempt,   
      TaskAttemptEvent event) {  
    //request for container  
    if (rescheduled) {} 
    else {  
      taskAttempt.eventHandler.handle(new ContainerRequestEvent(...);  
    }  
  }  
}

public ContainerRequestEvent(TaskAttemptId attemptID,   
    Resource capability,  
    String[] hosts, String[] racks) {  
  super(attemptID, ContainerAllocator.EventType.CONTAINER_REQ);  
  this.capability = capability;  
  this.hosts = hosts;  
  this.racks = racks;  
}									  
```
#### 3.9.4.9 ContainerAllocator.EventType.CONTAINER_REQ
```java
// RMContainerAllocator收到事件后，会放在队列中
// 之后通过handleEvent方法来处理
  
  recalculateReduceSchedule = true;  
  if (event.getType() == ContainerAllocator.EventType.CONTAINER_REQ) {  
    ContainerRequestEvent reqEvent = (ContainerRequestEvent) event;  
    boolean isMap = reqEvent.getAttemptID().getTaskId().getTaskType().  
        equals(TaskType.MAP);  
    if (isMap) {  
      handleMapContainerRequest(reqEvent);  
    } else {  
      handleReduceContainerRequest(reqEvent);  
    }  
  }
}
// 给任务分配到资源后启动YarnChild
```
### 3.9.5 启动 YarnChild
```java
public static void main(String[] args) throws Throwable {  
	// job.xml
	final JobConf job = new JobConf(MRJobConfig.JOB_CONF_FILE);
	
	task = myTask.getTask();  
	YarnChild.taskid = task.getTaskID();  
	  
	// Create the job-conf and set credentials  
	configureTask(job, task, credentials, jt);

	final Task taskFinal = task;  
childUGI.doAs(new PrivilegedExceptionAction<Object>() {  
  @Override  
  // MapTask|ReduceTask的run()方法
  public Object run() throws Exception {  
    taskFinal.run(job, umbilical); // run the task  
    return null;  
  }  
});
}
```
# 4 Shell 操作
- `hadoop fs` 或 `hdfs dfs`
```shell
# 追加数据到文件
-appendToFIle <localsrc> <dst>

# 移动文件到指定目录
-mv <src> <dst>

# 从本地上传文件
-put [-f] <localsrc> <dst>
-f: 覆盖目标文件

# 下载文件
-get [-f] [-p] <src> <localdst>
```
# 5 复习
# 6 面试
- 1 如何检测 `Hadoop` 集群的健康状态
	- 执行命令
		- `hdfs dfsadmin -report`：提供了 `HDFS` 的健康状态报告，包括数据节点的状态和块的复制因子
		- `yarn node -list -all`：查看所有 `NodeManager` 的状态
		- `hdfs dfsadmin -safemode get`：检查是否处于安全模式
	- 通过 `Prometheus + Grafana` 监控
- 2 Hadoop 高可用
	- `NameNode` 高可用
		- core-site.xml
			- `fs.defaultFS -> hdfs://{group_name}`
			- `ha.zookeeper.quarum: zookeeper` 
		- hdfs-site.xml
			- `dfs.nameservices nn 组名称`
			- `dfs.ha.namenodes.{group_name} -> nn1,nn2`
			- `dfs.client.failover.proxy.provider.{group_name} -> org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider`
	- `ResourceManager` 高可用
		- yarn-site.xml
			- `yarn.resourcemanager.ha.enabled -> true`
			- `yarn.resourcemanager.recovery.enabled -> true`
			- `yarn.resourcemanager.store.class -> org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore` 数据状态保持介质
			- `yarn.resourcemanager.zk-address`
			- `yarn.resourcemanager.cluster-id`
			- `yarn.resourcemanager.ha.rm-ids -> rm1, rm2`
			- `yarn.resourcemanager.hostname.rm1 -> hadoop102`
- 3 Hadoop 的详细加载顺序
	- `core-default.xml -> core-site.xml -> hdfs-default.xml -> hdfs-site.xml`
	- `core-default.xml -> core-site.xml -> mapred-default.xml -> mapred-site.xml`
- 4 NameNode 加载哪些信息，启动过程中，元数据信息保存在哪里
	- 初始化 `FSNameSystem`，加载 `FSImage` 和 `Edits` 文件
	- 元数据存储位置由配置 `dfs.namenode.name.dir` 决定
- 5 Hadoop3 的纠删码
	- 3 + 2 模式：3 个数据单元，2 个校验单元
	- 任意两个单元挂掉，通过其他单元仍然能够恢复出挂掉的单元
	- 通过因特尔 `ISA-L` 存储加速库可以提升 `HDFS` 纠删码的编码和解码效率
 - 6 Hadoop 读写流程数据同步如何实现
	 - 写
		 - 契约机制
		 - 发送的 `packet` 放在 `dataQueue` 和 `ackQueue` 中，只有所有节点都发送成功时，才会移除 `ackQueue` 中的 `packet`
	- 读
		- 读取到的 `packet` 先在本地缓存，再写入目标文件
- 7 Hadoop 数据校验
	- 读取文件时，`DN` 产生一个校验文件和之前上传的校验文件进行对比
	- 上传文件时，本地会通过 `crc-32算法` 生成一个校验文件，`HDFS` 接收数据后也会产生校验文件进行对比，相同则会存储
 - 8 Hadoop 挂掉怎么处理
 - 9 Hadoop 中的 shuffle 中反向溢写为什么不能设置成 100%
	 - 避免 map 操作的阻塞，预留 20% 的空间
- 10 Hadoop 怎么保证副本数据一致性
	- 副本机制：每个数据块都会被复制到多个节点进行保存
	- 主节点定期与数据节点进行心跳同步数据
	- 写入和读取的一致性
- 11 Hadoop 节点退役
	- 设置黑名单来实现退役
		- 在 `/etc/hadoop` 下创建 `dfs.hosts.exclude` 文件，添加黑名单节点
		- 在 `hdfs-site.xml` 中增加 `dfs.hosts.exclude` 指向其路径
		- 刷新 `NameNode`：`hdfs dfsadmin -refreshNodes`
		- 更新 `RM` 节点：`yarn rmadmin -refreshNodes`
- 12 HDFS 磁盘均衡
	- 生成均衡计划：`hdfs diskbalancer -plan host_name`
	- 执行：`hdfs diskbalancer -execute host_name.plan.json`
	- 查看执行情况：`hdfs diskbalancer -query host_name`
- 13 HDFS 能不能并发写入？如何解决并发问题的
	- 不支持并发写
- 14 为什么是文件块大小是128m？HDFS 中平均寻址的时间？
	- `HDFS` 平均寻址时间大概为 `10ms`
	- 寻址时间为传输时间的 `1%` 时为最佳状态
	- 磁盘的传输速率普遍为 `100MB/s`
	- 最佳 `block` 的大小：`100MB/s * 1s = 100MB`
 - 15 HDFS 文件系统缺点
	 - 不适合低延迟数据访问
	 - 无法高效对大量小文件进行存储
	 - 不支持并发写入和随机修改
- 16 提交不同的应用到 Hdfs，有的要2副本，有的要3副本，怎么指定
- 17 HDFS 什么情况下会进入安全模式
	- 安全模式只接受读请求
	- 部分 `DataNode` 启动失败或因网络原因与 `NameNode` 心跳失败
	- 部分 `DataNode` 节点磁盘损坏，导致数据无法读取
	- 部分 `DataNode` 节点磁盘使用过多，导致数据无法正常读取
- 18 HDFS 的 DataNode 挂了怎么办
	- `HDFS` 会自动处理 `DataNode` 宕机，会在宕机的 `DataNode` 上复制数据块的副本到其他正常的 `DataNode`
- 19 RocksDB 和 HDFS 有什么区别
	- RocksDB 是 KV 类型的数据库，HDFS 是文件系统
- 20 HDFS 回收站
	- 默认回收站不开启
	- 修改 core-site.xml
		- `fs.trash.interval` 单位 `min`，为 0 时表示禁用
		- `fs.trash.checkpoint.interval` 创建 `ck` 的间隔
	- 删除的文件会进入 `/user/${user.name}/.Trash/` 目录
- 21 HDFS 存储格式
	- SequenceFile
		- 存储 `KV` 的二进制文件
		- 由一个 `Header` 和多个 `Record` 组成
		- 通常作为中间数据存储格式
	- Text
		- 纯文本，行式存储
		- 不支持块级别压缩
	- ORC(Optimized Row Columnar)
		- 支持多种压缩方式，可切分
		- 以二进制方式存储，不可以直接读取
	- Parquet
		- 支持嵌套结构的列式存储格式
		- 适用于 `OLAP` 场景，按列存储和扫描
- 22 HDFS 修改文件名是不是一个原子性过程
	- HDFS 创建文件，删除文件，重命名，创建目录都是原子性操作
- 23 HDFS 写过程中，NN 挂掉会发生什么？DN ？客户端 ?
	- NN 挂了
		- 一个 `Block` 写完向 `NN` 报告时，报告失败，但不影响 `DN` 的工作，数据会被保存下来，最后 `Client` 向 `NN` 请求关闭时会出错
	- DN 挂了
		- `NN` 会从 `pipeline` 中移除故障的 `DN`，恢复写入
		- `pipeline recovery`
	- Client 挂了
		- 租约超时后，`HDFS` 会释放该文件的租约并关闭该文件
		- `lease recovery`
		- `lease recovery` 时如果有文件的 `Block` 在多个 `DN` 上处于不一致的状态，首先需要将其恢复到一致长度的状态，称为 `block recovery` 
- 24 MR 有没有遇到过没有只有 Map 没有 Reduce 的业务
### 6.31 MR 怎么实现去重，hql 去重底层原理是什么
### 6.32 Yarn 任务执行中，maptask 已完成100%，reduce task 完成60%，applicationMaster 挂掉会发生什么
### 6.33 Yarn 三种引擎和三种调度器
### 6.34 Yarn 资源配置
### 6.35 Yarn 高可用
### 6.36 Yarn 可以调用 GPU 吗
### 6.37 Yarn 可以到达什么粒度的资源分配
### 6.38 Yarn 日志很长怎么定位 error
### 6.39 在 YARN 分配资源的时候 ,因为资源不足,会杀死优先级低的任务,这个问题如何解决
### 6.40 HDFS 的存储格式是什么
### 6.41 HDFS 删除文件的过程？ NN, DN 是怎么操作的
### 6.42 调整 HDFS 的三个参数解决小文件问题,具体设置的参数是怎样的
### 6.43 HDFS 满足 CAP 原则吗
### 6.44 Hadoop 是怎样实现权限的管控和资源的隔离的
### 6.45 MR 的 stage 是什么
### 6.46 写一个脚本，杀掉 Yarn 上正在运行的程序
### 6.47 下载 HDFS 上 Yarn 的错误日志该怎么做
### 6.48 Hadoop 相对 MySQL 为什么可以做到分布式处理更大的数据量
### 6.49 Hadoop 高可用以及热备份和冷备份
### 6.50 Hadoop 怎么迁移
### 6.51 Hadoop 更改副本数
### 6.52 Hadoop 故障排错步骤
### 6.53 Yarn 日志有哪四类