# 1 提交 Job 流程

- HDFS, Spark, Flink 提交对比

![[submitJob.svg]]

![[flink_submit_job.svg]]
## 1.1 解析命令, 准备启动集群
### 1.1.1 flink 脚本
```shell
# flink
# bin/flink run-application -t yarn-application -c
# Add HADOOP_CLASSPATH to allow the usage of Hadoop file systems

# get flink config
# 读取到yarn的地址
. "$bin"/config.sh

# 执行主类CliFrontend
exec "${JAVA_RUN}" $JVM_ARGS $FLINK_ENV_JAVA_OPTS "${log_setting[@]}" -classpath "`manglePathList "$CC_CLASSPATH:$INTERNAL_HADOOP_CLASSPATHS"`" org.apache.flink.client.cli.CliFrontend "$@"
```
### 1.1.2 CliFrontend.main()
```java
public static void main(final String[] args) {
        // int INITIAL_RET_CODE = 31;
        int retCode = INITIAL_RET_CODE;
        try {
            retCode = mainInternal(args);
        } finally {
            System.exit(retCode);
        }
}

static int mainInternal(final String[] args) {

        // 1. find the configuration directory
    	// String ENV_FLINK_CONF_DIR = "FLINK_CONF_DIR"
    	// 从FLINK_CONF_DIR中获取配置文件的目录
        final String configurationDirectory = getConfigurationDirectoryFromEnv();

        // 2. load the global configuration
    	// 从目录下的FLINK_CONF_FILENAME( = "flink-conf.yaml")中加载配置到Configuration对象中
        final Configuration configuration =
                GlobalConfiguration.loadConfiguration(configurationDirectory);

        // 3. load the custom command lines
    	// 封装命令
        final List<CustomCommandLine> customCommandLines =
                loadCustomCommandLines(configuration, configurationDirectory);

        int retCode = INITIAL_RET_CODE;
        try {
            // 新建一个CliFrontend对象，将自己的命令存储在customCommandLineOptions中
            final CliFrontend cli = new CliFrontend(configuration, customCommandLines);
            CommandLine commandLine =
                    cli.getCommandLine(
                            new Options(),
                            Arrays.copyOfRange(args, min(args.length, 1), args.length),true);
            Configuration securityConfig = new Configuration(cli.configuration);
            DynamicPropertiesUtil.encodeDynamicProperties(commandLine, securityConfig);
            SecurityUtils.install(new SecurityConfiguration(securityConfig));
            // 执行命令
            retCode = SecurityUtils.getInstalledContext().runSecured(() -> cli.parseAndRun(args));
        } catch (Throwable t) {
            final Throwable strippedThrowable =
                    ExceptionUtils.stripException(t, UndeclaredThrowableException.class);
            LOG.error("Fatal error while running command line interface.", strippedThrowable);
            strippedThrowable.printStackTrace();
        }
        return retCode;
}
```
#### 1.1.2.1loadCustomCommandLines()
```java
public static List<CustomCommandLine> loadCustomCommandLines(
    Configuration configuration, String configurationDirectory) {
    // 通过List存放命令行，最后再根据情况选择创建哪一种命令行CLI
    List<CustomCommandLine> customCommandLines = new ArrayList<>();
    // 首先添加GenericCLI
    customCommandLines.add(new GenericCLI(configuration, configurationDirectory));

    //	Command line interface of the YARN session, with a special initialization here
    //	to prefix all options with y/yarn.
    final String flinkYarnSessionCLI = "org.apache.flink.yarn.cli.FlinkYarnSessionCli";
    try {
        customCommandLines.add(
            loadCustomCommandLine(
                // 再添加flinkYarnSessionCLI
                flinkYarnSessionCLI,
                configuration,
                configurationDirectory,
                "y",
                "yarn"));
    } catch (NoClassDefFoundError | Exception e) {
        final String errorYarnSessionCLI = "org.apache.flink.yarn.cli.FallbackYarnSessionCli";
        try {...} catch (Exception exception) {...}
    }

    // 最后添加DefaultCLI，用于standalone模式
    customCommandLines.add(new DefaultCLI());
	
    return customCommandLines;
}
```
#### 1.1.2.2 parseAndRun()
```java
public int parseAndRun(String[] args) {

        // check for action
        if (args.length < 1) {
            CliFrontendParser.printHelp(customCommandLines);
            System.out.println("Please specify an action.");
            return 1;
        }

        // get action
        String action = args[0]; // run-application

        // remove action from parameters
        final String[] params = Arrays.copyOfRange(args, 1, args.length);

        try {
            // do action
            switch (action) {
                case ACTION_RUN_APPLICATION:
                    runApplication(params);
                    return 0;
                ...
                case "-h":
                case "--help":
                    CliFrontendParser.printHelp(customCommandLines);
                    return 0;
                case "-v":
                case "--version":
                    String version = EnvironmentInformation.getVersion();
                    String commitID = EnvironmentInformation.getRevisionInformation().commitId;
                    System.out.print("Version: " + version);
                    System.out.println(
                            commitID.equals(EnvironmentInformation.UNKNOWN)
                                    ? ""
                                    : ", Commit ID: " + commitID);
                    return 0;
                default:
                    System.out.printf("\"%s\" is not a valid action.\n", action);
                    System.out.println();
                    System.out.println(
                            "Valid actions are \"run\", \"run-application\", \"list\", \"info\", \"savepoint\", \"stop\", or \"cancel\".");
                    System.out.println();
                    System.out.println(
                            "Specify the version option (-v or --version) to print Flink version.");
                    System.out.println();
                    System.out.println(
                            "Specify the help option (-h or --help) to get help on the command.");
                    return 1;
            }
        } catch (CliArgsException ce) {...}
}
```
### 1.1.3 runApplication()
```java
protected void runApplication(String[] args) throws Exception {
    LOG.info("Running 'run-application' command.");
    // 加载默认配置项h, help, v, PARALLELISM_OPTION, JAR_OPTION, SAVE_POINT相关的配置等
    final Options commandOptions = CliFrontendParser.getRunCommandOptions();
    /* 将上面加载的默认配置项和自己的命令merge后,new一个DefaultParser对象进行解析
     * 解析自己的命令:
     *  	是否以'-'开头: -> -c
     		是否以'--'开头: -> --class
     		其他情况: -> -p 2
     */
    final CommandLine commandLine = getCommandLine(commandOptions, args, true);
	// 打印help信息
    if (commandLine.hasOption(HELP_OPTION.getOpt())) {
        CliFrontendParser.printHelpForRunApplication(customCommandLines);
        return;
    }

    final CustomCommandLine activeCommandLine =
        // 为参数生成一个客户端
        validateAndGetActiveCommandLine(checkNotNull(commandLine));
	
    // 创建一个ApplicationDeployer对象，检查clusterClientServiceLoader是否为null
    final ApplicationDeployer deployer =
        new ApplicationClusterDeployer(clusterClientServiceLoader);

    final ProgramOptions programOptions;
    final Configuration effectiveConfiguration;

    // No need to set a jarFile path for Pyflink job.
    if (ProgramOptionsUtils.isPythonEntryPoint(commandLine)) {...} 
    else {
        // 创建ProgramOptions对象，包含jar包路径，入口类名等属性
        programOptions = new ProgramOptions(commandLine);
        programOptions.validate();
        // 获取到jar包的uri
        final URI uri = PackagedProgramUtils.resolveURI(programOptions.getJarFilePath());
        // 获取有效配置: HA的id，target，JM内存，TM内存，slots
        effectiveConfiguration =
            getEffectiveConfiguration(
            activeCommandLine,
            commandLine,
            programOptions,
            Collections.singletonList(uri.toString()));
    }

    final ApplicationConfiguration applicationConfiguration =
        new ApplicationConfiguration(
        // EntryPointClassName -c全类名
        programOptions.getProgramArgs(), programOptions.getEntryPointClassName());
    deployer.run(effectiveConfiguration, applicationConfiguration);
}
```
#### 1.1.3.1 validateAndGetActiveCommandLine()
```java
public CustomCommandLine validateAndGetActiveCommandLine(CommandLine commandLine) {
        LOG.debug("Custom commandlines: {}", customCommandLines);
    	// 三个cli
        for (CustomCommandLine cli : customCommandLines) {
            LOG.debug(
                    "Checking custom commandline {}, isActive: {}", cli, cli.isActive(commandLine));
            if (cli.isActive(commandLine)) {
                return cli;
            }
        }
        throw new IllegalStateException("No valid command-line found.");
}

// GenericCLI.isActive()
public boolean isActive(CommandLine commandLine) {
    	// 检查是否存在Option
        return configuration.getOptional(DeploymentOptions.TARGET).isPresent()
            	   // executor相关配置
                || commandLine.hasOption(executorOption.getOpt())
                || commandLine.hasOption(targetOption.getOpt()); // -t
}

public static final ConfigOption<String> TARGET =
    key("execution.target")
    .stringType()
    .noDefaultValue()
    .withDescription(
    Description.builder()
    .text(...)
    .list(
        text("remote"),
        text("local"),
        text("yarn-per-job (deprecated)"),
        text("yarn-session"),
        text("kubernetes-session"))
    .text(
        "And one of the following values when calling %s:",
        TextElement.code("bin/flink run-application"))
    .list(text("yarn-application"), text("kubernetes-application"))
    .build());

// FlinkYarnSessionCli.isActive()
public boolean isActive(CommandLine commandLine) {
    	// 先判断父类AbstractYarnCli是否active
        if (!super.isActive(commandLine)) {
            return (isYarnPropertiesFileMode(commandLine)
                    && yarnApplicationIdFromYarnProperties != null);
        }
        return true;
}

// AbstractYarnCli.isActive()
public boolean isActive(CommandLine commandLine) {
    // -m 指定JM的地址
    final String jobManagerOption = commandLine.getOptionValue(addressOption.getOpt(), null);
    // String ID = "yarn-cluster"
    final boolean yarnJobManager = ID.equals(jobManagerOption);
    // yarnSession对应的appId，即yarn有appId
    final boolean hasYarnAppId =
        commandLine.hasOption(applicationId.getOpt())
        || configuration.getOptional(YarnConfigOptions.APPLICATION_ID).isPresent();
    // 是否配置了executor
    final boolean hasYarnExecutor =
        // NAME("yarn-session")
        YarnSessionClusterExecutor.NAME.equalsIgnoreCase(
        configuration.get(DeploymentOptions.TARGET))
        // NAME("yarn-per-job")
        || YarnJobClusterExecutor.NAME.equalsIgnoreCase(
        configuration.get(DeploymentOptions.TARGET));
    return hasYarnExecutor || yarnJobManager || hasYarnAppId;
}

//最后只能走DefaultCLI
public boolean isActive(CommandLine commandLine) {
        // always active because we can try to read a JobManager address from the config
        return true;
}
```
#### 1.1.3.2 getEffectiveConfiguration()
```java
private <T> Configuration getEffectiveConfiguration(
    final CustomCommandLine activeCustomCommandLine,
    final CommandLine commandLine,
    final ProgramOptions programOptions,
    final List<T> jobJars)
    throws FlinkException {
    final Configuration effectiveConfiguration =
        getEffectiveConfiguration(activeCustomCommandLine, commandLine);

    final ExecutionConfigAccessor executionParameters =
        ExecutionConfigAccessor.fromProgramOptions(
        checkNotNull(programOptions), checkNotNull(jobJars));

    executionParameters.applyToConfiguration(effectiveConfiguration);

    LOG.debug(
        "Effective configuration after Flink conf, custom commandline, and program options: {}",
        effectiveConfiguration);
    return effectiveConfiguration;
}

private <T> Configuration getEffectiveConfiguration(
    final CustomCommandLine activeCustomCommandLine, final CommandLine commandLine)
    throws FlinkException {

    final Configuration effectiveConfiguration = new Configuration(configuration);

    final Configuration commandLineConfiguration =
        // FlinkYarnSessionCli.toConfiguration()
        checkNotNull(activeCustomCommandLine).toConfiguration(commandLine);
	
    // 添加到effectiveConfiguration中
    effectiveConfiguration.addAll(commandLineConfiguration);

    return effectiveConfiguration;
}

public Configuration toConfiguration(CommandLine commandLine) throws FlinkException {
    // we ignore the addressOption because it can only contain "yarn-cluster"
    final Configuration effectiveConfiguration = new Configuration();

    applyDescriptorOptionToConfig(commandLine, effectiveConfiguration);

    final ApplicationId applicationId = getApplicationId(commandLine);
    if (applicationId != null) {...} 
    else {
        effectiveConfiguration.setString(DeploymentOptions.TARGET, YarnJobClusterExecutor.NAME);
    }

    if (commandLine.hasOption(jmMemory.getOpt())) {
        // JM的mem
        String jmMemoryVal = commandLine.getOptionValue(jmMemory.getOpt());
        if (!MemorySize.MemoryUnit.hasUnit(jmMemoryVal)) {
            jmMemoryVal += "m";
        }
        effectiveConfiguration.set(
            JobManagerOptions.TOTAL_PROCESS_MEMORY, MemorySize.parse(jmMemoryVal));
    }

    if (commandLine.hasOption(tmMemory.getOpt())) {
        // TM的mem
        String tmMemoryVal = commandLine.getOptionValue(tmMemory.getOpt());
        if (!MemorySize.MemoryUnit.hasUnit(tmMemoryVal)) {
            tmMemoryVal += "m";
        }
        effectiveConfiguration.set(
            TaskManagerOptions.TOTAL_PROCESS_MEMORY, MemorySize.parse(tmMemoryVal));
    }

    if (commandLine.hasOption(slots.getOpt())) {
        // 插槽数 "taskmanager.numberOfTaskSlots"
        effectiveConfiguration.setInteger(
            TaskManagerOptions.NUM_TASK_SLOTS,
            Integer.parseInt(commandLine.getOptionValue(slots.getOpt())));
    }

   ...
    }
}
```
#### 1.1.3.3 deployer.run()
```java
public <ClusterID> void run(
    final Configuration configuration,
    final ApplicationConfiguration applicationConfiguration)
    throws Exception {
    checkNotNull(configuration);
    checkNotNull(applicationConfiguration);

    LOG.info("Submitting application in 'Application Mode'.");
	/*
	 * 接口ClusterClientServiceLoader的实现类DefaultClusterClientServiceLoader
	 * 	-> 获取到一个ClusterClientFactory的实现类对象YarnClusterClientFactory
	 * 		-> 创建一个YarnClusterDescriptor
	 * 			-> 创建一个ClusterSpecification，传给YarnClusterDescriptor进行部署
	 * DefaultClusterClientServiceLoader为cluster, client, factory提供服务
	 * YarnClusterDescriptor携带了在yarn上部署flink集群相关部署信息
	 * ClusterSpecification是ClusterDescriptor要启动的集群的描述信息
	 */
    final ClusterClientFactory<ClusterID> clientFactory =
        clientServiceLoader.getClusterClientFactory(configuration);
    try (final ClusterDescriptor<ClusterID> clusterDescriptor =
         clientFactory.createClusterDescriptor(configuration)) {
        final ClusterSpecification clusterSpecification =
            clientFactory.getClusterSpecification(configuration);

        clusterDescriptor.deployApplicationCluster(
            clusterSpecification, applicationConfiguration);
    }
}

private YarnClusterDescriptor getClusterDescriptor(Configuration configuration) {
    final YarnClient yarnClient = YarnClient.createYarnClient();
    final YarnConfiguration yarnConfiguration =
        Utils.getYarnAndHadoopConfiguration(configuration);
	
    // 初始化并启动一些服务
    yarnClient.init(yarnConfiguration);
    /* 调用serviceStart
     *  泛型方法，通过协议的Class创建一个连接RM的协议对象
     *	rmClient = ClientRMProxy.createRMProxy(getConfig(),
     *    ApplicationClientProtocol.class);
     */
    yarnClient.start();

    return new YarnClusterDescriptor(
        configuration,
        yarnConfiguration,
        yarnClient, // 将启动服务的YarnClientImpl传给YarnClusterDescriptor
        YarnClientYarnClusterInformationRetriever.create(yarnClient),
        false);
}

// createYarnClient会返回一个YarnClientImpl对象，它是抽象类YarnClient的子类
public static YarnClient createYarnClient() {
    YarnClient client = new YarnClientImpl();
    return client;
}
```
### 1.1.4 deployApplicationCluster()
```java
public ClusterClientProvider<ApplicationId> deployApplicationCluster(
    final ClusterSpecification clusterSpecification,
    final ApplicationConfiguration applicationConfiguration)
    throws ClusterDeploymentException {
    checkNotNull(clusterSpecification);
    checkNotNull(applicationConfiguration);

    final YarnDeploymentTarget deploymentTarget =
        YarnDeploymentTarget.fromConfig(flinkConfiguration);
    // 再次检验是否为yarn-application
    if (YarnDeploymentTarget.APPLICATION != deploymentTarget) {...}

    applicationConfiguration.applyToConfiguration(flinkConfiguration);

    // No need to do pipelineJars validation if it is a PyFlink job.
    // 不是python任务的情况下，检查jar包个数
    if (!(PackagedProgramUtils.isPython(applicationConfiguration.getApplicationClassName())
          || PackagedProgramUtils.isPython(applicationConfiguration.getProgramArguments()))) {
        final List<String> pipelineJars =
            flinkConfiguration
            .getOptional(PipelineOptions.JARS)
            .orElse(Collections.emptyList());
        Preconditions.checkArgument(pipelineJars.size() == 1, "Should only have one jar");
    }

    try {
        return deployInternal(
            clusterSpecification,
            "Flink Application Cluster", // applicationName
            YarnApplicationClusterEntryPoint.class.getName(), // yarnClusterEntrypoint
            null, // jobGraph
            false); // detached
    } catch (Exception e) {
        throw new ClusterDeploymentException("Couldn't deploy Yarn Application Cluster", e);
    }
}
```
### 1.1.5 deployInternal()
```java
private ClusterClientProvider<ApplicationId> deployInternal(
    ClusterSpecification clusterSpecification,
    String applicationName,
    String yarnClusterEntrypoint,
    @Nullable JobGraph jobGraph,
    boolean detached)
    throws Exception {
	
    // 当前用户
    final UserGroupInformation currentUser = UserGroupInformation.getCurrentUser();
    if (HadoopUtils.isKerberosSecurityEnabled(currentUser)) {...}
	
    // 检查部署需要的资源
    isReadyForDeployment(clusterSpecification);

	// 检查yarn客户端中是否存在队列
    checkYarnQueues(yarnClient);

    // 创建app
    final YarnClientApplication yarnApplication = yarnClient.createApplication();
    final GetNewApplicationResponse appResponse = yarnApplication.getNewApplicationResponse();
	
    // 获取最大资源容量
    Resource maxRes = appResponse.getMaximumResourceCapability();

	// yarn最小分配内存
    // DEFAULT_RM_SCHEDULER_MINIMUM_ALLOCATION_MB = 1024
    final int yarnMinAllocationMB =
        yarnConfiguration.getInt(
        YarnConfiguration.RM_SCHEDULER_MINIMUM_ALLOCATION_MB,
        YarnConfiguration.DEFAULT_RM_SCHEDULER_MINIMUM_ALLOCATION_MB);
    if (yarnMinAllocationMB <= 0) {...}

    ApplicationReport report =
        // 设置appMaster
        startAppMaster(
        flinkConfiguration,
        applicationName,
        yarnClusterEntrypoint,
        jobGraph,
        yarnClient,
        yarnApplication,
        validClusterSpecification);

    // print the application id for user to cancel themselves.
    if (detached) {
        final ApplicationId yarnApplicationId = report.getApplicationId();
        logDetachedClusterInformation(yarnApplicationId, LOG);
    }

    setClusterEntrypointInfoToConfig(report);

    return...
}
```
#### 1.1.5.1 isReadyForDeployment()
```java
private void isReadyForDeployment(ClusterSpecification clusterSpecification) throws Exception {
	
    // 检查jar包路径
    if (this.flinkJarPath == null) {
        throw new YarnDeploymentException("The Flink jar path is null");
    }
    // 检查配置文件对象
    if (this.flinkConfiguration == null) {
        throw new YarnDeploymentException("Flink configuration object has not been set");
    }

    // Check if we don't exceed YARN's maximum virtual cores.
    final int numYarnMaxVcores = yarnClusterInformationRetriever.getMaxVcores();

    int configuredAmVcores = flinkConfiguration.getInteger(YarnConfigOptions.APP_MASTER_VCORES);
    // 检查分配给am的Vcore个数
    if (configuredAmVcores > numYarnMaxVcores) {...}

    int configuredVcores =
        // 指定了container的核数则返回，或者默认值为TM的插槽数
        flinkConfiguration.getInteger(
        YarnConfigOptions.VCORES, clusterSpecification.getSlotsPerTaskManager());
    // don't configure more than the maximum configured number of vcores
    if (configuredVcores > numYarnMaxVcores) {...}

    // check if required Hadoop environment variables are set. If not, warn user
    // 检查yarn和hadoop的配置文件路径是否配置
    if (System.getenv("HADOOP_CONF_DIR") == null && System.getenv("YARN_CONF_DIR") == null) {...}
}
```
#### 1.1.5.2 createApplication()
```java
public YarnClientApplication createApplication()
    throws YarnException, IOException {
    // 创建提交app的上下文
    ApplicationSubmissionContext context = Records.newRecord
        (ApplicationSubmissionContext.class);
    // 通过rmClient获取一个app
    // 1.1.3.3中创建了rmClient
    GetNewApplicationResponse newApp = getNewApplication();
    ApplicationId appId = newApp.getApplicationId();
    // 设置app的id
    context.setApplicationId(appId);
    // 创建app对象
    return new YarnClientApplication(newApp, context);
}

private GetNewApplicationResponse getNewApplication()
      throws YarnException, IOException {
    GetNewApplicationRequest request =
        Records.newRecord(GetNewApplicationRequest.class);
    return rmClient.getNewApplication(request);
}

public YarnClientApplication(GetNewApplicationResponse newAppResponse,
                             ApplicationSubmissionContext appContext) {
    this.newAppResponse = newAppResponse;
    this.appSubmissionContext = appContext;
}
```
#### 1.1.5.3 startAppMaster()
```java
private ApplicationReport startAppMaster(
    Configuration configuration,
    String applicationName,
    String yarnClusterEntrypoint,
    JobGraph jobGraph,
    YarnClient yarnClient,
    YarnClientApplication yarnApplication,
    ClusterSpecification clusterSpecification)
    throws Exception {

    // ------------------ Initialize the file systems -------------------------

    org.apache.flink.core.fs.FileSystem.initialize(
        configuration, PluginUtils.createPluginManagerFromRootFolder(configuration));

    final FileSystem fs = FileSystem.get(yarnConfiguration);

	// 构建上下文
    ApplicationSubmissionContext appContext = yarnApplication.getApplicationSubmissionContext();
	
    // 提前上传好的jar包的位置
    final List<Path> providedLibDirs =
        Utils.getQualifiedRemoteProvidedLibDirs(configuration, yarnConfiguration);

    final Optional<Path> providedUsrLibDir =
        Utils.getQualifiedRemoteProvidedUsrLib(configuration, yarnConfiguration);

    Path stagingDirPath = getStagingDir(fs);
    FileSystem stagingDirFs = stagingDirPath.getFileSystem(yarnConfiguration);
    // 新建一个fileUploader用于上传jar包、配置文件
    /*
     * systemShipFiles: 日志配置文件，lib/目录下除了主要的dist之外的jar包
     * shipOnlyFiles: plugins/下的文件
     * userJarFiles: 用户代码的jar包
     */
    final YarnApplicationFileUploader fileUploader =
        YarnApplicationFileUploader.from(
        stagingDirFs,
        stagingDirPath,
        providedLibDirs,
        appContext.getApplicationId(),
        getFileReplication());

    // ------------------ Add Zookeeper namespace to local flinkConfiguraton ------
    // HA配置
    ...
  
    // Register all files in provided lib dirs as local resources with public visibility
    // and upload the remaining dependencies as local resources with APPLICATION visibility.
    // 将provided lib dirs中的文件注册为本地资源
    final List<String> systemClassPaths = fileUploader.registerProvidedLocalResources();
    final List<String> uploadedDependencies =
        fileUploader.registerMultipleLocalResources(
        systemShipFiles.stream()
        .map(e -> new Path(e.toURI()))
        .collect(Collectors.toSet()),
        Path.CUR_DIR,
        LocalResourceType.FILE);
    systemClassPaths.addAll(uploadedDependencies);

    // upload and register ship-only files
    // Plugin files only need to be shipped and should not be added to classpath.
    if (providedLibDirs == null || providedLibDirs.isEmpty()) {...}
	// 上传其他文件
    if (!shipArchives.isEmpty()) {
        fileUploader.registerMultipleLocalResources(
            shipArchives.stream().map(e -> new Path(e.toURI())).collect(Collectors.toSet()),
            Path.CUR_DIR,
            LocalResourceType.ARCHIVE);
    }
	...
    // Upload the flink configuration
    // write out configuration file
    // 上传配置文件
    File tmpConfigurationFile = null;
    try {
        tmpConfigurationFile = File.createTempFile(appId + "-flink-conf.yaml", null);
	...
    final JobManagerProcessSpec processSpec =
JobManagerProcessUtils.processSpecFromConfigWithNewOptionToInterpretLegacyHeap(
        flinkConfiguration, JobManagerOptions.TOTAL_PROCESS_MEMORY);
    // 设置appMasterContainer
    final ContainerLaunchContext amContainer =
        setupApplicationMasterContainer(yarnClusterEntrypoint, hasKrb5, processSpec);
	
	...
    // amContainer通过fileUploader获取到本地资源后，关闭fileUploader
    amContainer.setLocalResources(fileUploader.getRegisteredLocalResources());
    fileUploader.close();

    // Setup CLASSPATH and environment variables for ApplicationMaster
    // 为AM设置环境
    final Map<String, String> appMasterEnv =
	    // 创建一个Map存储环境参数
        generateApplicationMasterEnv(
        fileUploader,
        classPathBuilder.toString(),
        localResourceDescFlinkJar.toString(),
        appId.toString());
	// 将环境配置传给amContainer
    amContainer.setEnvironment(appMasterEnv);

    // Set up resource type requirements for ApplicationMaster
    // Resource是代表集群中资源的模型，包含内存和CPU
    // 通过配置为am分配资源
    Resource capability = Records.newRecord(Resource.class);
    capability.setMemory(clusterSpecification.getMasterMemoryMB());
    capability.setVirtualCores(
        flinkConfiguration.getInteger(YarnConfigOptions.APP_MASTER_VCORES));

    final String customApplicationName = customName != null ? customName : applicationName;
	// 向app上下文设置容器名，任务类型，资源大小，appContainer
    appContext.setApplicationName(customApplicationName);
    appContext.setApplicationType(applicationType != null ? applicationType : "Apache Flink");
    appContext.setAMContainerSpec(amContainer);
    appContext.setResource(capability);

    // Set priority for application
    int priorityNum = flinkConfiguration.getInteger(YarnConfigOptions.APPLICATION_PRIORITY);
    if (priorityNum >= 0) {
        Priority priority = Priority.newInstance(priorityNum);
        appContext.setPriority(priority);
    }
	// 设置提交队列
    if (yarnQueue != null) {
        appContext.setQueue(yarnQueue);
    }
	// 提交app
    yarnClient.submitApplication(appContext);
        
    ApplicationReport report;
    YarnApplicationState lastAppState = YarnApplicationState.NEW;
    loop:
    // 循环switch(appState)判断任务是否成功
    while (true) {...}

    return report;
}
```
##### 1.1.5.3.1 setupApplicationMasterContainer()
```java
ContainerLaunchContext setupApplicationMasterContainer(
    String yarnClusterEntrypoint, boolean hasKrb5, JobManagerProcessSpec processSpec) {
    // ------------------ Prepare Application Master Container  ------------------------------
	// 从yaml配置文件读取到配置后，拼接在javaOpts后
    // respect custom JVM options in the YAML file
    String javaOpts = flinkConfiguration.getString(CoreOptions.FLINK_JVM_OPTIONS);
    if (flinkConfiguration.getString(CoreOptions.FLINK_JM_JVM_OPTIONS).length() > 0) {
        javaOpts += " " + flinkConfiguration.getString(CoreOptions.FLINK_JM_JVM_OPTIONS);
    }

    // Set up the container launch context for the application master
    ContainerLaunchContext amContainer = Records.newRecord(ContainerLaunchContext.class);

    final Map<String, String> startCommandValues = new HashMap<>();
    startCommandValues.put("java", "$JAVA_HOME/bin/java");

    String jvmHeapMem =
        JobManagerProcessUtils.generateJvmParametersStr(processSpec, flinkConfiguration);
    startCommandValues.put("jvmmem", jvmHeapMem);
    startCommandValues.put("jvmopts", javaOpts);
    startCommandValues.put(
        "logging", YarnLogConfigUtil.getLoggingYarnCommand(flinkConfiguration));
    // 要启动的类是YarnApplicationClusterEntryPoint
    startCommandValues.put("class", yarnClusterEntrypoint);
    startCommandValues.put(
        "redirects",
        "1> "
        + ApplicationConstants.LOG_DIR_EXPANSION_VAR
        + "/jobmanager.out "
        + "2> "
        + ApplicationConstants.LOG_DIR_EXPANSION_VAR
        + "/jobmanager.err");
    String dynamicParameterListStr =
        JobManagerProcessUtils.generateDynamicConfigsStr(processSpec);
    startCommandValues.put("args", dynamicParameterListStr);

    final String commandTemplate =
        flinkConfiguration.getString(
        ConfigConstants.YARN_CONTAINER_START_COMMAND_TEMPLATE,
        ConfigConstants.DEFAULT_YARN_CONTAINER_START_COMMAND_TEMPLATE);
    final String amCommand =
        BootstrapTools.getStartCommand(commandTemplate, startCommandValues);

    amContainer.setCommands(Collections.singletonList(amCommand));

    LOG.debug("Application Master start command: " + amCommand);

    return amContainer;
}
```
##### 1.1.5.3.2 submitApplication()
- 向 `Yarn` 提交任务，将 `YarnApplicationClusterEntryPoint` 作为 `AppMaster` 启动
- 提交流程见 `Yarn`
- 封装启动 `Container` 的命令后，执行 `default_container_executor.sh` 脚本
- 调用` default_container_executor_session.sh`
- 最后调用 `launch_container.sh` 启动 `AppMaster` 
## 1.2 启动集群，开始创建 JobManager 组件
### 1.2.1 YarnApplicationClusterEntryPoint.main()
```java
// JobManager, AppMaster
public static void main(final String[] args) {
    // 检测工作路径是否存在，加载配置
    ...
    PackagedProgram program = null;
    try {
        // 将执行的程序封装在PackagedProgram中
        program = getPackagedProgram(configuration);
    } catch (Exception e) {...}

    try {
        // 通过配置对程序进行设置后，传给YarnApplicationClusterEntryPoint
        configureExecution(configuration, program);
    } catch (Exception e) {...}

    YarnApplicationClusterEntryPoint yarnApplicationClusterEntrypoint =
        new YarnApplicationClusterEntryPoint(configuration, program);
	// 通过startCluster方法启动cluster
    ClusterEntrypoint.runClusterEntrypoint(yarnApplicationClusterEntrypoint);
}
```
#### 1.2.1.1 new YarnApplicationClusterEntryPoint()
```java
// 继承了ApplicationClusterEntryPoint
private YarnApplicationClusterEntryPoint(
    final Configuration configuration, final PackagedProgram program) {
    // 通过YarnResourceManagerFactory创建一个YarnResourceManagerFactory实例
    super(configuration, program, YarnResourceManagerFactory.getInstance());
}

protected ApplicationClusterEntryPoint(
    final Configuration configuration,
    final PackagedProgram program,
    final ResourceManagerFactory<?> resourceManagerFactory) {
    super(configuration);
    this.program = checkNotNull(program);
    this.resourceManagerFactory = checkNotNull(resourceManagerFactory);
}
```
#### 1.2.1.2 runClusterEntrypoint()
```java
public static void runClusterEntrypoint(ClusterEntrypoint clusterEntrypoint) {

        final String clusterEntrypointName = clusterEntrypoint.getClass().getSimpleName();
        try {
            clusterEntrypoint.startCluster();
        } catch (ClusterEntrypointException e) {...}
	...
}

public void startCluster() throws ClusterEntrypointException {
        ClusterEntrypointUtils.configureUncaughtExceptionHandler(configuration);
        securityContext.runSecured(
            (Callable<Void>)
            () -> {
                runCluster(configuration, pluginManager);

                return null;
            });
    } catch (Throwable t) {...}
```
### 1.2.2 runCluster()
```java
private void runCluster(Configuration configuration, PluginManager pluginManager)
    throws Exception {
    synchronized (lock) {
        // 开启Akka的ActorService
        initializeServices(configuration, pluginManager);

        final DispatcherResourceManagerComponentFactory
            dispatcherResourceManagerComponentFactory =
            createDispatcherResourceManagerComponentFactory(configuration);
		
        // 创建JobManager的组件
        clusterComponent =
            dispatcherResourceManagerComponentFactory.create(
            configuration,
            resourceId.unwrap(),
            ioExecutor,
            commonRpcService,
            haServices,
            blobServer,
            heartbeatServices,
            delegationTokenManager,
            metricRegistry,
            executionGraphInfoStore,
            new RpcMetricQueryServiceRetriever(
                metricRegistry.getMetricQueryServiceRpcService()),
            this);
}
```
## 1.3 启动 JobManager 组件：Dispatcher，RM
```java
// DispatcherResourceManagerComponentFactory有一个实现类DefaultDispatcherResourceManagerComponentFactory
public DispatcherResourceManagerComponent create(
    Configuration configuration,
    ResourceID resourceId,
    Executor ioExecutor,
    RpcService rpcService,
    HighAvailabilityServices highAvailabilityServices,
    BlobServer blobServer,
    HeartbeatServices heartbeatServices,
    DelegationTokenManager delegationTokenManager,
    MetricRegistry metricRegistry,
    ExecutionGraphInfoStore executionGraphInfoStore,
    MetricQueryServiceRetriever metricQueryServiceRetriever,
    FatalErrorHandler fatalErrorHandler)
    throws Exception {

    LeaderRetrievalService dispatcherLeaderRetrievalService = null;
    LeaderRetrievalService resourceManagerRetrievalService = null;
    WebMonitorEndpoint<?> webMonitorEndpoint = null;
    ResourceManagerService resourceManagerService = null;
    DispatcherRunner dispatcherRunner = null;

    try {
        // HA下各组件leader的获取
        ...
		
        // 创建一个ExecutorThreadFactory
        final ScheduledExecutorService executor =
            WebMonitorEndpoint.createExecutorService(
            configuration.getInteger(RestOptions.SERVER_NUM_THREADS),
            configuration.getInteger(RestOptions.SERVER_THREAD_PRIORITY),
            "DispatcherRestEndpoint");

        // 通过SessionRestEndpointFactory创建一个DispatcherRestEndpoint
        // 开启rest endpoint
        webMonitorEndpoint =
            restEndpointFactory.createRestEndpoint(
            configuration,
            dispatcherGatewayRetriever,
            resourceManagerGatewayRetriever,
            blobServer,
            executor,
            metricFetcher,
            highAvailabilityServices.getClusterRestEndpointLeaderElectionService(),
            fatalErrorHandler);

        log.debug("Starting Dispatcher REST endpoint.");
        // 创建数个Handler
        webMonitorEndpoint.start();

        final String hostname = RpcUtils.getHostname(rpcService);
		
        // 创建ResourceManagerServiceImpl
        resourceManagerService =
            ResourceManagerServiceImpl.create(
            resourceManagerFactory,
            configuration,
            resourceId,
            rpcService,
            highAvailabilityServices,
            heartbeatServices,
            delegationTokenManager,
            fatalErrorHandler,
            new ClusterInformation(hostname, blobServer.getPort()),
            webMonitorEndpoint.getRestBaseUrl(),
            metricRegistry,
            hostname,
            ioExecutor);
		
        // 通过dispatcherRunnerFactory创建一个DispatcherRunner
        // 创建启动dispatcher
        dispatcherRunner =
            dispatcherRunnerFactory.createDispatcherRunner(
            highAvailabilityServices.getDispatcherLeaderElectionService(),
            fatalErrorHandler,
            new HaServicesJobPersistenceComponentFactory(highAvailabilityServices),
            ioExecutor,
            rpcService,
            partialDispatcherServices);

        log.debug("Starting ResourceManagerService.");
        // 启动RM
        resourceManagerService.start();

        resourceManagerRetrievalService.start(resourceManagerGatewayRetriever);
        dispatcherLeaderRetrievalService.start(dispatcherGatewayRetriever);

        return new DispatcherResourceManagerComponent(
            dispatcherRunner,
            resourceManagerService,
            dispatcherLeaderRetrievalService,
            resourceManagerRetrievalService,
            webMonitorEndpoint,
            fatalErrorHandler,
            dispatcherOperationCaches);

    }
}	
```
### 1.3.1 webMonitorEndpoint.start()
```java
public final void start() throws Exception {
    synchronized (lock) {
        Preconditions.checkState(
            state == State.CREATED, "The RestServerEndpoint cannot be restarted.");

        log.info("Starting rest endpoint.");

        final Router router = new Router();
        final CompletableFuture<String> restAddressFuture = new CompletableFuture<>();
		// 创建handler集合
        handlers = initializeHandlers(restAddressFuture);

        /* sort the handlers such that they are ordered the following:
             * /jobs
             * /jobs/overview
             * /jobs/:jobid
             * /jobs/:jobid/config
             * /:*
             */
        Collections.sort(handlers, RestHandlerUrlComparator.INSTANCE);

        checkAllEndpointsAndHandlersAreUnique(handlers);
        handlers.forEach(handler -> registerHandler(router, handler, log));

        bootstrap = new ServerBootstrap();
        bootstrap
            .group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(initializer);

        log.debug("Binding rest endpoint to {}:{}.", restBindAddress, chosenPort);
		// 选举
        startInternal();
    }
}

public void startInternal() throws Exception {
    leaderElectionService.start(this);
    startExecutionGraphCacheCleanupTask();

    if (hasWebUI) {
        log.info("Web frontend listening at {}.", getRestBaseUrl());
    }
}

public void grantLeadership(final UUID leaderSessionID) {
    log.info(
        "{} was granted leadership with leaderSessionID={}",
        getRestBaseUrl(),
        leaderSessionID);
    leaderElectionService.confirmLeadership(leaderSessionID, getRestBaseUrl());
}
```
### 1.3.2 createDispatcherRunner()
#### 1.3.2.1 创建 Runner
```java
public DispatcherRunner createDispatcherRunner(
            LeaderElectionService leaderElectionService,
            FatalErrorHandler fatalErrorHandler,
            JobPersistenceComponentFactory jobPersistenceComponentFactory,
            Executor ioExecutor,
            RpcService rpcService,
            PartialDispatcherServices partialDispatcherServices)
            throws Exception {
		
    	// 创建一个DispatcherLeaderProcessFactory，用于选举leader
        final DispatcherLeaderProcessFactory dispatcherLeaderProcessFactory =
                dispatcherLeaderProcessFactoryFactory.createFactory(
                        jobPersistenceComponentFactory,
                        ioExecutor,
                        rpcService,
                        partialDispatcherServices,
                        fatalErrorHandler);
		// 返回一个DefaultDispatcherRunner
        return DefaultDispatcherRunner.create(
                leaderElectionService, fatalErrorHandler, dispatcherLeaderProcessFactory);
}

public static DispatcherRunner create(
            LeaderElectionService leaderElectionService,
            FatalErrorHandler fatalErrorHandler,
            DispatcherLeaderProcessFactory dispatcherLeaderProcessFactory)
            throws Exception {
        final DefaultDispatcherRunner dispatcherRunner =
                new DefaultDispatcherRunner(
                        leaderElectionService, fatalErrorHandler, dispatcherLeaderProcessFactory);
        dispatcherRunner.start();
        return dispatcherRunner;
}
```
#### 1.3.2.2 启动 Runner 创建 Dispatcher
```java
// DefaultDispatcherRunner.java
void start() throws Exception {
    	// 启动leader选举
        leaderElectionService.start(this);
}

// StandaloneLeaderElectionService.java
public void start(LeaderContender newContender) throws Exception {
    if (contender != null) {
        // Service was already started
        throw new IllegalArgumentException(
            "Leader election service cannot be started multiple times.");
    }

    contender = Preconditions.checkNotNull(newContender);

    // directly grant leadership to the given contender
    contender.grantLeadership(HighAvailabilityServices.DEFAULT_LEADER_ID);
}

// DefaultDispatcherRunner.java
public void grantLeadership(UUID leaderSessionID) {
    runActionIfRunning(
        () -> {
            LOG.info(
                /*
                DefaultDispatcherRunner was granted leadership 
                with leader id 00000000-0000-0000-0000-000000000000. 
                Creating new DispatcherLeaderProcess.
                */
                "{} was granted leadership with leader id {}. Creating new {}.",
                getClass().getSimpleName(),
                leaderSessionID,
                DispatcherLeaderProcess.class.getSimpleName());
            startNewDispatcherLeaderProcess(leaderSessionID);
        });
}

private void startNewDispatcherLeaderProcess(UUID leaderSessionID) {
    stopDispatcherLeaderProcess();
	// 创建SessionDispatcherLeaderProcess
    dispatcherLeaderProcess = createNewDispatcherLeaderProcess(leaderSessionID);

    final DispatcherLeaderProcess newDispatcherLeaderProcess = dispatcherLeaderProcess;
    FutureUtils.assertNoException(
        previousDispatcherLeaderProcessTerminationFuture.thenRun(
            newDispatcherLeaderProcess::start));
}

// 通过父类AbstractDispatcherLeaderProcess的start方法，调用到子类的onStart方法
protected void onStart() {
    // 启动JobGraphStore
    startServices();

    onGoingRecoveryOperation =
        createDispatcherBasedOnRecoveredJobGraphsAndRecoveredDirtyJobResults();
}

private CompletableFuture<Void>
    createDispatcherBasedOnRecoveredJobGraphsAndRecoveredDirtyJobResults() {
    final CompletableFuture<Collection<JobResult>> dirtyJobsFuture =
        CompletableFuture.supplyAsync(this::getDirtyJobResultsIfRunning, ioExecutor);

    return dirtyJobsFuture
        .thenApplyAsync(
        dirtyJobs ->
        this.recoverJobsIfRunning(
            dirtyJobs.stream()
            .map(JobResult::getJobId)
            .collect(Collectors.toSet())),
        ioExecutor)
        // createDispatcherIfRunning创建dispatcher
        .thenAcceptBoth(dirtyJobsFuture, this::createDispatcherIfRunning)
        .handle(this::onErrorIfRunning);
}

// 接收到之前未完成的job再创建dispatcher
// Recover all persisted job graphs that are not finished, yet.
// Successfully recovered 0 persisted job graphs.

private void createDispatcherIfRunning(
    Collection<JobGraph> jobGraphs, Collection<JobResult> recoveredDirtyJobResults) {
    runIfStateIs(State.RUNNING, () -> createDispatcher(jobGraphs, recoveredDirtyJobResults));
}

private void createDispatcher(
    Collection<JobGraph> jobGraphs, Collection<JobResult> recoveredDirtyJobResults) {
	
    final DispatcherGatewayService dispatcherService =
        // 通过ApplicationDispatcherGatewayServiceFactory创建dispatcher
        dispatcherGatewayServiceFactory.create(
        DispatcherId.fromUuid(getLeaderSessionId()),
        jobGraphs,
        recoveredDirtyJobResults,
        jobGraphStore,
        jobResultStore);

    completeDispatcherSetup(dispatcherService);
}

public AbstractDispatcherLeaderProcess.DispatcherGatewayService create(
    DispatcherId fencingToken,
    Collection<JobGraph> recoveredJobs,
    Collection<JobResult> recoveredDirtyJobResults,
    JobGraphWriter jobGraphWriter,
    JobResultStore jobResultStore) {

    final List<JobID> recoveredJobIds = getRecoveredJobIds(recoveredJobs);

    final Dispatcher dispatcher;
    try {
        dispatcher =
            // 返回一个StandaloneDispatcher
            dispatcherFactory.createDispatcher(
            rpcService,
            fencingToken,
            recoveredJobs,
            recoveredDirtyJobResults,
            (dispatcherGateway, scheduledExecutor, errorHandler) ->
            // 获取用户指定的main方法
            new ApplicationDispatcherBootstrap(
                application,
                recoveredJobIds,
                configuration,
                dispatcherGateway,
                scheduledExecutor,
                errorHandler),
            PartialDispatcherServicesWithJobPersistenceComponents.from(
                partialDispatcherServices, jobGraphWriter, jobResultStore));
    } catch (Exception e) {
        throw new FlinkRuntimeException("Could not create the Dispatcher rpc endpoint.", e);
    }
	
    // Dispatcher继承了FencedRpcEndpoint
    // 启动Akka的RPC终端
    dispatcher.start();

    return DefaultDispatcherGatewayService.from(dispatcher);
}

public ApplicationDispatcherBootstrap(
    final PackagedProgram application,
    final Collection<JobID> recoveredJobIds,
    final Configuration configuration,
    final DispatcherGateway dispatcherGateway,
    final ScheduledExecutor scheduledExecutor,
    final FatalErrorHandler errorHandler) {
    this.configuration = checkNotNull(configuration);
    this.recoveredJobIds = checkNotNull(recoveredJobIds);
    this.application = checkNotNull(application);
    this.errorHandler = checkNotNull(errorHandler);

    this.applicationCompletionFuture =
        fixJobIdAndRunApplicationAsync(dispatcherGateway, scheduledExecutor);

    this.bootstrapCompletionFuture = finishBootstrapTasks(dispatcherGateway);
}

private CompletableFuture<Void> fixJobIdAndRunApplicationAsync(
    final DispatcherGateway dispatcherGateway, final ScheduledExecutor scheduledExecutor) {
    ...
    return runApplicationAsync(
        dispatcherGateway, scheduledExecutor, true, submitFailedJobOnApplicationError);
}

private CompletableFuture<Void> runApplicationAsync(
    final DispatcherGateway dispatcherGateway,
    final ScheduledExecutor scheduledExecutor,
    final boolean enforceSingleJobExecution,
    final boolean submitFailedJobOnApplicationError) {
    final CompletableFuture<List<JobID>> applicationExecutionFuture = new CompletableFuture<>();
    final Set<JobID> tolerateMissingResult = Collections.synchronizedSet(new HashSet<>());

    // we need to hand in a future as return value because we need to get those JobIs out
    // from the scheduled task that executes the user program
    applicationExecutionTask =
        scheduledExecutor.schedule(
        () ->
        runApplicationEntryPoint(
            applicationExecutionFuture,
            tolerateMissingResult,
            dispatcherGateway,
            scheduledExecutor,
            enforceSingleJobExecution,
            submitFailedJobOnApplicationError),
        0L,
        TimeUnit.MILLISECONDS);

    return applicationExecutionFuture.thenCompose(
        jobIds ->
        getApplicationResult(
            dispatcherGateway,
            jobIds,
            tolerateMissingResult,
            scheduledExecutor));
}

private void runApplicationEntryPoint(
    final CompletableFuture<List<JobID>> jobIdsFuture,
    final Set<JobID> tolerateMissingResult,
    final DispatcherGateway dispatcherGateway,
    final ScheduledExecutor scheduledExecutor,
    final boolean enforceSingleJobExecution,
    final boolean submitFailedJobOnApplicationError) {
    if (submitFailedJobOnApplicationError && !enforceSingleJobExecution) {
        jobIdsFuture.completeExceptionally(
            new ApplicationExecutionException(
                String.format(
                    "Submission of failed job in case of an application error ('%s') is not supported in non-HA setups.",
                    DeploymentOptions.SUBMIT_FAILED_JOB_ON_APPLICATION_ERROR
                    .key())));
        return;
    }
    final List<JobID> applicationJobIds = new ArrayList<>(recoveredJobIds);
    try {
        final PipelineExecutorServiceLoader executorServiceLoader =
            new EmbeddedExecutorServiceLoader(
            applicationJobIds, dispatcherGateway, scheduledExecutor);
		
        // 执行main方法
        ClientUtils.executeProgram(
            executorServiceLoader,
            configuration,
            application,
            enforceSingleJobExecution,
            true /* suppress sysout */);

        if (applicationJobIds.isEmpty()) {
            jobIdsFuture.completeExceptionally(
                new ApplicationExecutionException(
                    "The application contains no execute() calls."));
        } else {
            jobIdsFuture.complete(applicationJobIds);
        }
    } catch (Throwable t) {...}
}

public static void executeProgram(
    PipelineExecutorServiceLoader executorServiceLoader,
    Configuration configuration,
    PackagedProgram program,
    boolean enforceSingleJobExecution,
    boolean suppressSysout)
    throws ProgramInvocationException {
    checkNotNull(executorServiceLoader);
    final ClassLoader userCodeClassLoader = program.getUserCodeClassLoader();
    final ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();
    try {
        ...
        try {
            // callMainMethod调用main方法
            program.invokeInteractiveModeForExecution();
        } finally {
            ContextEnvironment.unsetAsContext();
            StreamContextEnvironment.unsetAsContext();
        }
    } finally {
        Thread.currentThread().setContextClassLoader(contextClassLoader);
    }
}
```
#### 1.3.2.3 启动 Dispatcher
```java
/*
 * RpcEndpoint有四个重要的子类:
 * 	TaskExecutor： Task Manager中的具体业务逻辑实现
 * 	Dispatcher -> StandaloneDispatcher
 * 	JobMaster：在提交成功⼀个作业后就会启动⼀个 Job Master 的主节点负责作业的执⾏
 * 	ResourceManager -> ActiveResourceManager
 * 	启动⼀个组件的实例对象时，都会通过RpcEndpoint执⾏onStart()⽅法
 */

public void onStart() throws Exception {
    try {
        // registerDispatcherMetrics
        startDispatcherServices();
    } catch (Throwable t) {
        final DispatcherException exception =
            new DispatcherException(
            String.format("Could not start the Dispatcher %s", getAddress()), t);
        onFatalError(exception);
        throw exception;
    }

    startCleanupRetries();
    // 启动从JobGraph中恢复出的job
    startRecoveredJobs();

    this.dispatcherBootstrap =
        this.dispatcherBootstrapFactory.create(
        getSelfGateway(DispatcherGateway.class),
        this.getRpcService().getScheduledExecutor(),
        this::onFatalError);
}
```
### 1.3.3 ResourceManagerServiceImpl.start()
#### 1.3.3.1 创建 ResourceManager
```java
// 与dispatcher类似，需要选举一个leader
public void start() throws Exception {
    synchronized (lock) {
        if (running) {
            LOG.debug("Resource manager service has already started.");
            return;
        }
        running = true;
    }

    LOG.info("Starting resource manager service.");

    leaderElectionService.start(this);
}

// StandaloneLeaderElectionService.start() -> ResourceManagerServiceImpl.grantLeadership()
public void grantLeadership(UUID newLeaderSessionID) {
    handleLeaderEventExecutor.execute(
        () -> {
            synchronized (lock) {
                if (!running) {...}

                LOG.info(
                    "Resource manager service is granted leadership with session id {}.",
                    newLeaderSessionID);

                try {
                startNewLeaderResourceManager(newLeaderSessionID);
                } catch (Throwable t) {...}
            }
        });
}

private void startNewLeaderResourceManager(UUID newLeaderSessionID) throws Exception {
    stopLeaderResourceManager();

    this.leaderSessionID = newLeaderSessionID;
    // 创建RM
    this.leaderResourceManager =
    resourceManagerFactory.createResourceManager(rmProcessContext, newLeaderSessionID);

    final ResourceManager<?> newLeaderResourceManager = this.leaderResourceManager;
}

public ResourceManager<T> createResourceManager(
    ResourceManagerProcessContext context, UUID leaderSessionId) throws Exception {
	
    // 创建SlotManager
    final ResourceManagerRuntimeServices resourceManagerRuntimeServices =
        createResourceManagerRuntimeServices(
        context.getRmRuntimeServicesConfig(),
        context.getRpcService(),
        context.getHighAvailabilityServices(),
        SlotManagerMetricGroup.create(
            context.getMetricRegistry(), context.getHostname()));
	// ActiveResourceManagerFactory创建一个ActiveResourceManager
    return createResourceManager(
        context.getRmConfig(),
        context.getResourceId(),
        context.getRpcService(),
        leaderSessionId,
        context.getHeartbeatServices(),
        context.getDelegationTokenManager(),
        context.getFatalErrorHandler(),
        context.getClusterInformation(),
        context.getWebInterfaceUrl(),
        ResourceManagerMetricGroup.create(
            context.getMetricRegistry(), context.getHostname()),
        resourceManagerRuntimeServices,
        context.getIoExecutor());
}

public ResourceManager<WorkerType> createResourceManager(
    Configuration configuration,
    ResourceID resourceId,
    RpcService rpcService,
    UUID leaderSessionId,
    HeartbeatServices heartbeatServices,
    DelegationTokenManager delegationTokenManager,
    FatalErrorHandler fatalErrorHandler,
    ClusterInformation clusterInformation,
    @Nullable String webInterfaceUrl,
    ResourceManagerMetricGroup resourceManagerMetricGroup,
    ResourceManagerRuntimeServices resourceManagerRuntimeServices,
    Executor ioExecutor)
    throws Exception {

    final ThresholdMeter failureRater = createStartWorkerFailureRater(configuration);
    final Duration retryInterval =
        // 默认3s
        configuration.get(ResourceManagerOptions.START_WORKER_RETRY_INTERVAL);
    final Duration workerRegistrationTimeout =
        // 5min
        configuration.get(ResourceManagerOptions.TASK_MANAGER_REGISTRATION_TIMEOUT);
    final Duration previousWorkerRecoverTimeout =
        configuration.get(
        ResourceManagerOptions.RESOURCE_MANAGER_PREVIOUS_WORKER_RECOVERY_TIMEOUT);
	// 返回一个ActiveResourceManager
    // 构造器一直向上调用最后开启RpcServer
    return new ActiveResourceManager<>(
        createResourceManagerDriver(...);
}

// YarnResourceManagerFactory.java 
protected ResourceManagerDriver<YarnWorkerNode> createResourceManagerDriver(
            Configuration configuration, String webInterfaceUrl, String rpcAddress) {
	final YarnResourceManagerDriverConfiguration yarnResourceManagerDriverConfiguration =
    new YarnResourceManagerDriverConfiguration(
        System.getenv(), rpcAddress, webInterfaceUrl);

  return new YarnResourceManagerDriver(
         configuration,
         yarnResourceManagerDriverConfiguration,
         DefaultYarnResourceManagerClientFactory.getInstance(),
         DefaultYarnNodeManagerClientFactory.getInstance());
}
```
#### 1.3.3.2 创建 SlotManager
```java
private ResourceManagerRuntimeServices createResourceManagerRuntimeServices(
    ResourceManagerRuntimeServicesConfiguration rmRuntimeServicesConfig,
    RpcService rpcService,
    HighAvailabilityServices highAvailabilityServices,
    SlotManagerMetricGroup slotManagerMetricGroup) {

    return ResourceManagerRuntimeServices.fromConfiguration(
        rmRuntimeServicesConfig,
        highAvailabilityServices,
        rpcService.getScheduledExecutor(),
        slotManagerMetricGroup);
}

public static ResourceManagerRuntimeServices fromConfiguration(
    ResourceManagerRuntimeServicesConfiguration configuration,
    HighAvailabilityServices highAvailabilityServices,
    ScheduledExecutor scheduledExecutor,
    SlotManagerMetricGroup slotManagerMetricGroup) {
	
    // 配置了enableFineGrainedResourceManagement返回FineGrainedSlotManager
    // 否则返回DeclarativeSlotManager
    // DeclarativeSlotManager实例化时会创建TaskExecutorManager
    final SlotManager slotManager =
        createSlotManager(configuration, scheduledExecutor, slotManagerMetricGroup);

    final JobLeaderIdService jobLeaderIdService =
        new DefaultJobLeaderIdService(
        highAvailabilityServices, scheduledExecutor, configuration.getJobTimeout());

    return new ResourceManagerRuntimeServices(slotManager, jobLeaderIdService);
}
```
#### 1.3.3.3 启动 ResourceManager
```java
public final void onStart() throws Exception {
    try {
        log.info("Starting the resource manager.");
        startResourceManagerServices();
        startedFuture.complete(null);
    } catch (Throwable t) {
        final ResourceManagerException exception =
            new ResourceManagerException(
            String.format("Could not start the ResourceManager %s", getAddress()),
            t);
        onFatalError(exception);
        throw exception;
    }
}

private void startResourceManagerServices() throws Exception {
    try {
        jobLeaderIdService.start(new JobLeaderIdActionsImpl());

        registerMetrics();

        startHeartbeatServices();

        slotManager.start(
            getFencingToken(),
            getMainThreadExecutor(),
            resourceAllocator,
            new ResourceEventListenerImpl(),
            blocklistHandler::isBlockedTaskManager);
		// 启动delegationTokenManager
        delegationTokenManager.start(this);

        initialize();
    } catch (Exception e) {
        handleStartResourceManagerServicesException(e);
    }
}

protected void initialize() throws ResourceManagerException {
    try {
        resourceManagerDriver.initialize(
            this,
            new GatewayMainThreadExecutor(),
            ioExecutor,
            blocklistHandler::getAllBlockedNodeIds);
    } catch (Exception e) {
        throw new ResourceManagerException("Cannot initialize resource provider.", e);
    }
}

// AbstractResourceManagerDriver.java
public final void initialize(
    ResourceEventHandler<WorkerType> resourceEventHandler,
    ScheduledExecutor mainThreadExecutor,
    Executor ioExecutor,
    BlockedNodeRetriever blockedNodeRetriever)
    throws Exception {
    this.resourceEventHandler = Preconditions.checkNotNull(resourceEventHandler);
    this.mainThreadExecutor = Preconditions.checkNotNull(mainThreadExecutor);
    this.ioExecutor = Preconditions.checkNotNull(ioExecutor);
    this.blockedNodeRetriever = Preconditions.checkNotNull(blockedNodeRetriever);

    initializeInternal();
}

// YarnResourceManagerDriver.java
protected void initializeInternal() throws Exception {
    isRunning = true;
    final YarnContainerEventHandler yarnContainerEventHandler = new YarnContainerEventHandler();
    try {
        resourceManagerClient =
            // AMRMClientAsyncImpl
            yarnResourceManagerClientFactory.createResourceManagerClient(
            yarnHeartbeatIntervalMillis, yarnContainerEventHandler);
        resourceManagerClient.init(yarnConfig);
        resourceManagerClient.start();
		
        // AM向RM注册
        final RegisterApplicationMasterResponse registerApplicationMasterResponse =
            registerApplicationMaster();
        getContainersFromPreviousAttempts(registerApplicationMasterResponse);
        taskExecutorProcessSpecContainerResourcePriorityAdapter =
            new TaskExecutorProcessSpecContainerResourcePriorityAdapter(
            registerApplicationMasterResponse.getMaximumResourceCapability(),
            ExternalResourceUtils.getExternalResourceConfigurationKeys(
                flinkConfig,
                YarnConfigOptions.EXTERNAL_RESOURCE_YARN_CONFIG_KEY_SUFFIX));
    } catch (Exception e) {
        throw new ResourceManagerException("Could not start resource manager client.", e);
    }
	
    // NMClientAsyncImpl
    nodeManagerClient =        yarnNodeManagerClientFactory.createNodeManagerClient(yarnContainerEventHandler);
    nodeManagerClient.init(yarnConfig);
    nodeManagerClient.start();
}
```
## 1.4 提交任务, 创建 JobMaster
### 1.4.1 execute()
```java
public JobExecutionResult execute() throws Exception {
        return execute((String) null);
}

public JobExecutionResult execute(String jobName) throws Exception {
    final List<Transformation<?>> originalTransformations = new ArrayList<>(transformations);
    // 创建StreamGraph
    StreamGraph streamGraph = getStreamGraph();
    if (jobName != null) {
        streamGraph.setJobName(jobName);
    }
    try {
        return execute(streamGraph);
    }
}

public JobExecutionResult execute(StreamGraph streamGraph) throws Exception {
    final JobClient jobClient = executeAsync(streamGraph);
    }

public JobClient executeAsync(StreamGraph streamGraph) throws Exception {
    checkNotNull(streamGraph, "StreamGraph cannot be null.");
    // 创建EmbeddedExecutor
    final PipelineExecutor executor = getPipelineExecutor();

    // 获取到Future对象
    CompletableFuture<JobClient> jobClientFuture =
        executor.execute(streamGraph, configuration, userClassloader);
}

// EmbeddedExecutor
public CompletableFuture<JobClient> execute(
    final Pipeline pipeline,
    final Configuration configuration,
    ClassLoader userCodeClassloader)
    throws MalformedURLException {
    checkNotNull(pipeline);
    checkNotNull(configuration);

    final Optional<JobID> optJobId =
        configuration
        .getOptional(PipelineOptionsInternal.PIPELINE_FIXED_JOB_ID)
        .map(JobID::fromHexString);
	// 已经提交过
    if (optJobId.isPresent() && submittedJobIds.contains(optJobId.get())) {
        return getJobClientFuture(optJobId.get(), userCodeClassloader);
    }

    return submitAndGetJobClientFuture(pipeline, configuration, userCodeClassloader);
}
```
### 1.4.2 submitAndGetJobClientFuture()
```java
private CompletableFuture<JobClient> submitAndGetJobClientFuture(
    final Pipeline pipeline,
    final Configuration configuration,
    final ClassLoader userCodeClassloader)
    throws MalformedURLException {
    final Time timeout =
        Time.milliseconds(configuration.get(ClientOptions.CLIENT_TIMEOUT).toMillis());

    // StreamGraph -> JobGraph
    final JobGraph jobGraph =
        PipelineExecutorUtils.getJobGraph(pipeline, configuration, userCodeClassloader);
    final JobID actualJobId = jobGraph.getJobID();

    this.submittedJobIds.add(actualJobId);
    LOG.info("Job {} is submitted.", actualJobId);

    if (LOG.isDebugEnabled()) {
        LOG.debug("Effective Configuration: {}", configuration);
    }
    // 提交job
    final CompletableFuture<JobID> jobSubmissionFuture =
        submitJob(configuration, dispatcherGateway, jobGraph, timeout);
	return...
}

private static CompletableFuture<JobID> submitJob(
    final Configuration configuration,
    final DispatcherGateway dispatcherGateway,
    final JobGraph jobGraph,
    final Time rpcTimeout) {
    checkNotNull(jobGraph);

    LOG.info("Submitting Job with JobId={}.", jobGraph.getJobID());

    return dispatcherGateway
        .getBlobServerPort(rpcTimeout)
        .thenApply(
        blobServerPort ->
        new InetSocketAddress(
            dispatcherGateway.getHostname(), blobServerPort))
        .thenCompose(
        blobServerAddress -> {
            try {
                // 通过BlobServer把JobGraph持久化为JobGraphFile
                ClientUtils.extractAndUploadJobGraphFiles(
                    jobGraph,
                    () -> new BlobClient(blobServerAddress, configuration));
            } catch (FlinkException e) {
                throw new CompletionException(e);
            }
			// 向网关提交job
            return dispatcherGateway.submitJob(jobGraph, rpcTimeout);
        })
        .thenApply(ack -> jobGraph.getJobID());
}

// submit的请求，会被RestServer端的JobSubmitHandler接收，将JobGraphFile恢复成JobGraph
public CompletableFuture<Acknowledge> submitJob(JobGraph jobGraph, Time timeout) {
    log.info(
        "Received JobGraph submission '{}' ({}).", jobGraph.getName(), jobGraph.getJobID());

    try {
        if (isDuplicateJob(jobGraph.getJobID())) {...}
        else if (isPartialResourceConfigured(jobGraph)) {...} 
        else {
            return internalSubmitJob(jobGraph);
        }
    } catch (FlinkException e) {
        return FutureUtils.completedExceptionally(e);
    }
}

private CompletableFuture<Acknowledge> internalSubmitJob(JobGraph jobGraph) {
    applyParallelismOverrides(jobGraph);
    log.info("Submitting job '{}' ({}).", jobGraph.getName(), jobGraph.getJobID());
    // 创建JobMaster
    return waitForTerminatingJob(jobGraph.getJobID(), jobGraph, this::persistAndRunJob)
        .handle((ignored, throwable) -> handleTermination(jobGraph.getJobID(), throwable))
        .thenCompose(Function.identity());
}
```
### 1.4.3 persistAndRunJob()
```java
private void persistAndRunJob(JobGraph jobGraph) throws Exception {
    jobGraphWriter.putJobGraph(jobGraph);
    initJobClientExpiredTime(jobGraph);
    runJob(createJobMasterRunner(jobGraph), ExecutionType.SUBMISSION);
}
```
#### 1.4.3.1 createJobMasterRunner()
```java
private JobManagerRunner createJobMasterRunner(JobGraph jobGraph) throws Exception {
    Preconditions.checkState(!jobManagerRunnerRegistry.isRegistered(jobGraph.getJobID()));
    return jobManagerRunnerFactory.createJobManagerRunner(
        jobGraph,
        configuration,
        getRpcService(),
        highAvailabilityServices,
        heartbeatServices,
        jobManagerSharedServices,
        new DefaultJobManagerJobMetricGroupFactory(jobManagerMetricGroup),
        fatalErrorHandler,
        System.currentTimeMillis());
}

// JobMasterServiceLeadershipRunnerFactory.java
public JobManagerRunner createJobManagerRunner(
    JobGraph jobGraph,
    Configuration configuration,
    RpcService rpcService,
    HighAvailabilityServices highAvailabilityServices,
    HeartbeatServices heartbeatServices,
    JobManagerSharedServices jobManagerServices,
    JobManagerJobMetricGroupFactory jobManagerJobMetricGroupFactory,
    FatalErrorHandler fatalErrorHandler,
    long initializationTimestamp)
    throws Exception {
	...
    // 返回一个JobMasterServiceLeadershipRunner
    return new JobMasterServiceLeadershipRunner(
        jobMasterServiceProcessFactory,
        jobManagerLeaderElectionService,
        jobResultStore,
        classLoaderLease,
        fatalErrorHandler);
}

public JobMasterServiceLeadershipRunner(
    JobMasterServiceProcessFactory jobMasterServiceProcessFactory,
    LeaderElectionService leaderElectionService,
    JobResultStore jobResultStore,
    LibraryCacheManager.ClassLoaderLease classLoaderLease,
    FatalErrorHandler fatalErrorHandler) {
    this.jobMasterServiceProcessFactory = jobMasterServiceProcessFactory;
    this.leaderElectionService = leaderElectionService;
    this.jobResultStore = jobResultStore;
    this.classLoaderLease = classLoaderLease;
    this.fatalErrorHandler = fatalErrorHandler;
}
```
#### 1.4.3.2 runJob()
```java
private void runJob(JobManagerRunner jobManagerRunner, ExecutionType executionType)
    throws Exception {
    // 启动上一步创建的runner
    jobManagerRunner.start();
    jobManagerRunnerRegistry.register(jobManagerRunner);

    final JobID jobId = jobManagerRunner.getJobID();

    final CompletableFuture<CleanupJobState> cleanupJobStateFuture =
        jobManagerRunner
        .getResultFuture()
        .handleAsync(
        (jobManagerRunnerResult, throwable) -> {
            Preconditions.checkState(
                jobManagerRunnerRegistry.isRegistered(jobId)
                && jobManagerRunnerRegistry.get(jobId)
                == jobManagerRunner,
                "The job entry in runningJobs must be bound to the lifetime of the JobManagerRunner.");

            if (jobManagerRunnerResult != null) {
                return handleJobManagerRunnerResult(
                    jobManagerRunnerResult, executionType);
            } else {
                return CompletableFuture.completedFuture(
                    jobManagerRunnerFailed(
                        jobId, JobStatus.FAILED, throwable));
            }
        },
        getMainThreadExecutor())
        .thenCompose(Function.identity());

    final CompletableFuture<Void> jobTerminationFuture =
        cleanupJobStateFuture.thenCompose(
        cleanupJobState ->
        removeJob(jobId, cleanupJobState)
        .exceptionally(
            throwable ->
            logCleanupErrorWarning(jobId, throwable)));

    FutureUtils.handleUncaughtException(
        jobTerminationFuture,
        (thread, throwable) -> fatalErrorHandler.onFatalError(throwable));
    registerJobManagerRunnerTerminationFuture(jobId, jobTerminationFuture);
}

// 与前几个组件寻找leader的步骤类似，调用grantLeadership方法
public void grantLeadership(UUID leaderSessionID) {
    runIfStateRunning(
        () -> startJobMasterServiceProcessAsync(leaderSessionID),
        "starting a new JobMasterServiceProcess");
}

private void startJobMasterServiceProcessAsync(UUID leaderSessionId) {
    sequentialOperation =
        sequentialOperation.thenRun(
        () ->
        // 如果是valid的leader，执行Runnable action
        runIfValidLeader(
            leaderSessionId,
            ThrowingRunnable.unchecked(
                () ->
                verifyJobSchedulingStatusAndCreateJobMasterServiceProcess(
                    leaderSessionId)),
            "verify job scheduling status and create JobMasterServiceProcess"));

    handleAsyncOperationError(sequentialOperation, "Could not start the job manager.");
}

private void verifyJobSchedulingStatusAndCreateJobMasterServiceProcess(UUID leaderSessionId)
    throws FlinkException {
    try {
        if (jobResultStore.hasJobResultEntry(getJobID())) {
            jobAlreadyDone();
        } else {
            createNewJobMasterServiceProcess(leaderSessionId);
        }
    } catch (IOException e) {...}
}

private void createNewJobMasterServiceProcess(UUID leaderSessionId) throws FlinkException {
    Preconditions.checkState(jobMasterServiceProcess.closeAsync().isDone());

    LOG.debug(
        "Create new JobMasterServiceProcess because we were granted leadership under {}.",
        leaderSessionId);

    jobMasterServiceProcess = jobMasterServiceProcessFactory.create(leaderSessionId);
}

// DefaultJobMasterServiceProcessFactory.java
public JobMasterServiceProcess create(UUID leaderSessionId) {
    return new DefaultJobMasterServiceProcess(
        jobId,
        leaderSessionId,
        jobMasterServiceFactory,
        cause -> createArchivedExecutionGraph(JobStatus.FAILED, cause));
}

public DefaultJobMasterServiceProcess(
    JobID jobId,
    UUID leaderSessionId,
    JobMasterServiceFactory jobMasterServiceFactory,
    Function<Throwable, ArchivedExecutionGraph> failedArchivedExecutionGraphFactory) {
    this.jobId = jobId;
    this.leaderSessionId = leaderSessionId;
    this.jobMasterServiceFuture =
        jobMasterServiceFactory.createJobMasterService(leaderSessionId, this);

    jobMasterServiceFuture.whenComplete(
        (jobMasterService, throwable) -> {
            if (throwable != null) {...} else {
                registerJobMasterServiceFutures(jobMasterService);
            }
        });
}

public CompletableFuture<JobMasterService> createJobMasterService(
    UUID leaderSessionId, OnCompletionActions onCompletionActions) {

    return CompletableFuture.supplyAsync(
        FunctionUtils.uncheckedSupplier(
            () -> internalCreateJobMasterService(leaderSessionId, onCompletionActions)),
        executor);
}

private JobMasterService internalCreateJobMasterService(
    UUID leaderSessionId, OnCompletionActions onCompletionActions) throws Exception {

    final JobMaster jobMaster =
        new JobMaster(...);
	// 开启JobMaster的Rpc终端
    jobMaster.start();

    return jobMaster;
}
```
## 1.5 转换 JobGraph，启动 JobMaster 连接到 RM
### 1.5.1 转换 JobGraph
```java
public JobMaster(
    RpcService rpcService,
    JobMasterId jobMasterId,
    JobMasterConfiguration jobMasterConfiguration,
    ResourceID resourceId,
    JobGraph jobGraph,
    HeartbeatServices heartbeatServices,
    ClassLoader userCodeLoader,)
    throws Exception {
    super(rpcService, RpcServiceUtils.createRandomName(JOB_MANAGER_NAME), jobMasterId);
    this.schedulerNG =
        createScheduler(
        slotPoolServiceSchedulerFactory,
        executionDeploymentTracker,
        jobManagerJobMetricGroup,
        jobStatusListener);
}

private SchedulerNG createScheduler(
    SlotPoolServiceSchedulerFactory slotPoolServiceSchedulerFactory,
    ExecutionDeploymentTracker executionDeploymentTracker,
    JobManagerJobMetricGroup jobManagerJobMetricGroup,
    JobStatusListener jobStatusListener)
    throws Exception {
    final SchedulerNG scheduler =
        slotPoolServiceSchedulerFactory.createScheduler(...);
    return scheduler;
}

// DefaultSlotPoolServiceSchedulerFactory.java
public SchedulerNG createScheduler(
    JobGraph jobGraph,
    Executor ioExecutor,
    SlotPoolService slotPoolService,
    ScheduledExecutorService futureExecutor,
    Time rpcTimeout,
    BlobWriter blobWriter,
    Time slotRequestTimeout)
    throws Exception {
    return schedulerNGFactory.createInstance(...);
}

// DefaultSchedulerFactory.java
public SchedulerNG createInstance(...)
    throws Exception {

    return new DefaultScheduler(
                log,
                jobGraph,
                ioExecutor,
                jobMasterConfiguration,
                schedulerComponents.getStartUpAction(),
                new ScheduledExecutorServiceAdapter(futureExecutor),
                userCodeLoader,
                new CheckpointsCleaner(),
                checkpointRecoveryFactory,
                jobManagerJobMetricGroup,
                schedulerComponents.getSchedulingStrategyFactory())
}

// DefaultScheduler.java
protected DefaultScheduler(...)
    throws Exception {
	
    // 调用父类SchedulerBase的构造器
    super(
        log,
        jobGraph,
        ioExecutor,
        jobMasterConfiguration,
        checkpointsCleaner,
        checkpointRecoveryFactory,
        jobManagerJobMetricGroup,
        executionVertexVersioner,
        initializationTimestamp,
        mainThreadExecutor,
        jobStatusListener,
        executionGraphFactory,
        vertexParallelismStore);

    this.executionSlotAllocator =
        checkNotNull(executionSlotAllocatorFactory)
        .createInstance(new DefaultExecutionSlotAllocationContext());

    this.executionDeployer =
        executionDeployerFactory.createInstance(
        log,
        executionSlotAllocator,
        executionOperations,
        executionVertexVersioner,
        rpcTimeout,
        this::startReserveAllocation,
        mainThreadExecutor);
}

// SchedulerBase.java
public SchedulerBase(
    final Logger log,
    final JobGraph jobGraph,
    final Executor ioExecutor,
    final Configuration jobMasterConfiguration,
    final CheckpointsCleaner checkpointsCleaner,
    final CheckpointRecoveryFactory checkpointRecoveryFactory,
    final JobManagerJobMetricGroup jobManagerJobMetricGroup,
    final ExecutionVertexVersioner executionVertexVersioner,
    long initializationTimestamp,
    final ComponentMainThreadExecutor mainThreadExecutor,
    final JobStatusListener jobStatusListener,
    final ExecutionGraphFactory executionGraphFactory,
    final VertexParallelismStore vertexParallelismStore)
    throws Exception {

    /*
     * 将JobGraph转换为ExecutionGraph 
     * 根据并行度将JobGraph的JobVertex转化为ExecutionVertex
	 * 根据并行度将JobGraph的IntermediateDataSet转化为IntermediateResultPartition
     * 根据上下游的依赖关系根据ExecutionVertex和IntermediateResultPartition生成
     * 创建DefaultExecutionGraph对象后，将JobStatus改为CREATED后，调用钩子函数
     */
    this.executionGraph =
        createAndRestoreExecutionGraph(
        completedCheckpointStore,
        checkpointsCleaner,
        checkpointIdCounter,
        initializationTimestamp,
        mainThreadExecutor,
        jobStatusListener,
        vertexParallelismStore);
}
```
### 1.5.2 启动 JobMaster
```java
protected void onStart() throws JobMasterException {
    try {
        startJobExecution();
    } catch (Exception e) {...}
}

private void startJobExecution() throws Exception {
    validateRunsInMainThread();

    JobShuffleContext context = new JobShuffleContextImpl(jobGraph.getJobID(), this);
    shuffleMaster.registerJob(context);

    startJobMasterServices();

    log.info(
        "Starting execution of job '{}' ({}) under job master id {}.",
        jobGraph.getName(),
        jobGraph.getJobID(),
        getFencingToken());
	// 后续申请资源
    startScheduling();
}
```
####  1.5.2.1 连接到 RM
```java
private void startJobMasterServices() throws Exception {
    try {
        this.taskManagerHeartbeatManager = createTaskManagerHeartbeatManager(heartbeatServices);
        this.resourceManagerHeartbeatManager =
            createResourceManagerHeartbeatManager(heartbeatServices);

        // start the slot pool make sure the slot pool now accepts messages for this leader	
        // 启动slotPoolService
        slotPoolService.start(getFencingToken(), getAddress(), getMainThreadExecutor());

        // job is ready to go, try to establish connection with resource manager
        //   - activate leader retrieval for the resource manager
        //   - on notification of the leader, the connection will be established and
        //     the slot pool will start requesting slots
        // 与RM建立连接
        resourceManagerLeaderRetriever.start(new ResourceManagerLeaderListener());
    } catch (Exception e) {
        handleStartJobMasterServicesError(e);
    }
}

// StandaloneLeaderRetrievalService.java
// standalone指各个组件只有一个处于leader状态
public void start(LeaderRetrievalListener listener) {
    checkNotNull(listener, "Listener must not be null.");

    synchronized (startStopLock) {
        checkState(!started, "StandaloneLeaderRetrievalService can only be started once.");
        started = true;

        // directly notify the listener, because we already know the leading JobManager's
        // address
        listener.notifyLeaderAddress(leaderAddress, leaderId);
    }
}

// ResourceManagerLeaderListener.notifyLeaderAddress()
public void notifyLeaderAddress(final String leaderAddress, final UUID leaderSessionID) {
    runAsync(
        () ->
        notifyOfNewResourceManagerLeader(
            leaderAddress,
            ResourceManagerId.fromUuidOrNull(leaderSessionID)));
}

private void notifyOfNewResourceManagerLeader(
    final String newResourceManagerAddress, final ResourceManagerId resourceManagerId) {
    resourceManagerAddress =
        createResourceManagerAddress(newResourceManagerAddress, resourceManagerId);
	
    // 监听RM的leader的变化，与leader的RM建立连接
    reconnectToResourceManager(
        new FlinkException(
            String.format(
                "ResourceManager leader changed to new address %s",
                resourceManagerAddress)));
}

private void reconnectToResourceManager(Exception cause) {
    closeResourceManagerConnection(cause);
    tryConnectToResourceManager();
}

private void tryConnectToResourceManager() {
    if (resourceManagerAddress != null) {
        connectToResourceManager();
    }
}

private void connectToResourceManager() {
    assert (resourceManagerAddress != null);
    assert (resourceManagerConnection == null);
    assert (establishedResourceManagerConnection == null);

    log.info("Connecting to ResourceManager {}", resourceManagerAddress);

    resourceManagerConnection =
        new ResourceManagerConnection(
        log,
        jobGraph.getJobID(),
        resourceId,
        getAddress(),
        getFencingToken(),
        resourceManagerAddress.getAddress(),
        resourceManagerAddress.getResourceManagerId(),
        futureExecutor);

    resourceManagerConnection.start();
}

public void start() {

    final RetryingRegistration<F, G, S, R> newRegistration = createNewRegistration();
CompletableFuture<RetryingRegistration.RetryingRegistrationResult<G, S, R>> future =
        newRegistration.getFuture();

    future.whenCompleteAsync(
        (RetryingRegistration.RetryingRegistrationResult<G, S, R> result,
         Throwable failure) -> {
            if (failure != null) {...} 
            else {
                if (result.isSuccess()) {
                    targetGateway = result.getGateway();
                    onRegistrationSuccess(result.getSuccess());
                } else if (result.isRejection()) {...} 
                else {...}
            }
        },
        executor);

    return newRegistration;
}

protected void onRegistrationSuccess(final JobMasterRegistrationSuccess success) {
    runAsync(
        () -> {
            // filter out outdated connections
            //noinspection ObjectEquality
            if (this == resourceManagerConnection) {
                establishResourceManagerConnection(success);
            }
        });
}

private void establishResourceManagerConnection(final JobMasterRegistrationSuccess success) {
    final ResourceManagerId resourceManagerId = success.getResourceManagerId();

    // verify the response with current connection
    if (resourceManagerConnection != null
        && Objects.equals(
            resourceManagerConnection.getTargetLeaderId(), resourceManagerId)) {

        log.info(
            "JobManager successfully registered at ResourceManager, leader id: {}.",
            resourceManagerId);

        final ResourceManagerGateway resourceManagerGateway =
            resourceManagerConnection.getTargetGateway();

        final ResourceID resourceManagerResourceId = success.getResourceManagerResourceId();

        establishedResourceManagerConnection =
            new EstablishedResourceManagerConnection(
            resourceManagerGateway, resourceManagerResourceId);

        blocklistHandler.registerBlocklistListener(resourceManagerGateway);
        // 后续通过slotPool与RM建立连接申请slot资源
        slotPoolService.connectToResourceManager(resourceManagerGateway);
        partitionTracker.connectToResourceManager(resourceManagerGateway);

        resourceManagerHeartbeatManager.monitorTarget(
            resourceManagerResourceId,
            new ResourceManagerHeartbeatReceiver(resourceManagerGateway));
    } else {
        log.debug(
            "Ignoring resource manager connection to {} because it's duplicated or outdated.",
            resourceManagerId);
    }
}
```
## 1.6 调度任务申请 slot
### 1.6.1 startScheduling()
```java
// 建立连接后，返回startJobExecution方法进行调度
private void startScheduling() {
    schedulerNG.startScheduling();
}

public final void startScheduling() {
    mainThreadExecutor.assertRunningInMainThread();
    registerJobMetrics(
        jobManagerJobMetricGroup,
        executionGraph,
        this::getNumberOfRestarts,
        deploymentStateTimeMetrics,
        executionGraph::registerJobStatusListener,
        executionGraph.getStatusTimestamp(JobStatus.INITIALIZING),
        jobStatusMetricsSettings);
    operatorCoordinatorHandler.startAllOperatorCoordinators();
    startSchedulingInternal();
}

protected void startSchedulingInternal() {
    log.info(
        "Starting scheduling with scheduling strategy [{}]",
        schedulingStrategy.getClass().getName());
    // 将Job的Status转为RUNNING
    transitionToRunning();
    schedulingStrategy.startScheduling();
}

public void startScheduling() {
    final Set<SchedulingPipelinedRegion> sourceRegions =
        IterableUtils.toStream(schedulingTopology.getAllPipelinedRegions())
        .filter(this::isSourceRegion)
        .collect(Collectors.toSet());
    maybeScheduleRegions(sourceRegions);
}

private void maybeScheduleRegions(final Set<SchedulingPipelinedRegion> regions) {
    final Set<SchedulingPipelinedRegion> regionsToSchedule = new HashSet<>();
    Set<SchedulingPipelinedRegion> nextRegions = regions;
    while (!nextRegions.isEmpty()) {
        nextRegions = addSchedulableAndGetNextRegions(nextRegions, regionsToSchedule);
    }
    // schedule regions in topological order.
    SchedulingStrategyUtils.sortPipelinedRegionsInTopologicalOrder(
        schedulingTopology, regionsToSchedule)
        .forEach(this::scheduleRegion);
}

private void scheduleRegion(final SchedulingPipelinedRegion region) {
    checkState(
        areRegionVerticesAllInCreatedState(region),
        "BUG: trying to schedule a region which is not in CREATED state");
    scheduledRegions.add(region);
    schedulerOperations.allocateSlotsAndDeploy(regionVerticesSorted.get(region));
}
```
### 1.6.2 allocateSlotsAndDeploy()
```java
public void allocateSlotsAndDeploy(final List<ExecutionVertexID> verticesToDeploy) {
    
executionDeployer.allocateSlotsAndDeploy(executionsToDeploy, requiredVersionByVertex);
}

public void allocateSlotsAndDeploy(
    final List<Execution> executionsToDeploy,
    final Map<ExecutionVertexID, ExecutionVertexVersion> requiredVersionByVertex) {

    final List<ExecutionSlotAssignment> executionSlotAssignments =
        allocateSlotsFor(executionsToDeploy);

    waitForAllSlotsAndDeploy(deploymentHandles);
}
```
### 1.6.3 allocateSlotsFor()
```java
private List<ExecutionSlotAssignment> allocateSlotsFor(
    final List<Execution> executionsToDeploy) {
    
    return executionSlotAllocator.allocateSlotsFor(executionAttemptIds);
}

// SimpleExecutionSlotAllocator.java
public List<ExecutionSlotAssignment> allocateSlotsFor(
    List<ExecutionAttemptID> executionAttemptIds) {
    return executionAttemptIds.stream()
        .map(id -> new ExecutionSlotAssignment(id, allocateSlotFor(id)))
        .collect(Collectors.toList());
}

private CompletableFuture<LogicalSlot> allocateSlotFor(ExecutionAttemptID executionAttemptId) {
    if (requestedPhysicalSlots.containsKeyA(executionAttemptId)) {
        return requestedPhysicalSlots.getValueByKeyA(executionAttemptId);
    }
    final SlotRequestId slotRequestId = new SlotRequestId();
    final ResourceProfile resourceProfile = resourceProfileRetriever.apply(executionAttemptId);
    final SlotProfile slotProfile =
        // A slot profile describes the profile of a slot into which a task wants to be scheduled
        // 返回一个SlotProfile
        SlotProfile.priorAllocation(
        resourceProfile,
        resourceProfile,
        Collections.emptyList(),
        Collections.emptyList(),
        Collections.emptySet());
    final PhysicalSlotRequest request =
        new PhysicalSlotRequest(slotRequestId, slotProfile, slotWillBeOccupiedIndefinitely);
    final CompletableFuture<LogicalSlot> slotFuture =
        slotProvider
        .allocatePhysicalSlot(request)
        .thenApply(
        physicalSlotRequest ->
        allocateLogicalSlotFromPhysicalSlot(
            slotRequestId,
            physicalSlotRequest.getPhysicalSlot(),
            slotWillBeOccupiedIndefinitely));
}

public CompletableFuture<PhysicalSlotRequest.Result> allocatePhysicalSlot(
    PhysicalSlotRequest physicalSlotRequest) {
    SlotRequestId slotRequestId = physicalSlotRequest.getSlotRequestId();
    SlotProfile slotProfile = physicalSlotRequest.getSlotProfile();
    ResourceProfile resourceProfile = slotProfile.getPhysicalSlotResourceProfile();

    LOG.debug(
        "Received slot request [{}] with resource requirements: {}",
        slotRequestId,
        resourceProfile);

    Optional<PhysicalSlot> availablePhysicalSlot =
        tryAllocateFromAvailable(slotRequestId, slotProfile);

    CompletableFuture<PhysicalSlot> slotFuture;
    slotFuture =
        availablePhysicalSlot
        .map(CompletableFuture::completedFuture)
        .orElseGet(
        () ->
        // 申请新的slot
        requestNewSlot(
            slotRequestId,
            resourceProfile,
            slotProfile.getPreferredAllocations(),
            physicalSlotRequest
            .willSlotBeOccupiedIndefinitely()));

    return slotFuture.thenApply(
        physicalSlot -> new PhysicalSlotRequest.Result(slotRequestId, physicalSlot));
}

private Optional<PhysicalSlot> tryAllocateFromAvailable(
    SlotRequestId slotRequestId, SlotProfile slotProfile) {
    Collection<SlotSelectionStrategy.SlotInfoAndResources> slotInfoList =
        slotPool.getAvailableSlotsInformation().stream()
        .map(SlotSelectionStrategy.SlotInfoAndResources::fromSingleSlot)
        .collect(Collectors.toList());

    Optional<SlotSelectionStrategy.SlotInfoAndLocality> selectedAvailableSlot =
        slotSelectionStrategy.selectBestSlotForProfile(slotInfoList, slotProfile);

    return selectedAvailableSlot.flatMap(
        slotInfoAndLocality ->
        // 请求slotPool分配空闲的slot
        slotPool.allocateAvailableSlot(
            slotRequestId,
            slotInfoAndLocality.getSlotInfo().getAllocationId(),
            slotProfile.getPhysicalSlotResourceProfile()));
}

//去RM申请新的slot
private CompletableFuture<PhysicalSlot> requestNewSlot(
    SlotRequestId slotRequestId,
    ResourceProfile resourceProfile,
    Collection<AllocationID> preferredAllocations,
    boolean willSlotBeOccupiedIndefinitely) {
    if (willSlotBeOccupiedIndefinitely) {
        return slotPool.requestNewAllocatedSlot(
            slotRequestId, resourceProfile, preferredAllocations, null);
    } else {
        return slotPool.requestNewAllocatedBatchSlot(
            slotRequestId, resourceProfile, preferredAllocations);
    }
}

public CompletableFuture<PhysicalSlot> requestNewAllocatedSlot(
    @Nonnull SlotRequestId slotRequestId,
    @Nonnull ResourceProfile resourceProfile,
    @Nonnull Collection<AllocationID> preferredAllocations,
    @Nullable Time timeout) {
    assertRunningInMainThread();

    log.debug(
        "Request new allocated slot with slot request id {} and resource profile {}",
        slotRequestId,
        resourceProfile);

    final PendingRequest pendingRequest =
        PendingRequest.createNormalRequest(
        slotRequestId, resourceProfile, preferredAllocations);

    return internalRequestNewSlot(pendingRequest, timeout);
}

private CompletableFuture<PhysicalSlot> internalRequestNewSlot(
    PendingRequest pendingRequest, @Nullable Time timeout) {
    internalRequestNewAllocatedSlot(pendingRequest);
}

private void internalRequestNewAllocatedSlot(PendingRequest pendingRequest) {
    // 将请求放在pendingRequests中 
    // Map<SlotRequestId, PendingRequest> pendingRequests
    pendingRequests.put(pendingRequest.getSlotRequestId(), pendingRequest);

    getDeclarativeSlotPool()
        .increaseResourceRequirementsBy(
        ResourceCounter.withResource(pendingRequest.getResourceProfile(), 1));
}
```
## 1.7 RM 分配 Container，创建 TaskManager
### 1.7.1 requestNewWorker()
```java
public void requestNewWorker(WorkerResourceSpec workerResourceSpec) {  
    final TaskExecutorProcessSpec taskExecutorProcessSpec =  
            TaskExecutorProcessUtils.processSpecFromWorkerResourceSpec(  
                    flinkConfig, workerResourceSpec);  
    // 通过resourceManagerDriver申请资源
    final CompletableFuture<WorkerType> requestResourceFuture =  
    resourceManagerDriver.requestResource(taskExecutorProcessSpec);  
unallocatedWorkerFutures.put(requestResourceFuture, workerResourceSpec);
}
```
### 1.7.2 requestResource()
```java
public CompletableFuture<YarnWorkerNode> requestResource(  
	TaskExecutorProcessSpec taskExecutorProcessSpec) {  

	final Optional<TaskExecutorProcessSpecContainerResourcePriorityAdapter.PriorityAndResource>  
        priorityAndResourceOpt =  
                taskExecutorProcessSpecContainerResourcePriorityAdapter  
                        .getPriorityAndResource(taskExecutorProcessSpec);

	// resource中包含taskExecutorProcessSpec
	// 申请container的上下文
	addContainerRequest(resource, priority);  

    return requestResourceFuture;  
}
```
### 1.7.3 YarnTaskExecutorRunner.main()
```java
public static void main(String[] args) {  
    runTaskManagerSecurely(args);  
}
private static void runTaskManagerSecurely(String[] args) {  

// 通过TaskManagerRunner启动TaskManager
TaskManagerRunner.runTaskManagerProcessSecurely(Preconditions.checkNotNull(configuration));  
}

public static void runTaskManagerProcessSecurely(Configuration configuration) { 
  ClusterEntrypointUtils.configureUncaughtExceptionHandler(configuration);  
    try {  
        SecurityUtils.install(new SecurityConfiguration(configuration));  
  
        exitCode =  
                SecurityUtils.getInstalledContext()  
		                // runTaskManager
                        .runSecured(() -> runTaskManager(configuration, pluginManager));  
    } catch (Throwable t) {...}    
    System.exit(exitCode);  
}
```
### 1.7.4 runTaskManager()
```java
public static int runTaskManager(Configuration configuration, PluginManager pluginManager)  
        throws Exception {  
    final TaskManagerRunner taskManagerRunner;  
  
    try {  
        taskManagerRunner =  
                new TaskManagerRunner(  
                        configuration,  
                        pluginManager,  
                        TaskManagerRunner::createTaskExecutorService);  
        // 开启服务
        taskManagerRunner.start();  
    } catch (Exception exception) {...}    
}
```
#### 1.7.4.1 createTaskExecutorService()
```java
public static TaskExecutorService createTaskExecutorService(  
        Configuration configuration,  
        ResourceID resourceID,  
        RpcService rpcService,  
        HighAvailabilityServices highAvailabilityServices,  
        HeartbeatServices heartbeatServices,  
        MetricRegistry metricRegistry,  
        BlobCacheService blobCacheService,  
        boolean localCommunicationOnly,  
        ExternalResourceInfoProvider externalResourceInfoProvider,  
        WorkingDirectory workingDirectory,  
        FatalErrorHandler fatalErrorHandler,  
        DelegationTokenReceiverRepository delegationTokenReceiverRepository)  
        throws Exception {  
  
    final TaskExecutor taskExecutor =  
            startTaskManager(...);  
  
    return TaskExecutorToServiceAdapter.createFor(taskExecutor);  
}

public static TaskExecutor startTaskManager(  
        Configuration configuration,  
        ResourceID resourceID,  
        RpcService rpcService,  
        HighAvailabilityServices highAvailabilityServices,  
        HeartbeatServices heartbeatServices,  
        MetricRegistry metricRegistry,  
        TaskExecutorBlobService taskExecutorBlobService,  
        boolean localCommunicationOnly,  
        ExternalResourceInfoProvider externalResourceInfoProvider,  
        WorkingDirectory workingDirectory,  
        FatalErrorHandler fatalErrorHandler,  
        DelegationTokenReceiverRepository delegationTokenReceiverRepository)  
        throws Exception {  

	// 创建TaskExecutor对象，调用父类构造器开启Rpc终端
	return new TaskExecutor(...);  
}
```
#### 1.7.4.2 TaskManagerRuuner.start()
```java
public void start() throws Exception {  
    synchronized (lock) {  
        startTaskManagerRunnerServices();  
        taskExecutorService.start();  
    }  
}

private void startTaskManagerRunnerServices() throws Exception {  
    synchronized (lock) {  
        rpcSystem = RpcSystem.load(configuration);  
  
        this.executor =  
                Executors.newScheduledThreadPool(  
                        Hardware.getNumberCPUCores(),  
                        new ExecutorThreadFactory("taskmanager-future")); 
		// 为TaskManager创建Rpc环境
        rpcService = createRpcService(configuration, highAvailabilityServices, rpcSystem);  
  
        taskExecutorService =  
                taskExecutorServiceFactory.createTaskExecutor(  
                        this.configuration,  
                        this.resourceId.unwrap(),  
                        rpcService,  
                        highAvailabilityServices,  
                        heartbeatServices,  
                        metricRegistry,  
                        blobCacheService,  
                        false,  
                        externalResourceInfoProvider,  
                        workingDirectory.unwrap(),  
                        this,  
                        delegationTokenReceiverRepository);  
	}
}

// TaskExecutorToServiceAdapter.java
public void start() { 
	// 开启Rpc终端
    taskExecutor.start();  
}
```
## 1.8 开启 TaskExecutor
```java
public void onStart() throws Exception {  
    try {  
        startTaskExecutorServices();  
    } 
    startRegistrationTimeout();  
}

private void startTaskExecutorServices() throws Exception {  
    try {  
        // start by connecting to the ResourceManager  
        resourceManagerLeaderRetriever.start(new ResourceManagerLeaderListener());  
  
        // tell the task slot table who's responsible for the task slot actions  
        taskSlotTable.start(new SlotActionsImpl(), getMainThreadExecutor());  
  
        // start the job leader service  
        jobLeaderService.start(  
                getAddress(), getRpcService(), haServices, new JobLeaderListenerImpl());  
  
        fileCache =  
                new FileCache(  
                        taskManagerConfiguration.getTmpDirectories(),  
                        taskExecutorBlobService.getPermanentBlobService());  
  
        tryLoadLocalAllocationSnapshots();  
    } catch (Exception e) {  
        handleStartTaskExecutorServicesException(e);  
    }  
}
```
### 1.8.1 向 RM 注册
```java
// 类似1.5.2.1，通过resourceManagerLeaderRetriever获取到RM的地址
// 调用TaskExecutor中的establishResourceManagerConnection方法：
private void establishResourceManagerConnection(  
        ResourceManagerGateway resourceManagerGateway,  
        ResourceID resourceManagerResourceId,  
        InstanceID taskExecutorRegistrationId,  
        ClusterInformation clusterInformation) {  
  
    final CompletableFuture<Acknowledge> slotReportResponseFuture =  
            resourceManagerGateway.sendSlotReport(  
                    getResourceID(),  
                    taskExecutorRegistrationId, 
                    // 注册slot 
                    taskSlotTable.createSlotReport(getResourceID()),  
                    taskManagerConfiguration.getRpcTimeout());  

    taskExecutorBlobService.setBlobServerAddress(blobServerAddress);  
  
    establishedResourceManagerConnection =  
            new EstablishedResourceManagerConnection(  
                    resourceManagerGateway,  
                    resourceManagerResourceId,  
                    taskExecutorRegistrationId);  
  
    stopRegistrationTimeout();  
}

public CompletableFuture<Acknowledge> sendSlotReport(  
        ResourceID taskManagerResourceId,  
        InstanceID taskManagerRegistrationId,  
        SlotReport slotReport,  
        Time timeout) {  
    if (workerTypeWorkerRegistration.getInstanceID().equals(taskManagerRegistrationId)) {  
        SlotManager.RegistrationResult registrationResult =  
                // 注册TaskManager
                slotManager.registerTaskManager(  
                        workerTypeWorkerRegistration,  
                        slotReport,  
                        workerTypeWorkerRegistration.getTotalResourceProfile(),  
                        workerTypeWorkerRegistration.getDefaultSlotResourceProfile());  
	    // 注册成功
        if (registrationResult == SlotManager.RegistrationResult.SUCCESS) {  
            WorkerResourceSpec workerResourceSpec =  
                    WorkerResourceSpec.fromTotalResourceProfile(  
                            workerTypeWorkerRegistration.getTotalResourceProfile(),  
                            slotReport.getNumSlotStatus());  
            onWorkerRegistered(workerTypeWorkerRegistration.getWorker(), workerResourceSpec);  
        }  
}

public RegistrationResult registerTaskManager(  
        final TaskExecutorConnection taskExecutorConnection,  
        SlotReport initialSlotReport,  
        ResourceProfile totalResourceProfile,  
        ResourceProfile defaultSlotResourceProfile) {  
    checkInit();  
  
    // we identify task managers by their instance id  
    // 已经注册过
    if (taskExecutorManager.isTaskManagerRegistered(taskExecutorConnection.getInstanceID())) {  
        LOG.debug(  
                "Task executor {} was already registered.",  
                taskExecutorConnection.getResourceID());  
        reportSlotStatus(taskExecutorConnection.getInstanceID(), initialSlotReport);  
        return RegistrationResult.IGNORED;  
    } else {  
	    // 注册失败返回RegistrationResult.REJECTED
        if (!taskExecutorManager.registerTaskManager(  
                taskExecutorConnection,  
                initialSlotReport,  
                totalResourceProfile,  
                defaultSlotResourceProfile)) {  
            LOG.debug(  
                    "Task executor {} could not be registered.",  
                    taskExecutorConnection.getResourceID());  
            return RegistrationResult.REJECTED;  
        }  
  
        // register the new slots  
        // 注册slot
        for (SlotStatus slotStatus : initialSlotReport) {  
            slotTracker.addSlot(  
                    slotStatus.getSlotID(),  
                    slotStatus.getResourceProfile(),  
                    taskExecutorConnection,  
                    slotStatus.getJobID());  
        }  
  
        checkResourceRequirementsWithDelay();  
        return RegistrationResult.SUCCESS;  
    }  
}
```
## 1.9 运行 Task
```java
// 1.6.2中waitForAllSlotsAndDeploy()
private void waitForAllSlotsAndDeploy(final List<ExecutionDeploymentHandle> deploymentHandles) {  
    FutureUtils.assertNoException(  
            assignAllResourcesAndRegisterProducedPartitions(deploymentHandles)  
                    .handle(deployAll(deploymentHandles)));  
}

private BiFunction<Void, Throwable, Void> deployAll(  
        final List<ExecutionDeploymentHandle> deploymentHandles) {  
    return (ignored, throwable) -> {  
        propagateIfNonNull(throwable);  
        for (final ExecutionDeploymentHandle deploymentHandle : deploymentHandles) {  
            final CompletableFuture<LogicalSlot> slotAssigned =  
                    deploymentHandle.getLogicalSlotFuture();  
            checkState(slotAssigned.isDone());  
  
            FutureUtils.assertNoException(  
            // deploy
            slotAssigned.handle(deployOrHandleError(deploymentHandle)));  
        }  
        return null;  
    };  
}

private BiFunction<Object, Throwable, Void> deployOrHandleError(  
        final ExecutionDeploymentHandle deploymentHandle) {  
  
    return (ignored, throwable) -> {  
        if (throwable == null) {  
            deployTaskSafe(execution);  
        } else {  
            handleTaskDeploymentFailure(execution, throwable);  
        }  
        return null;  
    };  
}

private void deployTaskSafe(final Execution execution) {  
    try {  
        executionOperations.deploy(execution);  
    } catch (Throwable e) {  
        handleTaskDeploymentFailure(execution, e);  
    }  
}

public void deploy(Execution execution) throws JobException {  
    execution.deploy();  
}

public void deploy() throws JobException {  

	// 转换ExecutionGraph 
	final TaskDeploymentDescriptor deployment =  
		TaskDeploymentDescriptorFactory.fromExecution(this)  
				.createDeploymentDescriptor(  
						slot.getAllocationId(),  
						taskRestore,  
						producedPartitions.values());  

        // null taskRestore to let it be GC'ed  
	taskRestore = null;  
  
	final TaskManagerGateway taskManagerGateway = slot.getTaskManagerGateway();  
						// 提交
	() -> taskManagerGateway.submitTask(deployment, rpcTimeout), executor)  
		.thenCompose(Function.identity())  
		.whenCompleteAsync(  
				(ack, failure) -> {  
					if (failure == null) {  
						vertex.notifyCompletedDeployment(this);  
					} else {  
						final Throwable actualFailure =  
								ExceptionUtils.stripCompletionException(failure);  
}

public CompletableFuture<Acknowledge> submitTask(  
        TaskDeploymentDescriptor tdd, JobMasterId jobMasterId, Time timeout) {  
        Task task = new Task(...);  
	boolean taskAdded;  
	  
	if (taskAdded) {  
	    task.startTaskThread();}
}

// 开启Task线程
// TaskExecutor.run()
public void run() {  
    try {  
        doRun();  
    } finally {  
        terminationFuture.complete(executionState);  
    }  
}

private void doRun() {
	invokable =  
        loadAndInstantiateInvokable(  
                userCodeClassLoader.asClassLoader(), nameOfInvokableClass, env);
	// 调用StreamTask的invoke方法
	restoreAndInvoke(invokable);
}

public final void invoke() throws Exception {  
    // Allow invoking method 'invoke' without having to call 'restore' before it.  
    if (!isRunning) {  
        LOG.debug("Restoring during invoke will be called.");  
        restoreInternal();  
    }  
  
    // final check to exit early before starting to run  
    ensureNotCanceled();  
  
    scheduleBufferDebloater();  
  
    // let the task do its work  
    getEnvironment().getMetricGroup().getIOMetricGroup().markTaskStart();  
    runMailboxLoop();  
  
    // if this left the run() method cleanly despite the fact that this was canceled,  
    // make sure the "clean shutdown" is not attempted    ensureNotCanceled();  
  
    afterInvoke();  
}

public void runMailboxLoop() throws Exception {  
    mailboxProcessor.runMailboxLoop();  
}

public void runMailboxLoop() throws Exception {  
    suspended = !mailboxLoopRunning;  
  
    final TaskMailbox localMailbox = mailbox;  
  
    checkState(  
            localMailbox.isMailboxThread(),  
            "Method must be executed by declared mailbox thread!");  
  
    assert localMailbox.getState() == TaskMailbox.State.OPEN : "Mailbox must be opened!";  
  
    final MailboxController mailboxController = new MailboxController(this);  
  
    while (isNextLoopPossible()) {  
        // The blocking `processMail` call will not return until default action is available.  
        // 处理任务
        processMail(localMailbox, false);  
        if (isNextLoopPossible()) {  
            mailboxDefaultAction.runDefaultAction(  
                    mailboxController); // lock is acquired inside default action as needed  
        }  
    }  
}
```
# 2 四种 Graph 的转换
- Flink 中有四种 Graph：
	- **StreamGraph**：提交 Job 后生成的最初的 DAG 图
	- **JobGraph**：经过算子链之后生成 `JobGrpah`
	- **ExecutionGraph**：`JobMaster` 将 `StreamGraph` 根据并行度进行 `SubTask` 的拆分，并明确上下游算子间数据的传输方式，形成 `ExecutionGraph`
	- **PhysicalGraph**：将 `ExecutionGraph` 发送给 `TaskManager` 后，`TaskManager` 根据图上的部署运行每个 `SubTask`，此时 `SubTask` 变为 `Task`，最终的物理执行流程称为 `Physical Graph`
## 2.1 StreamGraph
- **StreamNode**：代表 operator 的类，存在并发度、入边、出边等属性
- **StreamEdge**：表示连接两个 `StreamNode` 的边
### 2.1.1 StreamGraph 的创建入口
```java
// 1.4.1中执行execute方法时创建
public StreamGraph getStreamGraph() {  
    return getStreamGraph(true);  
}

public StreamGraph getStreamGraph(boolean clearTransformations) {  
    final StreamGraph streamGraph = getStreamGraph(transformations);  
    if (clearTransformations) {  
        transformations.clear();  
    }  
    return streamGraph;  
}

// 此处传入的是transformations的List
private StreamGraph getStreamGraph(List<Transformation<?>> transformations) {  
    synchronizeClusterDatasetStatus();  
    return getStreamGraphGenerator(transformations).generate();  
}
```
### 2.1.2 transformation 的生成
- transformation 代表了从一个或多个 `DataStream` 生成新的 `DataStream` 的操作
- 常见的 transformation 有 `map，flatmap，filter` 等，这些 transformation 构造出一棵 `StreamTransformation` 树，再转换为 `StreamGraph`
```java
// 以map为例
public <R> SingleOutputStreamOperator<R> map(MapFunction<T, R> mapper) {  

	// 通过反射获取返回值类型
    TypeInformation<R> outType =  
            TypeExtractor.getMapReturnTypes(  
                    clean(mapper), getType(), Utils.getCallLocationName(), true);  
  
    return map(mapper, outType);  
}

public <R> SingleOutputStreamOperator<R> map(  
        MapFunction<T, R> mapper, TypeInformation<R> outputType) {  
    // 返回一个新的DataStream
    return transform("Map", outputType, new StreamMap<>(clean(mapper)));  
}

public <R> SingleOutputStreamOperator<R> transform(  
        String operatorName,  
        TypeInformation<R> outTypeInfo,  
        OneInputStreamOperator<T, R> operator) {  
  
    return doTransform(operatorName, outTypeInfo, SimpleOperatorFactory.of(operator));  
}

protected <R> SingleOutputStreamOperator<R> doTransform(  
        String operatorName,  
        TypeInformation<R> outTypeInfo,  
        StreamOperatorFactory<R> operatorFactory) {  
  
    // read the output type of the input Transform to coax out errors about MissingTypeInfo  
    transformation.getOutputType();  
  
    OneInputTransformation<T, R> resultTransform =  
            new OneInputTransformation<>(  
                    this.transformation,  
                    operatorName,  
                    // 将mapper传入，构建一棵树
                    operatorFactory,  
                    outTypeInfo,  
                    environment.getParallelism(),  
                    false);  
  
    @SuppressWarnings({"unchecked", "rawtypes"})  
    SingleOutputStreamOperator<R> returnStream =  
            new SingleOutputStreamOperator(environment, resultTransform);  
	// 将所有的transformation存入env中
    getExecutionEnvironment().addOperator(resultTransform);  
  
    return returnStream;  
}
```
- 而对于 `union, partition` 等算子，不会生成 `StreamGraph` 和 `StreamNode`：
```java
// PartitionTransformationTranslator.java
private Collection<Integer> translateInternal(  
        final PartitionTransformation<OUT> transformation,  
        final Context context,  
        boolean supportsBatchExchange) {  
    
    for (Integer inputId : context.getStreamNodeIds(input)) {  
        final int virtualId = Transformation.getNewNodeId();  
        // 添加虚拟节点
        streamGraph.addVirtualPartitionNode(  
                inputId, virtualId, transformation.getPartitioner(), exchangeMode);  
        resultIds.add(virtualId);  
    }  
    return resultIds;  
}

public void addVirtualPartitionNode(  
        Integer originalId,  
        Integer virtualId,  
        StreamPartitioner<?> partitioner,  
        StreamExchangeMode exchangeMode) {  
  
    if (virtualPartitionNodes.containsKey(virtualId)) {  
        throw new IllegalStateException(  
                "Already has virtual partition node with id " + virtualId);  
    }  
  
    virtualPartitionNodes.put(virtualId, new Tuple3<>(originalId, partitioner, exchangeMode));  
}
```
### 2.1.3 generate()
```java
public StreamGraph generate() {  
    streamGraph = new StreamGraph(executionConfig, checkpointConfig, savepointRestoreSettings);  
    shouldExecuteInBatchMode = shouldExecuteInBatchMode();  
    configureStreamGraph(streamGraph);  
  
    alreadyTransformed = new IdentityHashMap<>();  
  
    for (Transformation<?> transformation : transformations) { 
	    // 遍历transformations进行转换
        transform(transformation);  
    }  
  
    streamGraph.setSlotSharingGroupResource(slotSharingGroupResources);  
  
    setFineGrainedGlobalStreamExchangeMode(streamGraph);  
  
    for (StreamNode node : streamGraph.getStreamNodes()) {  
        if (node.getInEdges().stream().anyMatch(this::shouldDisableUnalignedCheckpointing)) {  
            for (StreamEdge edge : node.getInEdges()) {  
                edge.setSupportsUnalignedCheckpoints(false);  
            }  
        }  
    }  
  
    final StreamGraph builtStreamGraph = streamGraph;  
  
    alreadyTransformed.clear();  
    alreadyTransformed = null;  
    streamGraph = null;  
  
    return builtStreamGraph;  
}
```
### 2.1.4 transform()
```java
// 将transformation转换为StreamGraph中的StreamNode和StreamEdge
private Collection<Integer> transform(Transformation<?> transform) {  
	// 已经转换过
    if (alreadyTransformed.containsKey(transform)) {  
        return alreadyTransformed.get(transform);  
    }  
  
    LOG.debug("Transforming " + transform);  

    // call at least once to trigger exceptions about MissingTypeInfo  
    transform.getOutputType();  

	// TransformationTranslator用于将Transformation转换为运行时实现
    @SuppressWarnings("unchecked")  
    final TransformationTranslator<?, Transformation<?>> translator =  
            (TransformationTranslator<?, Transformation<?>>)  
                    translatorMap.get(transform.getClass());  
  
    Collection<Integer> transformedIds;  
    if (translator != null) {  
	    // 通过translator进行翻译
        transformedIds = translate(translator, transform);  
    } else {  
        transformedIds = legacyTransform(transform);  
    }  
  
    // need this check because the iterate transformation adds itself before  
    // transforming the feedback edges    if (!alreadyTransformed.containsKey(transform)) {  
        alreadyTransformed.put(transform, transformedIds);  
    }  
  
    return transformedIds;  
}

private Collection<Integer> translate(  
        final TransformationTranslator<?, Transformation<?>> translator,  
        final Transformation<?> transform) {  
    
    return shouldExecuteInBatchMode  
            ? translator.translateForBatch(transform, context)  
            // 流式
            : translator.translateForStreaming(transform, context);  
}

public final Collection<Integer> translateForStreaming(  
        final T transformation, final Context context) {  
    checkNotNull(transformation);  
    checkNotNull(context);  
  
    final Collection<Integer> transformedIds =  
            translateForStreamingInternal(transformation, context);  
    configure(transformation, context);  
  
    return transformedIds;  
}

// OneInputTransformationTranslator.java
public Collection<Integer> translateForStreamingInternal(  
        final OneInputTransformation<IN, OUT> transformation, final Context context) {  
    return translateInternal(  
            transformation,  
            transformation.getOperatorFactory(),  
            transformation.getInputType(),  
            transformation.getStateKeySelector(),  
            transformation.getStateKeyType(),  
            context);  
}

protected Collection<Integer> translateInternal(  
        final Transformation<OUT> transformation,  
        final StreamOperatorFactory<OUT> operatorFactory,  
        final TypeInformation<IN> inputType,  
        @Nullable final KeySelector<IN, ?> stateKeySelector,  
        @Nullable final TypeInformation<?> stateKeyType,  
        final Context context) {  
        
    final StreamGraph streamGraph = context.getStreamGraph();  
    final String slotSharingGroup = context.getSlotSharingGroup();  
    final int transformationId = transformation.getId();  
    final ExecutionConfig executionConfig = streamGraph.getExecutionConfig();  
	  // 添加StreamNode
	  // 新建一个StreamNode对象通过addNode方法添加到Mao中
    streamGraph.addOperator(  
            transformationId,  
            slotSharingGroup,  
            transformation.getCoLocationGroupKey(),  
            operatorFactory,  
            inputType,  
            transformation.getOutputType(),  
            transformation.getName());   

	// 添加StreamEdge
    for (Integer inputId : context.getStreamNodeIds(parentTransformations.get(0))) {  
        streamGraph.addEdge(inputId, transformationId, 0);  
    }  
  
    if (transformation instanceof PhysicalTransformation) {  
        streamGraph.setSupportsConcurrentExecutionAttempts(  
                transformationId,  
                ((PhysicalTransformation<OUT>) transformation)  
                        .isSupportsConcurrentExecutionAttempts());  
    }  
  
    return Collections.singleton(transformationId);  
}
```
### 2.1.5 一个 transformation 树的例子
```java
env.socketTextStream().flatMap().shuffle().filter().print()
```
- 生成的转换树如下：
![[transformation_tree.svg]]
- 每个 transformation 通过 input 指针指向上游的 transformation，通过 `generate()` 生成 StreamGraph
- 首先处理 Source 生成 Source 的 StreamNode
- 处理 FlatMap，生成 FlatMap 的 StreamNode，以及 StreamEdge 连接上游
- 处理 shuffle，不生成实际节点，将分区信息存在 virtualPartitionNodes 中
- 处理 Filter，创建节点和 FlatMap 相连，将分区信息写人 StreamEdge 中
- 最后处理 Sink
## 2.2 JobGraph
- **JobVertex**：多个 `StreamNode` 可能通过算子链组合起来形成一个 `JobVertex`，即一个 `JobVertex` 包括一个或多个 `operator`，`JobVertex` 的输入是 `JobEdge`，输出是 `IntermediateDataSet`
- **IntermediateDataSet**：经过 `operator` 产生的数据集
- **JobEdge**：代表 `JobGraph` 中的一条数据传输通道，数据通过 `IntermediateDataSet` 传向 `JobEdge`
# 3 DataStream窗口
## 3.1 窗口的定义
- 窗口
	- 窗口是 **Flink** 中的抽象类，有两个子类：
		- `GlobalWindow`：计数窗口
		- `TimeWindow`：时间窗口，有 `start` 和 `end` 两个参数
	- 窗口功能用来对流进行有限范围的计算，如统计过去 1 个小时用户的访问量
	- 窗口有两个属性：大小和关闭时机

- 窗口的分类：
- 按功能分类
	- 滚动：
		- `size = slide`, 两个窗口之间无缝衔接
		- 计数：`countWindow(long size))`
		- 时间：`xxxTumblingWindows(long size)`
![[tumbling-windows.svg]]
	- 滑动 
			- 定义 `size`, `slide`
			- `size > slide` 时，滑动的距离小，会出现重复计算的情况
			- `size < slide` 时，滑动的距离大，会出漏算的情况
			- `size = slide` 时等价于滚动
			- 计数：`countWindow(long size, long slide)`
			- 时间：`xxxSlidingWindows(long size, long slide)`
![[sliding-windows.svg]]
	- 会话
		- 会话窗口只有时间类型
		- 会话窗口的 **size** 不确定
		- 定义一个 `gap`，如果两条数据间的间隔超过 `gap`，就会创建一个新窗口，关闭旧窗口后开始计算
		- `xxxSessionWindows.withXxxGap(Time size)`
![[session-windows.svg]]
- 按并行度分类
	- **全局**窗口：调用了窗口算子后，计算算子的并行度只能是 1，所有的数据都会进入到同一个窗口
	- **keyed** 窗口：
		- 调用了窗口算子后，计算的算子的并行度可以是 N
		- 在每个并行度上，都会按照 key 进行开窗
		- 相同 key 的数据，才会进入同一个窗口
		- 必须先 keyBy，再开窗
## 3.2 窗口的 API
### 3.2.1 计数窗口
- 计数滚(滑)动窗口
	- 全局：`SingleOutputStreamOperator.countWindowAll(long size, long slide)`
	- Keyed：`KeyedStream.countWindow(long size, lone slide)`
- 全局窗口会自动进行 keyby，将所有的数据放在同一个窗口中，因此不能设置并行度
```java
env.socketTextStream()
   .map()
   .countWindowAll() // 将数据收集到窗口中
   /*
    * IN: 输入的数据类型
    * OUT: 指定输出的数据类型
    * W: 窗口类型
    * context: 存储window的元数据
    */
   .process(new ProcessAllWindowFunction<IN, OUT, W extends Window>() {
		@override
		public void process(Context context, Iterable<IN> elements, Collector<OUT> out)}
   ) // 对收集到的数据进行处理

env.socketTextStream()
   .map()
   .keyBy()
   .countWindow() // keyBy后的流count时不加all
   /*
    * s: 分组的key
    * 此时process中的函数也不带all
    */
   .process(new ProcessWindowFunction<IN, OUT, KEY, W extends Window>() {
		@override
		public void process(KEY key, Context context, Iterable<IN> elements, Collector<OUT> out)}
   ) // 对收集到的数据进行处理
```
### 3.2.2 时间窗口
- 全局窗口(ProcessingTime，EventTime)
	- `windowAll(WindowAssigner)`
		- `WindowAssigner` 有许多子类
			- `TumblingProcessingTimeWindows.of(Time size)` 滚动时间
			- `SlidingProcessingTimeWindows.of(Time size)` 滑动时间
			- `ProcessingTimeSessionWindows.withGap(Time size)` 会话
		- 滚(滑)动窗口也可以指定 `EventTime` 用于搭配 watermark
		- 会话窗口也可以使用 `withDynamicGap`
- Keyed 窗口调用 `window` 方法
```java
env.socketTextStream()  
        .map()  
        .windowAll(  
		        // 滚动
                TumblingProcessingTimeWindows.of(Time.seconds(long seconds))
                // 滑动
                SlidingProcessingTimeWindows.of(...)  
		        // 会话
		        ProcessingTimeSessionWindows.withGap(Time.seconds(...))
        )  
        .process(new ProcessAllWindowFunction<IN, OUT, W extends Window>() {  
            @Override  
            public void process(Context context, Iterable<IN> elements, Collector<String> out) throws Exception {  
                // 滚动窗口时间范围起始时间从ts为0开始，有数据才会触发窗口的构造
                // 通过context可以获取到当前窗口，从窗口中可以获取到时间段的起始和结束时间的ts
                // 滑动窗口的时间范围不从ts为0开始
        })  
        .print();
```
### 3.2.3 滚动时间窗口底层实现
- 常用的只有滚动时间窗口，针对时间窗口进行分析
- **windowAll()**
```java
public <W extends Window> AllWindowedStream<T, W> windowAll(  
        WindowAssigner<? super T, W> assigner) {  
    return new AllWindowedStream<>(this, assigner);  
}

public AllWindowedStream(DataStream<T> input, WindowAssigner<? super T, W> windowAssigner) {  
	// NullByteKeySelector中getKey方法直接return 0
	// 强制keyBy
    this.input = input.keyBy(new NullByteKeySelector<T>());  
    this.windowAssigner = windowAssigner;  
    this.trigger = windowAssigner.getDefaultTrigger(input.getExecutionEnvironment());  
}
```
- **getDeafaultTrigger()**
```java
public Trigger<Object, TimeWindow> getDefaultTrigger(StreamExecutionEnvironment env) {  
    return ProcessingTimeTrigger.create();  
}

public static ProcessingTimeTrigger create() {  
    return new ProcessingTimeTrigger();  
}
```
- **process**()
```java
public <R> SingleOutputStreamOperator<R> process(ProcessAllWindowFunction<T, R, W> function) {  
	// 返回调用位置和时间
    String callLocation = Utils.getCallLocationName();  
    function = input.getExecutionEnvironment().clean(function);  
    TypeInformation<R> resultType =  
            getProcessAllWindowFunctionReturnType(function, getInputType());  
    return apply(  
            new InternalIterableProcessAllWindowFunction<>(function), resultType, callLocation);  
}

private <R> SingleOutputStreamOperator<R> apply(  
        InternalWindowFunction<Iterable<T>, R, Byte, W> function,  
        TypeInformation<R> resultType,  
        String callLocation) {  
  
    String udfName = "AllWindowedStream." + callLocation;  
  
    String opName;  
    KeySelector<T, Byte> keySel = input.getKeySelector();  
  
    WindowOperator<Byte, T, Iterable<T>, R, W> operator;  
  
    if (evictor != null) {...} 
    else {  
        ListStateDescriptor<T> stateDesc =  
                new ListStateDescriptor<>(  
                        "window-contents",  
                        input.getType()  
                                .createSerializer(getExecutionEnvironment().getConfig()));  
  
        opName =  
                "TriggerWindow("  
                        + windowAssigner  
                        + ", "  
                        + stateDesc  
                        + ", "  
                        + trigger  
                        + ", "  
                        + udfName  
                        + ")";  
  
        operator =  
                new WindowOperator<>(  
                        windowAssigner,  
                        windowAssigner.getWindowSerializer(  
                                getExecutionEnvironment().getConfig()),  
                        keySel,  
                        input.getKeyType()  
                                .createSerializer(getExecutionEnvironment().getConfig()),  
                        stateDesc,  
                        function,  
                        trigger,  
                        allowedLateness,  
                        lateDataOutputTag);  
    }  
  
    return input.transform(opName, resultType, operator).forceNonParallel();  
}
```
- WindowOperator.processElement()
```java
// WindowOperator用于实现基于WindowAssigner和Trigger的逻辑
// 当接收到元素时，通过WindowAssigner将元素放在pane中，pane中存放的是同一个window中相同key的元素
// 当trigger处于fire状态时，调用InternalWindowFunction处理trigger所属的pane
public void processElement(StreamRecord<IN> element) throws Exception {  
	// 分配window
    final Collection<W> elementWindows =  
            windowAssigner.assignWindows(  
                    element.getValue(), element.getTimestamp(), windowAssignerContext);  
  
    // if element is handled by none of assigned elementWindows  
    boolean isSkippedElement = true;  
  
    final K key = this.<K>getKeyedStateBackend().getCurrentKey();  
  
    if (windowAssigner instanceof MergingWindowAssigner) {...}
    // TumblingProcessingTimeWindows不是MergingWindowAssigner的子类
    } else {  
        for (W window : elementWindows) {  
  
            // drop if the window is already late  
            if (isWindowLate(window)) {  
                continue;  
            }  
            isSkippedElement = false;  
  
            windowState.setCurrentNamespace(window);  
            windowState.add(element.getValue());  
  
            triggerContext.key = key;  
            triggerContext.window = window;  
			// 处理元素
            TriggerResult triggerResult = triggerContext.onElement(element);  
			// 触发器激活时进行计算
			// 调用userFunction.process()
            if (triggerResult.isFire()) {  
                ACC contents = windowState.get();  
                if (contents == null) {  
                    continue;  
                }  
                emitWindowContents(window, contents);  
            }  
  
            if (triggerResult.isPurge()) {  
                windowState.clear();  
            }  
            registerCleanupTimer(window);  
        }  
    }  
}
```
- TumblingProcessingTimeWindows.assignWindows()
```java
public Collection<TimeWindow> assignWindows(  
        Object element, long timestamp, WindowAssignerContext context) {  
    final long now = context.getCurrentProcessingTime();  
    if (staggerOffset == null) {  
        staggerOffset =  
                windowStagger.getStaggerOffset(context.getCurrentProcessingTime(), size);  
    }  
    // 返回start + size
    long start =  
            TimeWindow.getWindowStartWithOffset(  
                    now, (globalOffset + staggerOffset) % size, size);  
    return Collections.singletonList(new TimeWindow(start, start + size));  
}
```
- WindowOperator.onElement()
```java
public TriggerResult onElement(StreamRecord<IN> element) throws Exception {  
    return trigger.onElement(element.getValue(), element.getTimestamp(), window, this);  
}

public TriggerResult onElement(  
        Object element, long timestamp, TimeWindow window, TriggerContext ctx) {  
	// 通过internalTimerService注册Timer
    ctx.registerProcessingTimeTimer(window.maxTimestamp());  
    return TriggerResult.CONTINUE;  
}

public void registerProcessingTimeTimer(long time) {  
    internalTimerService.registerProcessingTimeTimer(window, time);  
}

public void registerProcessingTimeTimer(N namespace, long time) {  
    InternalTimer<K, N> oldHead = processingTimeTimersQueue.peek();  
    if (processingTimeTimersQueue.add(  
            new TimerHeapInternalTimer<>(time, (K) keyContext.getCurrentKey(), namespace))) {  
        long nextTriggerTime = oldHead != null ? oldHead.getTimestamp() : Long.MAX_VALUE;  
        // check if we need to re-schedule our timer to earlier  
        if (time < nextTriggerTime) {  
            if (nextTimer != null) {  
                nextTimer.cancel(false);  
            }
            // 触发trigger  
            nextTimer = processingTimeService.registerTimer(time, this::onProcessingTime);  
        }  
    }  
}

public TriggerResult onProcessingTime(long time, TimeWindow window, TriggerContext ctx) {  
    return TriggerResult.FIRE;  
}
```
- 窗口的划分
```java
public Collection<TimeWindow> assignWindows(  
        Object element, long timestamp, WindowAssignerContext context) {  
    if (timestamp > Long.MIN_VALUE) { 
        // stagger v. 蹒跚，摇晃 
        if (staggerOffset == null) {  
            staggerOffset =  
                    windowStagger.getStaggerOffset(context.getCurrentProcessingTime(), size);  
        }  
        // Long.MIN_VALUE is currently assigned when no timestamp is present  
        long start =  
                TimeWindow.getWindowStartWithOffset(  
                        timestamp, (globalOffset + staggerOffset) % size, size);  
		// end = start + size	
        return Collections.singletonList(new TimeWindow(start, start + size));  
    } 
}

public static long getWindowStartWithOffset(long timestamp, long offset, long windowSize) {  
    final long remainder = (timestamp - offset) % windowSize;  
    // handle both positive and negative cases  
    if (remainder < 0) {  
        return timestamp - (remainder + windowSize);  
    } else {  
        return timestamp - remainder;  
    }  
}
```
### 3.2.4 窗口的聚合
- 窗口的聚合分为滚动聚合和非滚动聚合
- 滚动聚合的特征是每一个进入窗口的元素都会触发一次窗口的聚合函数，等到窗口关闭时再触发计算，将结果输出到下游，<font color='red'>时效性好</font>
#### 3.2.4.1 常规聚合
- 开窗后进行 `sum, max, min, minBy, maxBy` 等算子都是滚动聚合
#### 3.2.4.2 Reduce
- 聚合逻辑不是 `sum` 或 `max`，`min`，可以通过 `reduce` 来自定义
```java
env.socketTextStream()  
        .map()  
        .keyBy()  
        .countWindow()   
        .reduce( new ReduceFunction<T>() {
			 @Override  
			 public T reduce(T value1, T value2)throws Exception {  
		        // value1是第一个加的数
		        // reduce只能返回和输入相同类型的数据
			}
	    })  
        .print();
```
#### 3.2.4.3 Aggregate
- `aggregate` 可以计算输出与输入类型不一致的情况
- `aggregate()` 需要传入一个 `AggregateFunction`
```java
/* 
 * IN: 需要聚合的数据类型
 * ACC：累加器的类型(中间聚合状态)
 * OUT: 聚合结果
 */
public interface AggregateFunction<IN, ACC, OUT> extends Function, Serializable {
	/*
	 * 创建一个新的累加器，开启一个新的聚合
	 * 聚合器是正在进行的聚合的状态
	 */
	ACC createAccumulator();
	/*
	 * @param value: The value to add
	 *        accumulator: The accumulator to add the value to
	 * @return The accumulator with the updated state
	 */
	ACC add(IN value, ACC accumulator);
	// 返回累加器的结果
	OUT getResult(ACC accumulator);
}
```
- 例子
```java
// 输入的是POJO，转换为Integer输出
env.socketTextStream()  
        .map()  
        .keyBy()  
        .countWindow()  
        .aggregate(new AggregateFunction<IN, Integer, Integer>() {  
            @Override  
            public Integer createAccumulator() {  
                return 0;  
            }  
  
            @Override  
            public Integer add(IN value, Integer accumulator) {  
                return accumulator + value.getV();  
            }  
  
            @Override  
            public Integer getResult(Integer accumulator) {  
                return accumulator;  
            }  
  
            // 用于批处理，此处不重写  
            @Override  
            public Integer merge(Integer a, Integer b) {  
                return null;  
            }  
        })  
        .print();
```
#### 3.2.4.4 在聚合过程中获取时间范围
- 之前使用 `process` 方法时，函数参数中有一个上下文参数，从其中可以获取到时间范围
- 而在聚合时，`reduce` 和 `aggregate` 方法都没有携带上下文
- 解决方法：`reduce` 和 `aggregate` 方法都有一个重载的方法，根据是否经过 hash，可以传入一个 `AllWindowFunction` 或 `WindowFunction`
```java
/* 
 * IN: ReduceFunction或AggregateFunction的输出
 * 时间范围可以通过W获取到
 * 在窗口关闭时只计算一次，可以对上次聚合的结果再进行一次处理
 * 传给WindowFunction时，迭代器中存放的是之前聚合的结果，因此只有一条数据
 */
public interface WindowFunction<IN, OUT, KEY, W extends Window> extends Function, Serializable {
	void apply(KEY key, W window, Iterable<IN> input, Collector<OUT> out) throws Exception;
}
```
- reduce
```java
.reduce((ReduceFunction<IN>) (v1, v2) -> {
	v2.setV(v1.getV() + v2.getV()); 
	return v2; 
	},  
	// xxxWindow的函数表示当窗口关闭时才会执行一次  
	new WindowFunction<WaterSensor, String, String, TimeWindow>() {  
		@Override  
		public void apply(String s, TimeWindow window, Iterable<IN> input, Collector<String> out) throws Exception {  
			out.collect(...);  
		}  
	});
```
- aggregate
```java
.aggregate(new AggregateFunction<WaterSensor, Integer, String>() {  
              ...
           }  
        , (WindowFunction<String, String, String, TimeWindow>) (s, window, input, out) -> {  
            out.collect(...);  
        });
```
#### 3.2.4.5 非滚动聚合
- 窗口关闭时才会进行聚合
- 需要通过 process()实现
- 一般用于实现 TopN 需求
```java
.process(new ProcessAllWindowFunction<WaterSensor, String, GlobalWindow>() {  
    @Override  
    public void process(Context context, Iterable<IN> elements, Collector<OUT> out) throws Exception {  
	    // StreamSupport.stream需要传入一个Spliterator对象以及布尔型的parallel
	    // Spliterator类似并行模式下的Iterator
        out.collect("topN" + StreamSupport.stream(elements.spliterator(), true)  
                .sorted((o1, o2) -> -Integer.compare(o1.vc, o2.vc))  
                .limit(N)  
                .collect(Collectors.toList()));  
    }  
})
```
# 4 水印
## 4.1 时间语义
- Flink 中存在两种时间语义
	- `ProcessingTime`：参考计算机时钟
	- `EventTime`：参考数据中的时间
- 在一些情况下，我们需要以处理数据的时间为准，例如通过 DS 调度昨天的数据时，操作时间是今天，但是要放在昨天的分区
## 4.2 Watermark 概述
- 如果要参考数据中的时间，数据中就必须携带时间，这种基于 **EventTime** 计算场景下的时间就被称为水印
- 使用场景：
	- 事件时间窗口
	- 事件事件的定时器
- 特点：
	- 先有数据，然后从中加工出水印
	- 水印随数据向下游流动，会广播给下游的每一个算子
	- 可以从任何位置产生
- 水印策略
```java
// WatermarkStrategy中的静态方法	
// 无水印
/*
 * NoWatermarksGenerator实现了WatermarkGenerator
 * NoWatermarksGenerator中onEvent和onPeriodicEmit方法均为空，所以不会产生水印
 */
static <T> WatermarkStrategy<T> noWatermarks() {  
    return (ctx) -> new NoWatermarksGenerator<>();  
}
// 单调水印
static <T> WatermarkStrategy<T> forMonotonousTimestamps() {  
    return (ctx) -> new AscendingTimestampsWatermarks<>();  
}
public AscendingTimestampsWatermarks() {  
    super(Duration.ofMillis(0));  
}
/*
 * AscendingTimestampsWatermarks继承了父类BoundedOutOfOrdernessWatermarks
 * 调用父类的构造器
 */
 public BoundedOutOfOrdernessWatermarks(Duration maxOutOfOrderness) {  
    checkNotNull(maxOutOfOrderness, "maxOutOfOrderness");  
    checkArgument(!maxOutOfOrderness.isNegative(), "maxOutOfOrderness cannot be negative");  
  
    this.outOfOrdernessMillis = maxOutOfOrderness.toMillis();  
  
    // start so that our lowest watermark would be Long.MIN_VALUE.  
    // 初始化时将最大的ts设置为long的最小值
    this.maxTimestamp = Long.MIN_VALUE + outOfOrdernessMillis + 1;  
}
// 收到数据时更新最大ts的值，只有大于当前ts的值时才会更新最大值，因此是单调的
public void onEvent(T event, long eventTimestamp, WatermarkOutput output) {  
    maxTimestamp = Math.max(maxTimestamp, eventTimestamp);  
}
public void onPeriodicEmit(WatermarkOutput output) {  
	// 周期(200ms)向下游发送maxTimestamp - 1的水印
    output.emitWatermark(new Watermark(maxTimestamp - outOfOrdernessMillis - 1));  
}
// 默认值位置
public static final ConfigOption<Duration> AUTO_WATERMARK_INTERVAL =  
        key("pipeline.auto-watermark-interval")  
                .durationType()  
                .defaultValue(Duration.ofMillis(200))  
                .withDescription(  
                        "The interval of the automatic watermark emission. Watermarks are used throughout"  
                                + " the streaming system to keep track of the progress of time. They are used, for example,"  
                                + " for time based windowing.");
// 可以通过env.getConfig.setAutoWatermarkInterval(long interval)设置自动发送时间

/* 
 * 自定义水印策略
 * 调用静态方法forGenerator()，传入一个WatermarkGeneratorSupplier的匿名实现类对象，重写createWatermarkGenerator方法，return一个自己定义的水印生成器对象即可
 */
static <T> WatermarkStrategy<T> forGenerator(WatermarkGeneratorSupplier<T> generatorSupplier) {  
    return generatorSupplier::createWatermarkGenerator;  
}

```
- 选择水印策略后，通过 `withTimestampAssigner()` 指定如何从水印中提取时间
- 生成示例
```java
WatermarkStrategy<T> strategy = WatermarkStrategy 
		// 单调水印策略，生成的值为 maxTimestamp - 1
        .<T>forMonotonousTimestamps()  
        .withTimestampAssigner(new SerializableTimestampAssigner<T>() {  
            @Override  
            public long extractTimestamp(T element, long recordTimestamp) {  
                // 指定提取水印方法
            }  
        });
// 为算子指定策略
.assignTimestampsAndWatermarks(strategy)
.process(new ProcessFunction<T, OUT>() {  
    @Override  
    public void processElement(T value, Context ctx, Collector<OUT> out) throws Exception { 
    /*
     * ctx.timestamp()可以获取数据中的ts
     * ctx.timerService().currentWatermark()可以获取到上一次计算的水印
     * ctx.timerService().currentProcessingTime()可以获取当前计算机时间
    */
    ...
    }  
});
```
- 水印的递增
```java
// BoundedOutOfOrdernessWatermarks
public void onPeriodicEmit(WatermarkOutput output) {  
    output.emitWatermark(new Watermark(maxTimestamp - outOfOrdernessMillis - 1));  
}

// WatermarkToDataOutput
public void emitWatermark(Watermark watermark) {  
    final long newWatermark = watermark.getTimestamp();  
    // 只有新的 Wm 大于上次的 Wm 才会发射
    if (newWatermark <= maxWatermarkSoFar) {  
        return;  
    }  
  
    maxWatermarkSoFar = newWatermark;  
    watermarkEmitted.updateCurrentEffectiveWatermark(maxWatermarkSoFar);  
  
    try {  
        markActiveInternally();  
  
        output.emitWatermark(  
                new org.apache.flink.streaming.api.watermark.Watermark(newWatermark));  
    } 
}
```
## 4.3 基于时间窗口的应用
```java
/*
 * 窗口的大小指定为5
 * 事件时间窗口的时间需要通过ts来计算
 * 假设ts从0开始，时间长度为5s，则ts达到5000时才会触发计算
 * 而数据落入哪个窗口，取决于数据的时间属性是否在窗口的时间范围内，和水印无关
 * 即watermark = 4999的数据，它的ts已经达到了5000，不在[0, 5000)的范围内，不属于第一个窗口
 * 当达到一个size时，窗口在计算后就会关闭，之后再收到的ts<size的数据就变成迟到数据
 * 而窗口未关闭时，出现的比已经接收到的数据的ts小的数据，如果在该窗口范围内，仍然有效
 */
.windowAll(TumblingEventTimeWindows.of(Time.seconds(5L)))  
.process(...);
```
## 4.4 处理迟到的数据
- 可以通过三种方式
	- 针对水印：使用 `BoundedOutOfOrdernessWatermarks` 水印策略，传入一个延迟，推迟窗口的计算时间
	- 针对窗口：使用 `..allowedLateness(Time size)`，实现窗口的延时关闭，窗口到达计算时间时正常计算，但是不会关闭。此时窗口每进入一条数据，都会再进行一次计算
	- 侧流：`.sideOutputLateData(OutputTag<T> outputTag)` 可以将迟到的数据通过侧流来获取处理
```java
// 方式一
WatermarkStrategy<IN> strategy = WatermarkStrategy  
        .<IN>forBoundedOutOfOrderness(Duration naxOutOfOrderness)  
        .withTimestampAssigner((SerializableTimestampAssigner<IN>) (element, recordTimestamp) -> element.getTs());

// 方式二
env.allowedLateness(Time lateness)

// 方式三
.sideOutputLateData(OutputTag<T> outputTag)
// 从主流中获取侧流
.getSideOutput(OutputTag<X> sideOutputTag)
```
## 4.5 多并行度下发送水印
- 数据到达时，先打印一个值
- 水印在数据后到达，根据两个 Task 发送的水印，取最小更新值
![[Drawing 2023-11-15 18.39.18.excalidraw.svg]]
## 4.6 水印超时时间的设置
- 如果上游有多个 `Task`，其中部分 `Task`由于长时间收不到数据而不更新水印。由于下游 `Task` 会获取上游中最小的水印来更新自己的水印，这种情况会拖累下游 `Task` 的计算
- 通过设置超时时间，长时间不产生水印的 `Task` 会被标记为 `Idle`
```java
WatermarkStrategy<T> watermarkStrategy = WatermarkStrategy
            .<T>forMonotonousTimestamps()
            .withTimestampAssigner( (e, ts) -> e.getTs())
           .withIdleness(Duration idleTimeout)

default WatermarkStrategy<T> withIdleness(Duration idleTimeout) {  
	// 返回一个WatermarkStrategyWithIdleness对象
    return new WatermarkStrategyWithIdleness<>(this, idleTimeout);  
}

// WatermarkStrategyWithIdleness对应的Generator是WatermarksWithIdleness
@Override  
public void onEvent(T event, long eventTimestamp, WatermarkOutput output) {  
	// 当处理了数据时，将isIdleNow置为false
    watermarks.onEvent(event, eventTimestamp, output);  
    // activity会将内部的counter+1，通过检查counter和上次的值是否一致，就可以判断是否idle
    idlenessTimer.activity();  
    isIdleNow = false;  
}  
  
@Override  
public void onPeriodicEmit(WatermarkOutput output) {  
    if (idlenessTimer.checkIfIdle()) {  
        if (!isIdleNow) {  
	        // 标记为idle
            output.markIdle();  
            isIdleNow = true;  
        }  
    } else {  
        watermarks.onPeriodicEmit(output);  
    }  
}
```
## 4.7 WindowJoin
- 对两个流先 `join` 再开窗，与 `connect` 类似，但是是通过一个方法同时处理两种流
```java
ds1.join(ds2)
	// 类似sql join时的on
	.where(KeySelector keySelector)
	// 类似join on的条件
	.equalTo(KeySelector keySelector)

// join方法返回一个JoinedStream
public <T2> JoinedStreams<T, T2> join(DataStream<T2> otherStream) {  
    return new JoinedStreams<>(this, otherStream);  
}

// JoinedStream的where()方法返回一个Where对象
public <KEY> Where<KEY> where(KeySelector<T1, KEY> keySelector) {  
    final TypeInformation<KEY> keyType =  
            TypeExtractor.getKeySelectorTypes(keySelector, input1.getType());  
    return where(keySelector, keyType);  
}

public <KEY> Where<KEY> where(KeySelector<T1, KEY> keySelector, TypeInformation<KEY> keyType) {  
    return new Where<>(input1.clean(keySelector), keyType);  
}

Where(KeySelector<T1, KEY> keySelector1, TypeInformation<KEY> keyType) {  
    this.keySelector1 = keySelector1;  
    this.keyType = keyType;  
}

// equals方法获取到Where对象中的key，进行对比
public EqualTo equalTo(KeySelector<T2, KEY> keySelector) {  
    final TypeInformation<KEY> otherKey =  
            TypeExtractor.getKeySelectorTypes(keySelector, input2.getType());  
    return equalTo(keySelector, otherKey);  
}
// 返回EqualTo对象
public EqualTo equalTo(KeySelector<T2, KEY> keySelector, TypeInformation<KEY> keyType) {  
    requireNonNull(keySelector);  
    requireNonNull(keyType);  
  
    if (!keyType.equals(this.keyType)) {  
        throw new IllegalArgumentException(  
                "The keys for the two inputs are not equal: "  
                        + "first key = "  
                        + this.keyType  
                        + " , second key = "  
                        + keyType);  
    }  
  
    return new EqualTo(input2.clean(keySelector));  

// EqualTo对象有window()方法，可以进行后续处理
}
```
## 4.8 IntervalJoin
- `IntervalJoin` 不需要在窗口中 `join`
- 一个流中的数据可以和对面流中<font color='red'>指定时间范围间隔</font>内的数据进行 `join`
![[interval-join.svg]]
```java
 ds1.intervalJoin(ds2)  
	.between(Time lowerBound, Time upperBound)

// 必须传入KeyedStream，返回一个IntervalJoin对象
public <T1> IntervalJoin<T, T1, KEY> intervalJoin(KeyedStream<T1, KEY> otherStream) {  
    return new IntervalJoin<>(this, otherStream);  
}

// between检查上下界后，返回IntervalJoined对象
public IntervalJoined<T1, T2, KEY> between(Time lowerBound, Time upperBound) {  
    if (timeBehaviour != TimeBehaviour.EventTime) {  
        throw new UnsupportedTimeCharacteristicException(  
                "Time-bounded stream joins are only supported in event time");  
    }  
  
    checkNotNull(lowerBound, "A lower bound needs to be provided for a time-bounded join");  
    checkNotNull(upperBound, "An upper bound needs to be provided for a time-bounded join");  
  
    return new IntervalJoined<>(  
            streamOne,  
            streamTwo,  
            lowerBound.toMilliseconds(),  
            upperBound.toMilliseconds(),  
            true,  
            true);  
}

// process方法需要传入一个ProcessJoinFunction
public <OUT> SingleOutputStreamOperator<OUT> process(  
        ProcessJoinFunction<IN1, IN2, OUT> processJoinFunction) {}
```
# 5. 触发器
- 触发器用来确认窗口什么时候准备完毕，可以处理窗口数据
- 触发器的触发有三种方式
	- `onElement()`
	- `onEventTime()`
	- `onProcessingTime()`
- 触发器的返回结果
```java
public enum TriggerResult {  
	// 不做操作
    CONTINUE(false, false),  
	// 触发且清除窗口数据
    FIRE_AND_PURGE(true, true),  
    // 只触发
    FIRE(true, false),  
    // 只清除
	PURGE(false, true);
}
```
## 5.1 EventTimeTrigger
```java
public TriggerResult onElement(  
        Object element, long timestamp, TimeWindow window, TriggerContext ctx)  
        throws Exception {  
    if (window.maxTimestamp() <= ctx.getCurrentWatermark()) {
	    // 事件事件 > 窗口的最大范围，立即计算  
        // if the watermark is already past the window fire immediately  
        return TriggerResult.FIRE;  
    } else {
	    // 继续  
        ctx.registerEventTimeTimer(window.maxTimestamp());  
        return TriggerResult.CONTINUE;  
    }  
}

public TriggerResult onEventTime(long time, TimeWindow window, TriggerContext ctx) {  
	// 会话窗口下，window 的 maxTimestamp 可能会发生变化
    return time == window.maxTimestamp() ? TriggerResult.FIRE : TriggerResult.CONTINUE;  
}

// 和处理事件无关
public TriggerResult onProcessingTime(long time, TimeWindow window, TriggerContext ctx)  
        throws Exception {  
    return TriggerResult.CONTINUE;  
}
```
## 5.2 ProcessingTimeTrigger
```java
// 来一条数据注册一个定时器
public TriggerResult onElement(  
        Object element, long timestamp, TimeWindow window, TriggerContext ctx) {  
    ctx.registerProcessingTimeTimer(window.maxTimestamp());  
    return TriggerResult.CONTINUE;  
}

// 和事件时间无关
public TriggerResult onEventTime(long time, TimeWindow window, TriggerContext ctx)  
        throws Exception {  
    return TriggerResult.CONTINUE;  
}

// Timer 触发执行
public TriggerResult onProcessingTime(long time, TimeWindow window, TriggerContext ctx) {  
    return TriggerResult.FIRE;  
}
```
## 5.3 ContinuousEventTimeTrigger
- 基于水印根据给定的时间连续触发
- 业务中有时窗口的长度过长，希望每隔一段时间就计算一次
- SQL 中有累积窗口
```java
public class ContinuousEventTimeTrigger<W extends Window> extends Trigger<Object, W> {

	// 触发间隔
	private final long interval;
}

// ReducingStateDescriptor
private final ReducingStateDescriptor<Long> stateDesc =  
        new ReducingStateDescriptor<>("fire-time", new Min(), LongSerializer.INSTANCE);


```
# 6. 状态管理
- `Flink` 中的算子任务可以分为有状态和无状态两种
	- 无状态在计算时不依赖其他数据，直接输出转换结果
	- 有状态收到上游发送到的数据后，需要获取当前状态，进行更新后再向下流发送
- 状态可以分为 `RawState` 和 `ManagedState`
	- `RawState` 需要用户手动完成状态的备份，恢复
	- `ManagedState` 由 Flink 完成状态的备份和恢复
	- `ManagedState` 可以分为 `OperatorState` 和 `KeyedState`
## 6.1 OperatorState
- operator state 的 API 使用时需要实现 `CheckpointedFunction` 或 `ListCheckpointed` 接口
### 5.1.1 ListState
```java
// CheckpointedFunction接口中有两个方法: snapshotState和initializeState
public interface CheckpointedFunction {  
    // 周期性备份状态
	void snapshotState(FunctionSnapshotContext context) throws Exception;  
    // Task重启后进行初始化，为声明的状态去赋值和恢复
	void initializeState(FunctionInitializationContext context) throws Exception;  
}

// ListState的使用
public void initializeState(FunctionInitializationContext context) throws Exception {  
	// 从context中获取到OperatorStateStore
    OperatorStateStore stateStore = context.getOperatorStateStore();  
    ListStateDescriptor<String> stringListStateDescriptor = new ListStateDescriptor<>("list", String.class);  
    // 向getListState传入一个ListStateDescriptor获取状态中的值
    strs = stateStore.getListState(stringListStateDescriptor);  
}
```
### 6.1.2 ListUnionState
- ListState 恢复状态后，会将状态尽量均匀地分配到多个 Task，每个 Task 只获取状态的一部分。而 UnionListState 则是将状态的<font color='red'>全部</font>，分配给所有的 Task
```java
operatorStateStore.getUnionListState(strsListStateDescriptor); 
```
### 6.1.3 BroadcastState
- BroadcastState 用于在 `connect` 的两个流中广播配置信息
```java
// 将配置存在MapState中
MapStateDescriptor<String, MyConf> config = new MapStateDescriptor<>("config", String.class, MyConf.class);  
// 将配置流进行广播
BroadcastStream<MyConf> broadcastStream = ds2.broadcast(config);

ds1.connect(broadcastStream)  
        .process(new BroadcastProcessFunction<IN1, IN2, OUT>() {  
            // 处理数据流的数据
            public void processElement(OUT value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception {  
                // 获取广播状态  
                MyConf conf = ctx.getBroadcastState(config).get(value.getId());
                // 另一个流修改配置  
                value.setId(conf.getValue());  
                out.collect(value);  
            }  
  
            // 处理配置流数据
            public void processBroadcastElement(MyConf value, BroadcastProcessFunction<IN1, IN2, OUT>.Context ctx, Collector<OUT> out) throws Exception {  
	            // 将配置存入广播状态
                ctx.getBroadcastState(config).put(value.getId(), value);  
            }  
        }).print();
```
## 6.2 KeyedState
- `KeyedState` 针对 `KeyedStream` 的计算，每个算子的处理函数中，每个 Key 各有各自的状态，互不影响
- `ValueState`
	- T value(): 获取当前状态的值
	- update(T value): 对状态进行更新
- `ListState`
- `MapState`
- `ReducingState`
	- add(): 添加一个元素
	- get(): 获取聚合结果
- `AggregatingState`: 同 `ReducingState`
```java
.process(new KeyedProcessFunction<String, WaterSensor, String>() {  
	// 
    private ListState<Integer> vcs;  
  
    @Override  
    public void open(Configuration parameters) throws Exception {  
        ListStateDescriptor<Integer> state = new ListStateDescriptor<>("top3", Integer.class);  
        // 在open方法中，通过上下文获得状态
        vcs = getRuntimeContext().getListState(state);  
    }  
  
    @Override  
    public void processElement(WaterSensor value, KeyedProcessFunction<String, WaterSensor, String>.Context ctx, Collector<String> out) throws Exception {  
	    // 将新来的数据加入到状态中
        vcs.add(value.getVc());    
    }  
});
```
## 6.3 状态后端
- 状态后端主要负责<font color='red'>管理本地状态的存储方式和位置</font>
- 状态后端有两种
- `HashMapStateBackend`
	- 将状态存储在 TM 的堆内存中，基于内存读写，效率高
- `EmbeddedRocksDBStateBackend`
	- 将状态存储在内置的 `RocksDB` 中，`RocksDB` 随 `TM` 的启动而创建，将数据以 `byte[]`形式存储在缓存，缓存空间不足时将其溢写到磁盘中
	- 可以存储大状态
- 可以在 `flink-conf.yaml` 中配置：`state.backend.type: hashmap|rocksdb`
# 7. 容错机制
## 7.1 Checkpoint 机制
- `checkpoint` 基于 <font color='red'>Chandy-Lamport</font> 算法，通过分布式异步快照将状态备份到检查点
- `JobManager` 通过 `Barrier` 通知 `Task` 进行备份
- `Barrier` 将数据分隔为批次，只能由源头产生，当前`barrier`到达某个算子，意味着当前批次的数据已经被这个算子全部处理完毕，可以进行状态的持久化
![[stream_barriers.svg]]
- 在程序故障需要恢复快照时，总是以最近完成的 `checkpoint` 数据来恢复每一个算子的状态信息
### 7.1.1 checkpoint 的流程

![[checkpoint.svg]]
#### 7.1.1.1 JM 创建调度线程
```java
// 1.6开始调度任务时
// 将job状态转换至running
protected final void transitionToRunning() {  
    executionGraph.transitionToRunning();  
}

public void transitionToRunning() {  
    if (!transitionState(JobStatus.CREATED, JobStatus.RUNNING)) {  
        throw new IllegalStateException(  
                "Job may only be scheduled from state " + JobStatus.CREATED);  
    }  
}

public boolean transitionState(JobStatus current, JobStatus newState) {  
    return transitionState(current, newState, null);  
}

private boolean transitionState(JobStatus current, JobStatus newState, Throwable error) {  

    // now do the actual state transition  
    if (state == current) {  
        state = newState;   
        notifyJobStatusChange(newState);  
        notifyJobStatusHooks(newState, error);  
        return true;  
    } else {  
        return false;  
    }  
}

private void notifyJobStatusChange(JobStatus newState) {  
    if (jobStatusListeners.size() > 0) {  
        final long timestamp = System.currentTimeMillis();  
  
        for (JobStatusListener listener : jobStatusListeners) {  
            try {  
                listener.jobStatusChanges(getJobID(), newState, timestamp);  
        }  
    }  
}
// 监听到 Job 的状态变化，开始进行 CK
// CheckpointCoordinatorDeActivator.java
public void jobStatusChanges(JobID jobId, JobStatus newJobStatus, long timestamp) {  
    if (newJobStatus == JobStatus.RUNNING) {  
        // start the checkpoint scheduler  
        coordinator.startCheckpointScheduler();  
    } else {  
        // anything else should stop the trigger for now  
        coordinator.stopCheckpointScheduler();  
    }  
}

public void startCheckpointScheduler() {  
    synchronized (lock) {  
        periodicScheduling = true;  
        currentPeriodicTrigger = scheduleTriggerWithDelay(getRandomInitDelay());  
    }  
}

private ScheduledFuture<?> scheduleTriggerWithDelay(long initDelay) {  
    // 以固定间隔调度一次ScheduledTrigger
    return timer.scheduleAtFixedRate(  
            new ScheduledTrigger(), initDelay, baseInterval, TimeUnit.MILLISECONDS);  
}
```
#### 7.1.1.2 调度 Checkpoint
```java
// CheckpointCoordinator内部类
private final class ScheduledTrigger implements Runnable {  
  
    @Override  
    public void run() {  
        try {  
            triggerCheckpoint(checkpointProperties, null, true);  
        } catch (Exception e) {  
            LOG.error("Exception while triggering checkpoint for job {}.", job, e);  
        }  
    }  
}

CompletableFuture<CompletedCheckpoint> triggerCheckpoint(  
        CheckpointProperties props,  
        @Nullable String externalSavepointLocation,  
        boolean isPeriodic) {  
  
    CheckpointTriggerRequest request =  
            new CheckpointTriggerRequest(props, externalSavepointLocation, isPeriodic);  
    chooseRequestToExecute(request).ifPresent(this::startTriggeringCheckpoint);  
    return request.onCompletionPromise;  
}

private void startTriggeringCheckpoint(CheckpointTriggerRequest request) {   
	// 原子自增
    checkpointIdCounter.getAndIncrement();
    // 初始化ck存储状态的位置
    initializeCheckpointLocation(...);  
    // 触发ck初始化ck的id
    OperatorCoordinatorCheckpoints                                             .triggerAndAcknowledgeAllCoordinatorCheckpointsWithCompletion(...)
    // 初始化ck的id后，开始依次调用TM上每个Execution的Checkpoint
	triggerCheckpointRequest(request, timestamp, checkpoint)
}
```
##### 7.1.1.2.1 triggerAndAcknowledgeAllCoordinatorCheckpointsWithCompletion()
```java
public static CompletableFuture<Void>  
        triggerAndAcknowledgeAllCoordinatorCheckpointsWithCompletion(  
                final Collection<OperatorCoordinatorCheckpointContext> coordinators,  
                final PendingCheckpoint checkpoint,  
                final Executor acknowledgeExecutor)  
                throws CompletionException {  
  
    try {  
        return triggerAndAcknowledgeAllCoordinatorCheckpoints(  
                coordinators, checkpoint, acknowledgeExecutor);  
    }
}

public static CompletableFuture<Void> triggerAndAcknowledgeAllCoordinatorCheckpoints(  
        final Collection<OperatorCoordinatorCheckpointContext> coordinators,  
        final PendingCheckpoint checkpoint,  
        final Executor acknowledgeExecutor)  
        throws Exception {  
  
    final CompletableFuture<AllCoordinatorSnapshots> snapshots =  
            triggerAllCoordinatorCheckpoints(coordinators, checkpoint.getCheckpointID());  
  
    return snapshots.thenAcceptAsync(  
            (allSnapshots) -> {  
                try {  
                    acknowledgeAllCoordinators(checkpoint, allSnapshots.snapshots);  
                } 
            },  
            acknowledgeExecutor);  
}

public static CompletableFuture<AllCoordinatorSnapshots> triggerAllCoordinatorCheckpoints(  
        final Collection<OperatorCoordinatorCheckpointContext> coordinators,  
        final long checkpointId)  
        throws Exception {  
    // 每个coordinator对应一个SourcejobVertex
    final Collection<CompletableFuture<CoordinatorSnapshot>> individualSnapshots =  
            new ArrayList<>(coordinators.size());  
  
    for (final OperatorCoordinatorCheckpointContext coordinator : coordinators) {  
        final CompletableFuture<CoordinatorSnapshot> checkpointFuture =  
                // JM根据SourceOperator个数依次持久化checkpointId
                triggerCoordinatorCheckpoint(coordinator, checkpointId);  
        individualSnapshots.add(checkpointFuture);  
    }  
}

public static CompletableFuture<CoordinatorSnapshot> triggerCoordinatorCheckpoint(  
        final OperatorCoordinatorCheckpointContext coordinatorContext, final long checkpointId)  
        throws Exception {  
  
    final CompletableFuture<byte[]> checkpointFuture = new CompletableFuture<>();  
    // checkpointId写入完成后封装成CoordinatorSnapshot返回
    coordinatorContext.checkpointCoordinator(checkpointId, checkpointFuture);  
  
    return checkpointFuture.thenApply(  
            (state) ->  
                    new CoordinatorSnapshot(  
                            coordinatorContext,  
                            new ByteStreamStateHandle(  
                                    coordinatorContext.operatorId().toString(), state)));  
}

public void checkpointCoordinator(long checkpointId, CompletableFuture<byte[]> result) {  
	mainThreadExecutor.execute(() -> checkpointCoordinatorInternal(checkpointId, result));  
}

private void checkpointCoordinatorInternal(  
        final long checkpointId, final CompletableFuture<byte[]> result) {  
    try {  
	    // coordinator的不同实现类将checkpointId持久化到state中
        coordinator.checkpointCoordinator(checkpointId, coordinatorCheckpoint);  
    }
}
```
##### 7.1.1.2.2 triggerCheckpointRequest()
```java
private void triggerCheckpointRequest(  
        CheckpointTriggerRequest request, long timestamp, PendingCheckpoint checkpoint) {  
    if (checkpoint.isDisposed()) {
    } else {  
        triggerTasks(request, timestamp, checkpoint)  
}

private CompletableFuture<Void> triggerTasks(  
        CheckpointTriggerRequest request, long timestamp, PendingCheckpoint checkpoint) {  
    // no exception, no discarding, everything is OK  
    final long checkpointId = checkpoint.getCheckpointID();  
  
    // send messages to the tasks to trigger their checkpoints  
    List<CompletableFuture<Acknowledge>> acks = new ArrayList<>();  
    for (Execution execution : checkpoint.getCheckpointPlan().getTasksToTrigger()) {  
	    // 都会调用triggerCheckpointHelper()
        if (request.props.isSynchronous()) {  
            acks.add(  
                    execution.triggerSynchronousSavepoint(  
                            checkpointId, timestamp, checkpointOptions));  
        } else {  
            acks.add(execution.triggerCheckpoint(checkpointId, timestamp, checkpointOptions));  
        }  
    }  
    return FutureUtils.waitForAll(acks);  
}

private CompletableFuture<Acknowledge> triggerCheckpointHelper(  
        long checkpointId, long timestamp, CheckpointOptions checkpointOptions) {  
  
        return taskManagerGateway.triggerCheckpoint(  
                attemptId, getVertex().getJobId(), checkpointId, timestamp, checkpointOptions);  
    }  
    return CompletableFuture.completedFuture(Acknowledge.get());  
}

public CompletableFuture<Acknowledge> triggerCheckpoint(  
        ExecutionAttemptID executionAttemptID,  
        long checkpointId,  
        long checkpointTimestamp,  
        CheckpointOptions checkpointOptions) {  
  
    final Task task = taskSlotTable.getTask(executionAttemptID);  
  
    if (task != null) {  
        task.triggerCheckpointBarrier(checkpointId, checkpointTimestamp, checkpointOptions);  
        return CompletableFuture.completedFuture(Acknowledge.get());  
    }
}

public void triggerCheckpointBarrier(  
        final long checkpointID,  
        final long checkpointTimestamp,  
        final CheckpointOptions checkpointOptions) {  
  
    final TaskInvokable invokable = this.invokable;  
    final CheckpointMetaData checkpointMetaData =  
            new CheckpointMetaData(  
                    checkpointID, checkpointTimestamp, System.currentTimeMillis());  
  
    if (executionState == ExecutionState.RUNNING) {  
        checkState(invokable instanceof CheckpointableTask, "invokable is not checkpointable");  
        try {  
            ((CheckpointableTask) invokable)  
                    .triggerCheckpointAsync(checkpointMetaData, checkpointOptions)  
            }
}
```
#### 7.1.1.3 算子产生 Barrier
```java
/*
 * StreamTask 是 TM 部署并执行的本地处理单元
 * 每个 StreamTask 运行一个或多个 chain 在一起的 StreamOperator
 * StreamTask
 *     -> AbstractTwoInputStreamTask
 *            -> TwoInputStreamTask
 *     -> MultipleInputStreamTask
 *     -> OneInputStreamTask
 *     -> SourceOperatorStreamTask
 *     -> SourceStreamTask
 * 最终会调用父类 StreamTask 的 triggerCheckpointAsync() 方法
*/
public CompletableFuture<Boolean> triggerCheckpointAsync(  
        CheckpointMetaData checkpointMetaData, CheckpointOptions checkpointOptions) {  
    triggerCheckpointAsyncInMailbox(checkpointMetaData, checkpointOptions));  
    return result;  
}

private boolean triggerCheckpointAsyncInMailbox(  
        CheckpointMetaData checkpointMetaData, CheckpointOptions checkpointOptions)  
        throws Exception {
    boolean success =  
        performCheckpoint(checkpointMetaData, checkpointOptions, checkpointMetrics);  
	return success;
}

private boolean performCheckpoint(  
        CheckpointMetaData checkpointMetaData,  
        CheckpointOptions checkpointOptions,  
        CheckpointMetricsBuilder checkpointMetrics)  
        throws Exception {
        
	subtaskCheckpointCoordinator.checkpointState(  
        checkpointMetaData,  
        checkpointOptions,  
        checkpointMetrics,  
        operatorChain,  
        finishedOperators,  
        this::isRunning);
}

// SubtaskCheckpointCoordinatorImpl
public void checkpointState(  
        CheckpointMetaData metadata,  
        CheckpointOptions options,  
        CheckpointMetricsBuilder metrics,  
        OperatorChain<?, ?> operatorChain,  
        boolean isTaskFinished,  
        Supplier<Boolean> isRunning)  
        throws Exception {  

	// AlignmentType = {AT_LEAST_ONCE, ALIGNED, UNALIGNED, FORCED_ALIGNED}
    if (options.getAlignment() == CheckpointOptions.AlignmentType.FORCED_ALIGNED) {  
        options = options.withUnalignedSupported();  
        initInputsCheckpoint(metadata.getCheckpointId(), options);  
    }  
  
    // Step (1): Prepare the checkpoint, allow operators to do some pre-barrier work.  
    // The pre-barrier work should be nothing or minimal in the common case.  
    operatorChain.prepareSnapshotPreBarrier(metadata.getCheckpointId());  
  
    // Step (2): Send the checkpoint barrier downstream  
	// 创建Barrier
    CheckpointBarrier checkpointBarrier =  
            new CheckpointBarrier(metadata.getCheckpointId(), metadata.getTimestamp(), options);  
    // 发送Barrier
    operatorChain.broadcastEvent(checkpointBarrier, options.isUnalignedCheckpoint());  
  
    // Step (3): Register alignment timer to timeout aligned barrier to unaligned barrier  
    registerAlignmentTimer(metadata.getCheckpointId(), operatorChain, checkpointBarrier);  
  
    // Step (4): Prepare to spill the in-flight buffers for input and output  
    // isUnalignedCheckpoint()
    if (options.needsChannelState()) {  
        // output data already written while broadcasting event  
        channelStateWriter.finishOutput(metadata.getCheckpointId());  
    }  
  
    // Step (5): Take the state snapshot. This should be largely asynchronous, to not impact  
    // progress of the    // streaming topology  
    Map<OperatorID, OperatorSnapshotFutures> snapshotFutures =  
            new HashMap<>(operatorChain.getNumberOfOperators());  
    try {  
        if (takeSnapshotSync(  
                snapshotFutures, metadata, metrics, options, operatorChain, isRunning)) {
            // 所有节点ck完成后回复Master
            finishAndReportAsync(  
                    snapshotFutures,  
                    metadata,  
                    metrics,  
                    operatorChain.isTaskDeployedAsFinished(),  
                    isTaskFinished,  
                    isRunning);  
        }
}
```
#### 7.1.1.4 向下游广播 Barrier
```java
public void broadcastEvent(AbstractEvent event, boolean isPriorityEvent) throws IOException {  
    for (RecordWriterOutput<?> streamOutput : streamOutputs) {  
        streamOutput.broadcastEvent(event, isPriorityEvent);  
    }  
}

public void broadcastEvent(AbstractEvent event, boolean isPriorityEvent) throws IOException {  
    if (isPriorityEvent  
            && event instanceof CheckpointBarrier  
            && !supportsUnalignedCheckpoints) {  
        final CheckpointBarrier barrier = (CheckpointBarrier) event;  
        event = barrier.withOptions(barrier.getCheckpointOptions().withUnalignedUnsupported());  
        isPriorityEvent = false;  
    }  
    recordWriter.broadcastEvent(event, isPriorityEvent);  
}

public void broadcastEvent(AbstractEvent event, boolean isPriorityEvent) throws IOException {  
    targetPartition.broadcastEvent(event, isPriorityEvent);  
}

// 通过BufferWritingResultPartition写出
public void broadcastEvent(AbstractEvent event, boolean isPriorityEvent) throws IOException {  
    checkInProduceState();  
    finishBroadcastBufferBuilder();  
    finishUnicastBufferBuilders();  
	// 封装成带优先级的buffer并add到（Pipelined/BoundedBlocking）ResultSubpartition的PrioritizedDeque队列中
    try (BufferConsumer eventBufferConsumer =  
            EventSerializer.toBufferConsumer(event, isPriorityEvent)) {  
        totalWrittenBytes += ((long) eventBufferConsumer.getWrittenBytes() * numSubpartitions);  
        for (ResultSubpartition subpartition : subpartitions) {  
            // addBuffer添加到优先级队列中
            subpartition.add(eventBufferConsumer.copy(), 0);  
        }  
    }  
}

// PipelinedSubpartition.java
public int add(BufferConsumer bufferConsumer, int partialRecordLength) {  
    return add(bufferConsumer, partialRecordLength, false);  
}

private int add(BufferConsumer bufferConsumer, int partialRecordLength, boolean finish) {  
  
    final boolean notifyDataAvailable;  
    int prioritySequenceNumber = DEFAULT_PRIORITY_SEQUENCE_NUMBER;  
    int newBufferSize;  
    synchronized (buffers) {  
  
        // Add the bufferConsumer and update the stats 
        // 增加buffer到PrioritizedDeque里，优先级高的buffer放入队列头 
        if (addBuffer(bufferConsumer, partialRecordLength)) {  
            prioritySequenceNumber = sequenceNumber;  
        }  
    }  
  
    notifyPriorityEvent(prioritySequenceNumber);  
    if (notifyDataAvailable) {  
	    // 通知下游消费
        notifyDataAvailable();  
    }  
  
    return newBufferSize;  
}

private void notifyDataAvailable() {  
    final PipelinedSubpartitionView readView = this.readView;  
    if (readView != null) {  
        readView.notifyDataAvailable();  
    }  
}

public void notifyDataAvailable() {  
    availabilityListener.notifyDataAvailable();  
}

// CreditBasedSequenceNumberingViewReader.java
// 向下游监听的availabilityListener发送事件消息
public void notifyDataAvailable() {  
    requestQueue.notifyReaderNonEmpty(this);  
}
```
#### 7.1.1.5 发送 Barrier 后执行 ck
```java
// 6.1.1.3 step5
private boolean takeSnapshotSync(  
        Map<OperatorID, OperatorSnapshotFutures> operatorSnapshotsInProgress,  
        CheckpointMetaData checkpointMetaData,  
        CheckpointMetricsBuilder checkpointMetrics,  
        CheckpointOptions checkpointOptions,  
        OperatorChain<?, ?> operatorChain,  
        Supplier<Boolean> isRunning)  
        throws Exception {
        
	CheckpointStreamFactory storage =  
        checkpointStorage.resolveCheckpointStorageLocation(  
                checkpointId, checkpointOptions.getTargetLocation());  
  
try {  
	// OperatorChain有两个子类：FinishedOperatorChain和RegularOperatorChain
	// 最终都会调用sendAcknowledgeCheckpointEvent()
    operatorChain.snapshotState(  
            operatorSnapshotsInProgress,  
            checkpointMetaData,  
            checkpointOptions,  
            isRunning,  
            channelStateWriteResult,  
            storage);
}

public void snapshotState(  
        Map<OperatorID, OperatorSnapshotFutures> operatorSnapshotsInProgress,  
        CheckpointMetaData checkpointMetaData,  
        CheckpointOptions checkpointOptions,  
        Supplier<Boolean> isRunning,  
        ChannelStateWriter.ChannelStateWriteResult channelStateWriteResult,  
        CheckpointStreamFactory storage)  
        throws Exception {  
    for (StreamOperatorWrapper<?, ?> operatorWrapper : getAllOperators(true)) {  
        if (!operatorWrapper.isClosed()) {  
            operatorSnapshotsInProgress.put(  
                    operatorWrapper.getStreamOperator().getOperatorID(), 
                    // 备份 
                    buildOperatorSnapshotFutures(  
                            checkpointMetaData,  
                            checkpointOptions,  
                            operatorWrapper.getStreamOperator(),  
                            isRunning,  
                            channelStateWriteResult,  
                            storage));  
        }  
    } 
    // 备份完成发送ack 
    sendAcknowledgeCheckpointEvent(checkpointMetaData.getCheckpointId()); 
}

private OperatorSnapshotFutures buildOperatorSnapshotFutures(  
        CheckpointMetaData checkpointMetaData,  
        CheckpointOptions checkpointOptions,  
        StreamOperator<?> op,  
        Supplier<Boolean> isRunning,  
        ChannelStateWriter.ChannelStateWriteResult channelStateWriteResult,  
        CheckpointStreamFactory storage)  
        throws Exception {  
    OperatorSnapshotFutures snapshotInProgress =  
            checkpointStreamOperator(  
                    op, checkpointMetaData, checkpointOptions, storage, isRunning);  
    snapshotChannelStates(op, channelStateWriteResult, snapshotInProgress);  
  
    return snapshotInProgress;  
}

private static OperatorSnapshotFutures checkpointStreamOperator(  
        StreamOperator<?> op,  
        CheckpointMetaData checkpointMetaData,  
        CheckpointOptions checkpointOptions,  
        CheckpointStreamFactory storageLocation,  
        Supplier<Boolean> isRunning)  
        throws Exception {  
    try {  
	    // 持久化state
        return op.snapshotState(  
                checkpointMetaData.getCheckpointId(),  
                checkpointMetaData.getTimestamp(),  
                checkpointOptions,  
                storageLocation);  
    } catch (Exception ex) {  
        if (isRunning.get()) {  
            LOG.info(ex.getMessage(), ex);  
        }  
        throw ex;  
    }  
}

// AbstractStreamOperatorV2.java
public final OperatorSnapshotFutures snapshotState(  
        long checkpointId,  
        long timestamp,  
        CheckpointOptions checkpointOptions,  
        CheckpointStreamFactory factory)  
        throws Exception {  
    return stateHandler.snapshotState(  
            this,  
            Optional.ofNullable(timeServiceManager),  
            getOperatorName(),  
            checkpointId,  
            timestamp,  
            checkpointOptions,  
            factory,  
            isUsingCustomRawKeyedState());  
}

public OperatorSnapshotFutures snapshotState(  
        CheckpointedStreamOperator streamOperator,  
        Optional<InternalTimeServiceManager<?>> timeServiceManager,  
        String operatorName,  
        long checkpointId,  
        long timestamp,  
        CheckpointOptions checkpointOptions,  
        CheckpointStreamFactory factory,  
        boolean isUsingCustomRawKeyedState)  
        throws CheckpointException {  
    KeyGroupRange keyGroupRange =  
            null != keyedStateBackend  
                    ? keyedStateBackend.getKeyGroupRange()  
                    : KeyGroupRange.EMPTY_KEY_GROUP_RANGE;  
  
    OperatorSnapshotFutures snapshotInProgress = new OperatorSnapshotFutures();  
  
    StateSnapshotContextSynchronousImpl snapshotContext =  
            new StateSnapshotContextSynchronousImpl(  
                    checkpointId, timestamp, factory, keyGroupRange, closeableRegistry);  
  
    snapshotState(  
            streamOperator,  
            timeServiceManager,  
            operatorName,  
            checkpointId,  
            timestamp,  
            checkpointOptions,  
            factory,  
            snapshotInProgress,  
            snapshotContext,  
            isUsingCustomRawKeyedState);  
  
    return snapshotInProgress;  
}

void snapshotState(  
        CheckpointedStreamOperator streamOperator,  
        Optional<InternalTimeServiceManager<?>> timeServiceManager,  
        String operatorName,  
        long checkpointId,  
        long timestamp,  
        CheckpointOptions checkpointOptions,  
        CheckpointStreamFactory factory,  
        OperatorSnapshotFutures snapshotInProgress,  
        StateSnapshotContextSynchronousImpl snapshotContext,  
        boolean isUsingCustomRawKeyedState)  
        throws CheckpointException {  
    try {  
        // 执行各种算子的备份state操作
        streamOperator.snapshotState(snapshotContext);  

        if (null != keyedStateBackend) {  
            if (isCanonicalSavepoint(checkpointOptions.getCheckpointType())) {  
                SnapshotStrategyRunner<KeyedStateHandle, ? extends FullSnapshotResources<?>>  
                        snapshotRunner =  
                                prepareCanonicalSavepoint(keyedStateBackend, closeableRegistry);  
  
                snapshotInProgress.setKeyedStateManagedFuture(  
                        snapshotRunner.snapshot(  
                                checkpointId, timestamp, factory, checkpointOptions));  
  
            } else {  
                snapshotInProgress.setKeyedStateManagedFuture(  
                        keyedStateBackend.snapshot(  
                                checkpointId, timestamp, factory, checkpointOptions));  
            }  
        }  
    }  
}
```
#### 7.1.1.6 向 coordinator 发送 ack
```java
// Flink 最终调用 finishAndReportAsync()向 Master 发送报告，所有的节点都报告结束时，Master 会生成 CompletedCheckpoint 持久化到状态后端中，结束 ck 流程
private void finishAndReportAsync(  
        Map<OperatorID, OperatorSnapshotFutures> snapshotFutures,  
        CheckpointMetaData metadata,  
        CheckpointMetricsBuilder metrics,  
        boolean isTaskDeployedAsFinished,  
        boolean isTaskFinished,  
        Supplier<Boolean> isRunning)  
        throws IOException {  
    AsyncCheckpointRunnable asyncCheckpointRunnable =  
            new AsyncCheckpointRunnable(...);  
}

// run方法
public void run() {  
        if (asyncCheckpointState.compareAndSet(  
                AsyncCheckpointState.RUNNING, AsyncCheckpointState.COMPLETED)) {  
            // 报告
            reportCompletedSnapshotStates(  
                    snapshotsFinalizeResult.jobManagerTaskOperatorSubtaskStates,  
                    snapshotsFinalizeResult.localTaskOperatorSubtaskStates,  
                    asyncDurationMillis);  
        } 
}

private void reportCompletedSnapshotStates(  
        TaskStateSnapshot acknowledgedTaskStateSnapshot,  
        TaskStateSnapshot localTaskStateSnapshot,  
        long asyncDurationMillis) {
	taskEnvironment  
        .getTaskStateManager()  
        // TaskStateManager是专门用来提供报告和检索任务状态的接口
        .reportTaskStateSnapshots(...)
}

public void reportTaskStateSnapshots(  
        @Nonnull CheckpointMetaData checkpointMetaData,  
        @Nonnull CheckpointMetrics checkpointMetrics,  
        @Nullable TaskStateSnapshot acknowledgedState,  
        @Nullable TaskStateSnapshot localState) {  

    checkpointResponder.acknowledgeCheckpoint(  
            jobId, executionAttemptID, checkpointId, checkpointMetrics, acknowledgedState);  
}

public void acknowledgeCheckpoint(  
        JobID jobID,  
        ExecutionAttemptID executionAttemptID,  
        long checkpointId,  
        CheckpointMetrics checkpointMetrics,  
        TaskStateSnapshot subtaskState) {  
    checkpointCoordinatorGateway.acknowledgeCheckpoint(...);  
}

public void acknowledgeCheckpoint(  
        final JobID jobID,  
        final ExecutionAttemptID executionAttemptID,  
        final long checkpointId,  
        final CheckpointMetrics checkpointMetrics,  
        @Nullable final SerializedValue<TaskStateSnapshot> checkpointState) {  
    schedulerNG.acknowledgeCheckpoint(...);  
}

// SchedulerBase.java
public void acknowledgeCheckpoint(  
        final JobID jobID,  
        final ExecutionAttemptID executionAttemptID,  
        final long checkpointId,  
        final CheckpointMetrics checkpointMetrics,  
        final TaskStateSnapshot checkpointState) {  
  
    executionGraphHandler.acknowledgeCheckpoint(  
            jobID, executionAttemptID, checkpointId, checkpointMetrics, checkpointState);  
}

public void acknowledgeCheckpoint(  
        final JobID jobID,  
        final ExecutionAttemptID executionAttemptID,  
        final long checkpointId,  
        final CheckpointMetrics checkpointMetrics,  
        final TaskStateSnapshot checkpointState) {  
    processCheckpointCoordinatorMessage(  
            "AcknowledgeCheckpoint",  
            coordinator ->  
			        // 将ack消息包装成AcknowledgeCheckpoint
			        // 调度器包装后的ack，会被传递给CheckpointCoordinator组件继续处理
                    coordinator.receiveAcknowledgeMessage(  
                            new AcknowledgeCheckpoint(...);  
}

public boolean receiveAcknowledgeMessage(  
        AcknowledgeCheckpoint message, String taskManagerLocationInfo)  
        throws CheckpointException {  
  
    final long checkpointId = message.getCheckpointId();  
  
    synchronized (lock) {  
  // 根据checkpointId，从映射关系为“checkpointId:PendingCheckpoint”的Map集合中取出对应的PendingCheckpoint
        final PendingCheckpoint checkpoint = pendingCheckpoints.get(checkpointId);  
  
  
        if (checkpoint != null && !checkpoint.isDisposed()) {  
  
            switch (checkpoint.acknowledgeTask(  
                    message.getTaskExecutionId(),  
                    message.getSubtaskState(),  
                    message.getCheckpointMetrics())) {  
                case SUCCESS:  

					// 收到了全部的ack
                    if (checkpoint.isFullyAcknowledged()) {  
                        completePendingCheckpoint(checkpoint);  
                    }  
                    break;  
}
```
#### 7.1.1.7 生成最终的 CompletedCheckpoint
```java
// CheckpointCoordinator.java
private void completePendingCheckpoint(PendingCheckpoint pendingCheckpoint)  
        throws CheckpointException {  
    try {  
        completedCheckpoint = finalizeCheckpoint(pendingCheckpoint);  
    }  
    // 通知算子ck已完成
    cleanupAfterCompletedCheckpoint(  
            pendingCheckpoint, checkpointId, completedCheckpoint, lastSubsumed, props);
}

private CompletedCheckpoint finalizeCheckpoint(PendingCheckpoint pendingCheckpoint)  
        throws CheckpointException {  
    try {  
        final CompletedCheckpoint completedCheckpoint =  
			    // 创建CompletedCheckpoint对象
                pendingCheckpoint.finalizeCheckpoint(  
                        checkpointsCleaner, this::scheduleTriggerRequest, executor);  
  
    return completedCheckpoint;  
    }
}

private void cleanupAfterCompletedCheckpoint(  
        PendingCheckpoint pendingCheckpoint,  
        long checkpointId,  
        CompletedCheckpoint completedCheckpoint,  
        CompletedCheckpoint lastSubsumed,  
        CheckpointProperties props) {
    sendAcknowledgeMessages(  
        pendingCheckpoint.getCheckpointPlan().getTasksToCommitTo(),  
        checkpointId,  
        completedCheckpoint.getTimestamp(),  
        extractIdIfDiscardedOnSubsumed(lastSubsumed));
}

void sendAcknowledgeMessages(  
        List<ExecutionVertex> tasksToCommit,  
        long completedCheckpointId,  
        long completedTimestamp,  
        long lastSubsumedCheckpointId) {  
    // commit tasks  
    for (ExecutionVertex ev : tasksToCommit) {  
        Execution ee = ev.getCurrentExecutionAttempt();  
        if (ee != null) {  
	        /**
			 * 通过Execution向TaskManagerGateway发送CheckpointComplete消息，通知所有的Task实例：本次Checkpoint操作结束
			 * TaskExecutor收到CheckpointComplete消息后，会从TaskSlotTable中取出对应的Task实例，并向其发送CheckpointComplete消息
			 * 所有实现了CheckpointListener监听器的观察者，在收到Checkpoint完成的消息后，都会进行各自的处理
			 */
            ee.notifyCheckpointOnComplete(  
                    completedCheckpointId, completedTimestamp, lastSubsumedCheckpointId);  
        }  
    }  
}

public void notifyCheckpointOnComplete(  
        long completedCheckpointId, long completedTimestamp, long lastSubsumedCheckpointId) {  
    final LogicalSlot slot = assignedResource;  
  
    if (slot != null) {  
        final TaskManagerGateway taskManagerGateway = slot.getTaskManagerGateway();  
        taskManagerGateway.notifyCheckpointOnComplete(  
                attemptId,  
                getVertex().getJobId(),  
                completedCheckpointId,  
                completedTimestamp,  
                lastSubsumedCheckpointId);  
    }
}

public void notifyCheckpointOnComplete(  
        ExecutionAttemptID executionAttemptID,  
        JobID jobId,  
        long completedCheckpointId,  
        long completedTimestamp,  
        long lastSubsumedCheckpointId) {  
    taskExecutorGateway.confirmCheckpoint(  
            executionAttemptID,  
            completedCheckpointId,  
            completedTimestamp,  
            lastSubsumedCheckpointId);  
}

public CompletableFuture<Acknowledge> confirmCheckpoint(  
        ExecutionAttemptID executionAttemptID,  
        long completedCheckpointId,  
        long completedCheckpointTimestamp,  
        long lastSubsumedCheckpointId) {  );  
  
    final Task task = taskSlotTable.getTask(executionAttemptID);  
  
    if (task != null) {  
	    // task收到消息
        task.notifyCheckpointComplete(completedCheckpointId);  
  
        task.notifyCheckpointSubsumed(lastSubsumedCheckpointId);  
        return CompletableFuture.completedFuture(Acknowledge.get());  
    }
}

/*
 * Task实例将其通知给StreamTask中的算子
 * StreamTask负责通知给OperatorChain中的各个StreamOperator
 * StreamOperator收到Checkpoint完成的消息后，会根据自身需要进行后续处理
 * 如FlinkKafkaConsumerBase的notifyCheckpointComplete()方法，会在ck成功后，将offset写入Kafka
 */
```
### 7.1.2 非 Source 算子对 ck 的处理
- 上游产生的数据通过 `ResultPartition` 的形式，通过 `InputGate` 中的 `InputChannel` 发往下一个 `Task
- `CheckpointedInputGate` 通过 `CheckpointBarrierHandler` 处理 `InputGate` 中的 `Barrier`
#### 7.1.2.1 下游接收数据
```java
// StreamTask.java
protected void processInput(MailboxDefaultAction.Controller controller) throws Exception {  
    DataInputStatus status = inputProcessor.processInput();  
    switch (status) {  
        case MORE_AVAILABLE:  
            if (taskIsAvailable()) {  
                return;  
            }  
            break;
}

public DataInputStatus processInput() throws Exception {  
    DataInputStatus status = input.emitNext(output);
    return status;  
}

public DataInputStatus emitNext(DataOutput<T> output) throws Exception {  
    while (true) {  
		// 从 checkpointedInputGate 中取出一个数据
        Optional<BufferOrEvent> bufferOrEvent = checkpointedInputGate.pollNext();  
    }  
}

public Optional<BufferOrEvent> pollNext() throws IOException, InterruptedException {  
    Optional<BufferOrEvent> next = inputGate.pollNext();  
  
    BufferOrEvent bufferOrEvent = next.get();  
  
    if (bufferOrEvent.isEvent()) {  
         // 判断是否为 Barrier
        return handleEvent(bufferOrEvent);
     }
 }

private Optional<BufferOrEvent> handleEvent(BufferOrEvent bufferOrEvent) throws IOException {  
    Class<? extends AbstractEvent> eventClass = bufferOrEvent.getEvent().getClass();  
    if (eventClass == CheckpointBarrier.class) {  
        CheckpointBarrier checkpointBarrier = (CheckpointBarrier) bufferOrEvent.getEvent();  
        // CheckpointBarrierHandler 处理 Barrier
        barrierHandler.processBarrier(checkpointBarrier, bufferOrEvent.getChannelInfo(), false);
}
```
#### 7.1.2.2 根据是否精准一次生成不同的 Handler
```java
public static CheckpointBarrierHandler createCheckpointBarrierHandler(  
        CheckpointableTask toNotifyOnCheckpoint,  
        StreamConfig config,  
        SubtaskCheckpointCoordinator checkpointCoordinator,  
        String taskName,  
        List<IndexedInputGate>[] inputGates,  
        List<StreamTaskSourceInput<?>> sourceInputs,  
        MailboxExecutor mailboxExecutor,  
        TimerService timerService) {  

    switch (config.getCheckpointMode()) {  
        case EXACTLY_ONCE:  
            return createBarrierHandler(...);  
        case AT_LEAST_ONCE:  
            return new CheckpointBarrierTracker(...);  
    }  
}

// 精准一次有对齐和非对齐
private static SingleCheckpointBarrierHandler createBarrierHandler(  
        CheckpointableTask toNotifyOnCheckpoint,  
        StreamConfig config,  
        SubtaskCheckpointCoordinator checkpointCoordinator,  
        String taskName,  
        MailboxExecutor mailboxExecutor,  
        TimerService timerService,  
        CheckpointableInput[] inputs,  
        Clock clock,  
        int numberOfChannels) {  
 
    if (config.isUnalignedCheckpointsEnabled()) {  
        return SingleCheckpointBarrierHandler.alternating(  
                taskName,  
                toNotifyOnCheckpoint,  
                checkpointCoordinator,  
                clock,  
                numberOfChannels, BarrierAlignmentUtil.createRegisterTimerCallback(mailboxExecutor, timerService), nableCheckpointAfterTasksFinished, inputs);  
    } else {  
        return SingleCheckpointBarrierHandler.aligned(  
                taskName,  
                toNotifyOnCheckpoint,  
                clock,  
                numberOfChannels,  
BarrierAlignmentUtil.createRegisterTimerCallback(mailboxExecutor, timerService), enableCheckpointAfterTasksFinished, inputs);  
    }  
}
```
#### 7.1.2.3 对齐精准一次
```java
public static SingleCheckpointBarrierHandler aligned(  
        String taskName,  
        CheckpointableTask toNotifyOnCheckpoint,  
        Clock clock,  
        int numOpenChannels,  
        DelayableTimer registerTimer,  
        boolean enableCheckpointAfterTasksFinished,  
        CheckpointableInput... inputs) {  
    return new SingleCheckpointBarrierHandler(  
            taskName,  
            toNotifyOnCheckpoint,  
            null,  
            clock,  
            numOpenChannels,  
            new WaitingForFirstBarrier(inputs),  
            false,  
            registerTimer,  
            inputs,  
            enableCheckpointAfterTasksFinished);  
}

public void processBarrier(  
        CheckpointBarrier barrier, InputChannelInfo channelInfo, boolean isRpcTriggered)  
        throws IOException {  
    long barrierId = barrier.getId();  
    // 检查是否为新的 ck
    checkNewCheckpoint(barrier);  
  
    markCheckpointAlignedAndTransformState(  
            channelInfo,  
            barrier,  
            state -> state.barrierReceived(context, channelInfo, barrier, !isRpcTriggered));  
}

// AbstractAlignedBarrierHandlerState
public final BarrierHandlerState barrierReceived(  
        Controller controller,  
        InputChannelInfo channelInfo,  
        CheckpointBarrier checkpointBarrier,  
        boolean markChannelBlocked)  
        throws IOException, CheckpointException {  
    
    if (markChannelBlocked) {  
        state.blockChannel(channelInfo);  
    }  
    // Barrier 对齐后触发 ck
    if (controller.allBarriersReceived()) {  
	    // 通知 StreamTask 执行 ck
        return triggerGlobalCheckpoint(controller, checkpointBarrier);  
    }  
  
    return convertAfterBarrierReceived(state);  
}

public boolean allBarriersReceived() {  
    return alignedChannels.size() == targetChannelCount;  
}
```
#### 7.1.2.4 对齐至少一次
```java
// CheckpointBarrierTracker
public void processBarrier(  
        CheckpointBarrier receivedBarrier, InputChannelInfo channelInfo, boolean isRpcTriggered)  
        throws IOException {  
    final long barrierId = receivedBarrier.getId();  
  
    // 只有一个 channel 立即 ck    
    if (receivedBarrier.getId() > latestPendingCheckpointID && numOpenChannels == 1) {  
        markAlignmentStartAndEnd(barrierId, receivedBarrier.getTimestamp()); 
        notifyCheckpoint(receivedBarrier);  
        return;  
    }  

    if (barrierCount != null) {  
        // add one to the count to that barrier and check for completion  
        int numChannelsNew = barrierCount.markChannelAligned(channelInfo);  
        if (numChannelsNew == barrierCount.getTargetChannelCount()) {    
		    // 已对齐
            // notify the listener  
            if (!barrierCount.isAborted()) {  
                triggerCheckpointOnAligned(barrierCount);  
            }  
        }  
    }
}
```
#### 7.1.2.5 非对齐
```java
// 入口同 7.1.2.3，barrierReceived 通过 
// AlternatingWaitingForFirstBarrierUnaligned 实现
public BarrierHandlerState barrierReceived(  
        Controller controller,  
        InputChannelInfo channelInfo,  
        CheckpointBarrier checkpointBarrier,  
        boolean markChannelBlocked)  
        throws CheckpointException, IOException {  
  
    CheckpointBarrier unalignedBarrier = checkpointBarrier.asUnaligned();  
    // 第一次缓存数据
    controller.initInputsCheckpoint(unalignedBarrier);  
    for (CheckpointableInput input : channelState.getInputs()) {  
        input.checkpointStarted(unalignedBarrier);  
    }  
    // 保存完之后触发 ck
    controller.triggerGlobalCheckpoint(unalignedBarrier);
}

public void initInputsCheckpoint(CheckpointBarrier checkpointBarrier)  
        throws CheckpointException {  
    subTaskCheckpointCoordinator.initInputsCheckpoint(  
            barrierId, checkpointBarrier.getCheckpointOptions());  
}

public void initInputsCheckpoint(long id, CheckpointOptions checkpointOptions)  
        throws CheckpointException {  
    if (checkpointOptions.isUnalignedCheckpoint()) {  
        channelStateWriter.start(id, checkpointOptions);  
        // 进行缓存数据
        prepareInflightDataSnapshot(id);  
    }
}

private void prepareInflightDataSnapshot(long checkpointId) throws CheckpointException {  
    // prepareInputSnapshot 是 SubtaskCheckpointCoordinatorImpl 的属性
	// StreamTask 初始化时会创建 SubtaskCheckpointCoordinatorImpl
    prepareInputSnapshot  
            .apply(channelStateWriter, checkpointId)  
            .whenComplete(...);  
}

this.subtaskCheckpointCoordinator =  
        new SubtaskCheckpointCoordinatorImpl(this::prepareInputSnapshot...)

private CompletableFuture<Void> prepareInputSnapshot(  
        ChannelStateWriter channelStateWriter, long checkpointId) throws CheckpointException {  
    if (inputProcessor == null) {  
        return FutureUtils.completedVoidFuture();  
    }  
    return inputProcessor.prepareSnapshot(channelStateWriter, checkpointId);  
}

public CompletableFuture<Void> prepareSnapshot(  
        ChannelStateWriter channelStateWriter, long checkpointId) throws CheckpointException {  
        try {  
	         // 将数据保存在 State
            channelStateWriter.addInputData(  
                    checkpointId,  
                    e.getKey(),  
                    ChannelStateWriter.SEQUENCE_NUMBER_UNKNOWN,  
                    e.getValue().getUnconsumedBuffer());  
        } 
    }  
    return checkpointedInputGate.getAllBarriersReceivedFuture(checkpointId);  
}
```
## 7.2 端到端精准一次
- Source：可以多次重复读取数据
	-  如果 `Source` 本身不能重复读取数据，结果仍然可能是不准确的
- Flink：`Checkpoint` 精准一次
- Sink：
	- 幂等：
		- 利用 `MySQL` 主键 `upsert`
		- 利用 `hbase` 的 `rowkey` 唯一
	- 事务：外部系统提供，2PC, 预写日志
		- 用事务向外部写，与检查点绑定在一起
		- 当 `Sink` 遇到 `Barrier` 时，开启保存状态的同时开启一个事务，接下来所有的数据写入都在这个事务中
		- 等检查点保存完毕时，将事务提交，写入的数据就可以使用了
		- 如果出现了问题，状态会回退到上一个 ck，而事务也会进行回滚
		- 两阶段提交写 `Kafka`
		- 两阶段提交写 `MySQL`
	- 事务的特性
		- `Atomicity`：事务中的操作要么全部完成，要么全部失败
		- `Consistency`：事务按预期生效，数据是预期的状态
		- `Isolation`：多个并发事务之间相互隔离
		- `Durability`：提交事务产生的影响是永久性的
	- WAL 事务提交
		- `Flink` 中提供了接口 `GenericWriteAheadSink`，没有实现类
		- 过程
			- `Sink` 将数据的写出命令保存在日志中(`StreamStateHandle`)
			- 生成事务记录在状态中(`PendingCheckpoint`)
			- 进行持久化存储，向 `coordinator` 发送 handle 回调
			- 收到检查点完成的通知后，将所有结果一次性写入外部系统
			- 成功写入所有数据后，内部更新 `PendingCheckpoint` 的状态，将确认信息持久化，真正完成 `ck`
		- 缺点：最终确认信息时如果发生了故障，只能恢复到上个状态重新写，无法保证幂等性时会造成<font color='red'>重复写入</font>
	- 2PC(2 Phase Commit) 提交
		- `Flink` 中提供了抽象类 `TwoPhaseCommitSinkFunction` 以及新的 `TwoPhaseCommittingSink` 接口，实现类有 `KafkaSink`
		- 需要外部系统支持，`MySQL`, `Kafka` 都支持
		- 过程
			- 每批的第一条数据到达时，开启事务
			- 数据直接输出到外部系统，此时是临时不可见数据(预提交)
			- `ck` 完成后正式提交事务
			- 收到响应更新状态
			- 充分利用了 `ck` 的机制
			- `Barrier` 的到来标志着开始一个新事务
			- 收到 `JM` 的 `ck` 成功的消息，标志着提交事务
		- 如果确认信息时挂了，第二次提交时存在事务 id，不会再进行重复提交，就是精准一次
		- 隔离级别至少为读已提交
- `MySQL` 的事务的隔离级别
```sql
BEGIN -- 开启事务
COMMIT -- 提交事务
ROLLBACK -- 回滚

-- 查看隔离级别
/*
 * @x: 用户变量
 * @@x: 系统变量
*/
select @@TX_ISOLATION

-- 设置会话隔离级别
set SESSION TRANSACTION ISOLATION LEVEL xxx
```
- 读未提交
	- 任何操作<font color='red'>不加锁</font>
	- 会出现脏读
		- T1 修改某个值，T2 读取到后，T1 回滚，T2 的数据无效
		- 一般针对于 `update`
	- 会出现不可重复读
		- 同一个事务内多次读取同一行数据，得到的结果可能不一致
		- 可能有别的事务对其进行了修改
	- 会出现幻读
		- T1 读到了 T2 新插入的数据导致数据不一致
- 读已提交
	- <font color='red'>增删改会增加行锁</font>，读操作不加锁
	- 出现不可重复读，幻读
- 可重复读：出现幻读
- 可串行化：不出现问题，读加<font color='red'>共享锁</font>，写加<font color='red'>排它锁</font>
# 8. SQL
## 8.1 基础 API
### 7.1.1 创建 TableEnvironment
- `TableEnvironment` 用于：
	- 注册 Catalog 和表
	- 执行 SQL 的查询
	- 注册 UDF
	- `DataStrem` 和表之间的转换
- 第一种创建方式：从流环境创建表环境
```java
// TableEnvironment接口，有两个子接口StreamTableEnvironment和TableEnvironmentInternal
// StreamTableEnvironment用于创建Table环境和SQL API的入口，定义了静态的create方法
static StreamTableEnvironment create(StreamExecutionEnvironment executionEnvironment) {  
    return create(executionEnvironment, EnvironmentSettings.newInstance().build());  
}

// 传入流环境来创建表环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();  
StreamTableEnvironment tblEnv = StreamTableEnvironment.create(env);
```
- 第二种创建方式，当通过 `connector` 的方式直接创建表环境时，不需要通过流环境
```java
/*
 * TableEnvironment接口定义了静态的create方法
 * static TableEnvironment create(EnvironmentSettings settings) {  
    return TableEnvironmentImpl.create(settings);
    }
 */
static TableEnvironment create(EnvironmentSettings settings) {  
    return TableEnvironmentImpl.create(settings);  
}

public static TableEnvironmentImpl create(EnvironmentSettings settings) {  

	// 创建manager后生成一个TableEnvironmentImpl对象
	// TableEnvironmentImpl的构造器中会创建一个OperationTreeBuilder，里面包含一个Sql解析器
    return new TableEnvironmentImpl(  
            catalogManager,  
            moduleManager,  
            resourceManager,  
            tableConfig,  
            executor,  
            functionCatalog,  
            planner,  
            settings.isStreamingMode());         
}

/*
 * 需要传入一个EnvironmentSettings对象
 * EnvironmentSettings需要通过内部定义的Builder来实例化
 * EnvironmentSettings的newInstance()方法返回Builder对象
 * 通过Builder的inStreamingMode来设置流式模式
 */
 public EnvironmentSettings build() {  
    if (classLoader == null) {  
        classLoader = Thread.currentThread().getContextClassLoader();  
    }  
    return new EnvironmentSettings(configuration, classLoader);  
}
public Builder inStreamingMode() {  
    configuration.set(RUNTIME_MODE, STREAMING);  
    return this;  
}
```
- 流和表的操作转换
	- Append-only 流：通过 insert 来修改的动态表，标记为(+I)
	- Retract 流：包含添加消息和撤回消息，追加的标记为(+I)，删除为(-D)，更新会产生两条：更新前(-U)和更新后(+U)
	- Upsert 流：只有 Upsert-kafka 连接器模式支持
### 7.1.2 创建表对象
- 表是 `TableAPI` 的核心抽象，表对象描述了一个数据变换的管道，本身不包含数据，只是描述如何从 `DynamicTableSource ` 中读取数据，以及如何向 `DynamicTableSink` 写数据
- `Table` 接口有一个实现类 `TableImpl`，里面实现了对表的各种操作方法
- 表的创建主要通过两种方法
	- 连接器
		通过 `TableEnvironment` 的 `executeSql()` 传入 DDL 语句
	- 虚拟表
		给一个 Table 对象起别名 `TableEnvironment.createTemporaryView()`
### 7.1.3 表的查询
- `Table` 对象中的许多查询方法的参数都是一个 `Expression` 接口实现类的对象, `Expression` 可以通过 `Expressions` 类来实例化
```java
// Expression可以是字面量，函数
public interface Expression {}
// Expressions常用的方法：
// 引用字段
public static ApiExpression $(String name) {  
    return new ApiExpression(unresolvedRef(name));  
}
// 创建一个字面量：lit(12) -> INT, lit(new BigDecimal("123.45")) -> DECIMAL(5, 2)
public static ApiExpression lit(Object v) {  
    return new ApiExpression(valueLiteral(v));  
}
/* 
 * 调用函数
 * 抽象类 UserDefinedFunction主要有
 *     AsyncTableFunction
 *     DeclarativeAggregateFunction
 *     ImperativeAggregateFunction
 *     ScalarFunction
 *     TableFunction这几个子类
 */
public static ApiExpression call(  
        Class<? extends UserDefinedFunction> function, Object... arguments) {  
    final UserDefinedFunction functionInstance =  
            UserDefinedFunctionHelper.instantiateFunction(function);  
    return apiCall(functionInstance, arguments);  
}
```
## 8.2 Connector
### 7.2.1 读写文件
- 建表语句
```sql
```sql
CREATE TABLE MyUserTable (
  column_name1 INT,
  column_name2 STRING,
  ...
  part_name1 INT,
  part_name2 STRING
) PARTITIONED BY (part_name1, part_name2) WITH (
  'connector' = 'filesystem',           -- required: specify the connector
  'path' = 'file:///path/to/whatever',  -- required: path to a directory
  'format' = '...',                     -- required: file system connector requires to specify a format,
                                        -- Please refer to Table Formats
                                        -- section for more details
  'partition.default-name' = '...',     -- optional: default partition name in case the dynamic partition
                                        -- column value is null/empty string
  'source.path.regex-pattern' = '...',  -- optional: regex pattern to filter files to read under the 
                                        -- directory of `path` option. This regex pattern should be
                                        -- matched with the absolute file path. If this option is set,
                                        -- the connector  will recursive all files under the directory
                                        -- of `path` option

  -- optional: the option to enable shuffle data by dynamic partition fields in sink phase, this can greatly
  -- reduce the number of file for filesystem sink but may lead data skew, the default value is false.
  'sink.shuffle-by-partition.enable' = '...',
  ...
)
```
- 可用的 metadata
	- `file.path` -> `STRING NOT NULL`
	- `file.name` -> `STRING NOT NULL`
	- `file.size` -> `BIGINT NOT NULL `
- 读数据示例
```java
// 创建环境
TableEnvironment tblEnv = TableEnvironment.create(EnvironmentSettings.newInstance()  
        .inStreamingMode()  
        .build());

// 建表sql
String createTblSql = "CREATE TABLE t1 (" +  
        " id STRING," +  
        " ts BIGINT," +  
        " vc INT" +  
        ") WITH (" +  
        " 'connector' = 'filesystem', " +  
        " 'path' = 'data/ws.json', " +  
        " 'format' = 'json') ";

// 读数据
tblEnv.executeSql(createTblSql);
```
- 写数据示例
```java
// 通过表环境获取表，流环境写数据
// insert into 目标表 select * from 源表
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();  
StreamTableEnvironment tblEnv = StreamTableEnvironment.create(env);  
env.setParallelism(2);  
SingleOutputStreamOperator<WaterSensor> ds = env.socketTextStream("hadoop102", 8888)  
        .map(new WaterSensorMapFunc());  
  
// 源表  
Table table = tblEnv.fromDataStream(ds);  
tblEnv.createTemporaryView("t2", table);  
// 目标表  
String createTblSql = "CREATE TABLE t1 (" +  
        " id STRING," +  
        " ts BIGINT," +  
        " vc INT" +  
        ") WITH (" +  
        " 'connector' = 'filesystem', " +  
        " 'path' = 'filesink', " +  
        " 'format' = 'json') ";  
  
tblEnv.executeSql(createTblSql);  
  
// 执行插入  
tblEnv.executeSql( "insert into t1 select * from t2");
```


### 7.2.2 读写 Kafka
- 建表语句
```sql
CREATE TABLE KafkaTable (
  `user_id` BIGINT,
  `item_id` BIGINT,
  `behavior` STRING,
  `ts` TIMESTAMP(3) METADATA FROM 'timestamp'
) WITH (
  'connector' = 'kafka',
  'topic' = 'user_behavior',
  'properties.bootstrap.servers' = 'localhost:9092',
  'properties.group.id' = 'testGroup',
  'scan.startup.mode' = 'earliest-offset',
  'format' = 'csv'
)
```
- 可用的 metadata
	- `topic` -> `STRING NOT NULL`
	- `partition` -> `INT NOT NULL`
	- `offset` -> `BIGINT NOT NULL`
- 读写 Kafka 示例
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();  
    StreamTableEnvironment tblEnv = StreamTableEnvironment.create(env);  
  
    SingleOutputStreamOperator<WaterSensor> ds = env.socketTextStream("hadoop102", 8888)  
            .map(new WaterSensorMapFunc());  
  
    val tbl = tblEnv.fromDataStream(ds);  
    tblEnv.createTemporaryView("t2", tbl);  
  
    String sql = "create table t1" +  
            "(id string, ts bigint, vc int) " +  
            "with ('connector'='kafka', 'topic'='t12', " +  
            "'properties.bootstrap.servers'='hadoop102:9092', " +  
            "'sink.partitioner'='fixed', 'format'='csv')";  
  
    // 创建表  
    tblEnv.executeSql(sql);  
    // 写数据  
    tblEnv.executeSql("insert into t1 select * from t2");  
    // 两种流的环境都需要启动
    env.execute();  

}
```
### 7.2.3 读写 JDBC
- 建表语句
```sql
CREATE TABLE MyUserTable (
  id BIGINT,
  name STRING,
  age INT,
  status BOOLEAN,
  PRIMARY KEY (id) NOT ENFORCED --主键需要添加NOT ENFORCED
) WITH (
   'connector' = 'jdbc',
   'url' = 'jdbc:mysql://localhost:3306/mydatabase',
   'table-name' = 'users'
   'driver' = '...'
   'username' = '...'
   'password' = '...'
   
);
```
- 读写数据
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();  
SingleOutputStreamOperator<WaterSensor> ds = env.socketTextStream("hadoop102", 8888)  
        .map(new WaterSensorMapFunc());  
StreamTableEnvironment tblEnv = StreamTableEnvironment.create(env);  
  
tblEnv.createTemporaryView("t2", ds);  
String sql = "create table t1 (" +  
        " id string, " +  
        " ts bigint, " +  
        " vc int, " + 
        -- 写入时必须指定主键以及 not enforced
        -- 不会对主键做强制的唯一性约束、非空约束，只支持这种主键
        " primary key (id) not enforced " +  
        " ) with ( " +  
        " 'connector'='jdbc', " +  
        " 'url'='jdbc:mysql://hadoop102:3306/flinksink?useSSL=false&useUnicode=true&characterEncoding=UTF-8&allowPublicKeyRetrieval=true', " +  
        " 'table-name'='ws', " +  
        " 'username'='root', " +  
        " 'password'='000000', " +  
        " 'driver'='com.mysql.cj.jdbc.Driver'" +  
        ")";

tblEnv.executeSql(sql);  
tblEnv.executeSql("insert into t1 select * from t2");  
env.execute();
```

### 7.2.4 读写 Upsert-Kafka
- 如果当前流中有聚合操作，或者当前表中有聚合操作，那么此时随着数据的不断到达，触发连续查询后就需要不断地更新之前计算的结果。使用普通的 `Kafka Connector` 写出时，由于 `Kafka` 是一个日志系统，只允许 append 追加写，不允许随机修改，所以，无法对聚合场景下的数据进行写出
- 此时需要 `upsert-kafka connector`
- 不同操作类型的数据会使用不同格式写出

| 操作类型   | key  | value      |
| ---------- | ---- | ---------- |
| +I(insert) | 主键 | 数据       |
| +U(update) | 主键 | 更新后数据 |
| -D(delete) | 主键 | NULL |

- 写入
```java
// 不使用`upsert-kafka时，抛异常
// Table sink 'default_catalog.default_database.t1' doesn't support consuming update changes which is produced by node GroupAggregate(...)
// 使用时需要根据聚合字段指定主键，以及修改connector类型
String sql = "CREATE TABLE t1 ( " +  
        "id string primary key not enforced," +  
        "vc INT " +  
        ") WITH ( " +  
        "'connector' = 'upsert-kafka', " +  
        "'topic' = 't11', " +  
        "'properties.bootstrap.servers' = 'hadoop102:9092', " +  
        "'key.format' = 'json', " +  
        "'value.format' = 'json') ";

// 当聚合字段有两个时，使用联合主键
String sql = "CREATE TABLE t1 ( " +  
        " id string," +  
        " ts bigint, " +  
        " vc int, " +  
        // 指定联合主键  
        "primary key (id, ts) not enforced " +  
        " ) WITH ( " +  
        " 'connector' = 'upsert-kafka', " +  
        " 'topic' = 't11', " +  
        " 'properties.bootstrap.servers' = 'hadoop102:9092', " +  
        " 'key.format' = 'json', " +  
	    " 'value.format' = 'json') ";
```
- 读取
```java
String createTableSql = 
	" create table t1 ( id string , vc int, maxTs bigint ," +
	" PRIMARY KEY (id,vc)  NOT ENFORCED )" +
	" with ( " +
	" 'connector' = 'upsert-kafka' ,   " +
	" 'topic' =  'topicE'  ," +
	" 'properties.group.id' = 'haha', " +
	// 无需设置位置策略，因为upsert模式保存一个changelog流，只有从头读取才能知道数据的更新变化情况
	// "'scan.startup.mode' = 'earliest-offset', " +
	" 'properties.bootstrap.servers' = 'hadoop102:9092'," +
	" 'key.format' = 'json' ," +
	" 'value.format' = 'json' " +
	" )";
```
## 8.3 Function
### 7.3.1 使用系统自带函数
```java
String sql = "create table t1 ( " +  
        " id string, " +  
        " ts bigint, " +  
        " vc int " +  
        " ) with ( " +  
        " 'connector'='filesystem', " +  
        " 'path'='data/ws.json', " +  
        " 'format'='json')";  
  
tblEnv.executeSql(sql);  
tblEnv.sqlQuery("select *, concat(id, 'xxx') str from t1").execute().print();
```
### 7.3.2 ScalarFunction
- 输入一行，输出一行一列
- 要自定义标量函数，必须继承 `ScalarFunction` 类，不能声明为私有类，必须实现一个公开的 `eval()`方法
```java
// 如果字符串不是NULL，则转换为大写，否则返回字符串NULL
String sql = "create table t1 ( " +  
        " id string, " +  
        " ts bigint, " +  
        " vc int " +  
        " ) with ( " +  
        " 'connector'='filesystem', " +  
        " 'path'='data/ws1.json', " +  
        " 'format'='json')";  
tblEnv.executeSql(sql);  
// 自定义函数名不能和内置的函数重名
tblEnv.createTemporaryFunction("upperId", new MyScalar());  
tblEnv.sqlQuery("select id, ts, upperId(id) cap from t1").execute().print();
```
### 7.3.3 AggregateFunction
- 输入 N 行，输出一行一列
- 对每一个需要聚合的行，调用 `ACC createAccumulator()` 创建累加器，最后通过 ` T getValue(Object)` 返回聚合结果
- 聚合方法必须是 public，非 static，命名为 `void accumulate(ACC accumulator, I input)`
- 自定义函数必须继承 `AggregateFunction` 类
```java
// T -> 聚合结果类型
// ACC -> 聚合中间结果
public abstract class AggregateFunction<T, ACC> extends ImperativeAggregateFunction<T, ACC> {}

// 求每种传感器的平均水位
public static class MyAgg extends AggregateFunction<Double, MyAcc> {  
  
    @Override  
    public Double getValue(MyAcc accumulator) {  
        return accumulator.getVc() / accumulator.getCount();  
    }  
  
    @Override  
    public MyAcc createAccumulator() {  
        return new MyAcc(0, 0d);  
    }  
  
    public void accumulate(MyAcc acc, Integer vc) {  
        acc.count += 1;  
        acc.vc += vc;  
    }  
}  
  
// 定义一个MyAcc类保存中间状态
public static class MyAcc {  
    private Integer count;  
    private Double vc;  
}
```
### 7.3.4 TableFunction(UDTF)
- 输入一行 N 列，输出 N 行 N 列
```java
/*
 * 需要继承TableFunction<T>
 * 需要实现一个void的eval方法，在其中通过TableFunction中定义的collector收集结果进行输出
 * 一般使用Row类型作为输出的一行数据
 * 返回的T类型可以通过FunctionHint注解，在其中通过DataTypeHint注解对其output属性赋值
 */

// 将s1_a_b 按照'_'切分，输出整个id的长度 -> s1,6/n a,6/n b,6
@FunctionHint(output = @DataTypeHint("row<word string, length int>"))  
public static class MyTbl extends TableFunction<Row> {  
  
   public void eval(String s) {  
       if (!s.contains("s3")) {  
           for (String string : s.split("_")) {  
               collect(Row.of(string, s.length()));  
           }  
       }  
   }  
}

/* 
 * Hive中的lateral view语法:
 * lateral view explode(col) 临时表名 as 临时表col1, col2...
 * explode 在执行时会解析为join
 * 在flinksql中，通过" (LEFT) JOIN LATERAL TABLE(SplitFunction(myField)) ON TRUE" 来实现
 */
tblEnv.sqlQuery("select id, ts, vc, word, length from t1 " +  
        "left join lateral table(tblSplit(id)) on true"
```
### 7.3.5 TableAggregateFunction(UDTAF)
- 本质是 UDTF，只是输出的结果需要聚合后再进行输出
- 需要继承 `TableAggregateFunction<T, ACC>`
- 需要实现 
	- `void emitValue(ACC acc, Collector<Row> out)`
	- `ACC createAccumulator()`
	- `void accumulate(ACC accumulator ,I input)`
```java
// 统计每种传感器的最大vc的前两位
@FunctionHint(output = @DataTypeHint("row<rank string, top int>"))  
public static class MyTblAgg extends TableAggregateFunction<Row, Top2> {  
  
    @Override  
    public Top2 createAccumulator() {  
        return new Top2(0, 0);  
    }  
  
    public void accumulate(Top2 acc, Integer vc) {  
        if (vc > acc.firstVc) {  
            // 比最大的值大，更改top2  
            acc.secondVc = acc.firstVc;  
            acc.firstVc = vc;  
        }  
        else if (acc.secondVc != null && vc > acc.secondVc) {  
            // 存在第二，且vc比第二大，更新第二  
            acc.secondVc = vc;  
        }  
    }  
  
    public void emitValue(Top2 top2, Collector<Row> out) {  
        out.collect(Row.of("first", top2.firstVc));  
        if (top2.secondVc > 0) {  
            out.collect(Row.of("second", top2.secondVc));  
        }  
    }  
}  

public static class Top2 {  
    private Integer firstVc;  
    private Integer secondVc;  
}

// UDTAF只支持SqlApi
// 先groupBy再调用flatAggregate
t1.groupBy($("id"))  
        .flatAggregate(Expressions.call(new MyTblAgg(), $("vc")))  
        .select($("id"), $("rank"), $("top"))  
        .execute()  
```
## 8.4 SQL 窗口
### 8.4.1 定义时间
#### 8.4.1.1 TableApi
- `TableApi` 中，`Schema` 类代表一张表的元数据信息，可用通过 `column()` 方法，指定 `POJO` 中的属性构造一列，也可以通过 `columnByExpression()` 方法，通过表达式构造列
- `TableApi` 中的时间
- 根据原始数据中不同的时间格式通过不同的函数来定义
	- `2023-04-01 20:13:40:444` -> 带毫秒的通过 `TIMESTAMP(3)` 转换
	- `2023-04-01 20:13:40` -> 不带毫秒的通过 ` TIMESTAMP(0) ` 转换
	- `1618989564564` -> 13 位毫秒时间戳通过 `TIMESTAMP_LTZ(3)` 转换
	- `1618989564` -> 秒时间戳通过 `TIMESTAMP_LTZ(0)` 转换
- flink 中也提供了几个转换时间的函数
	- `PROCTIME()` -> 获取当前的处理时间
	- `TO_TIMESTAMP_LTZ(ts,3)` -> 将 long 型 ts 转换为 `TIMESTAMP_LTZ(3)`
	- `TO_TIMESTAMP_LTZ(ts,0)` -> 将 long 型 ts 转换为 `TIMESTAMP_LTZ(0)`
	- `SOURCE_WATERMARK()` -> 获取流中原有的水印
	- `事件事件 - INTERVAL 'n' 时间单位` -> 根据事件计算水印
```java
Schema schema = Schema.newBuilder()  
        .column("id", "string")  
        .column("ts", "bigint")  
        .column("vc", "int")  
        // columnByExpression的第二个参数是String类型，可以直接传入表达式
        .columnByExpression("vc+10", Expressions.$("vc").plus(10))  
        .columnByExpression("vcPlus20", "vc + 20")  
        .columnByExpression("pt", "PROCTIME()")  
        .columnByExpression("etmillts", "TO_TIMESTAMP_LTZ(ts, 3)")  
        .columnByExpression("etsecond", "TO_TIMESTAMP_LTZ(ts, 0)")
        // 生成水印  
        .watermark("etmillts", "etmillts - interval '0.001' second")  
        .build();
```
#### 8.4.1.2 SqlApi
```java
String sql = "create table t1 ( " +  
        " id string, " +  
        " ts bigint, " +  
        " vc int, " +  
        " pt as proctime(), " +  
        " et as to_timestamp_ltz(ts, 3), " +  
        " watermark for et as et - interval '0.001' second " +  
        " ) with ( " +  
        " 'connector'='filesystem', " +  
        " 'path'='data/ws.json'," +  
        " 'format'='json')";
```
### 8.4.2 计数窗口
- 计数窗口只有 `TableApi`，需要使用 processingTime，根据处理时间判断到达的顺序
```java
Schema schema = Schema.newBuilder()  
        .column("id", "string")  
        .column("ts", "bigint")  
        .column("vc", "int")  
        .columnByExpression("pt", "proctime()")  
        .build();

// 使用Tumble类创建滚动窗口
TumbleWithSizeOnTimeWithAlias w1 = Tumble
		// over方法定义窗口大小
		.over(rowInterval(3L))  
		// on方法指定用于分组的时间属性
        .on($("pt"))  
        // 起别名
        .as("w");

// 使用Slide类创建滑动窗口
SlideWithSizeAndSlideOnTimeWithAlias w2 = Slide
		.over(rowInterval(3L))
		// every指定滑动长度  
        .every(rowInterval(2L))  
        .on($("pt"))  
        .as("w");
        
// 窗口的并行度在运行时指定
// 通过window方法开窗
tbl.window(w2)  
	// 只根据窗口分组，全局窗口  
	.groupBy($("w"))  
	.select($("vc").sum().as("sumVc"))  
	.execute().print();
```
### 8.4.3 时间窗口
- 时间窗口一般通过 SQL 来定义，不使用 `GroupWindow` 语法，使用最新的 `TVF(table-valued functions) Window` 语法
#### 8.4.3.1 滚动窗口
- 语法
TUMBLE(<font color='red'>TABLE data</font>, <font color='red'>DESCRIPTOR</font>(timecol), <font color='red'>size</font> \[, <font color='red'>offset</font> ])
- `data` -> 表名
- `DESCRIPTOR` -> 传入一个时间字段
```java
Schema schema = Schema.newBuilder()  
        .column("id", "string")  
        .column("ts", "bigint")  
        .column("vc", "int")  
        .columnByExpression("pt", "proctime()")  
        .build();  
  
tblEnv.createTemporaryView("t1", ds, schema);  
  
String sql = "select id, " +  
        " window_start, " +  
        " window_end, " +  
        " sum(vc) sumVc " +  
        " from table( " +  
        " tumble(table t1, descriptor(pt), interval '3' second)) " +  
        // 固定写法，用到了其他字段，表示keyed窗口
        " group by window_start, window_end, id";
```
#### 8.4.3.2 滑动窗口
- 语法
HOP(<font color='red'>TABLE data</font>, <font color='red'>DESCRIPTOR</font>(timecol), slide, <font color='red'>size</font> \[, <font color='red'>offset</font> ])
```java
// 使用与滚动窗口类似
String sql = "select " +  
        " window_start, " +  
        " window_end, " +  
        " sum(vc) sumVc " +  
        " from table( " +  
        // size must be an integral multiple of slide  
        " hop(table t1, descriptor(et), interval '3' second, interval '6' second )) " +  
        " group by window_start, window_end";
```

#### 8.4.3.3 会话窗口
- 语法
- 会话窗口不支持 TVF 语法，只能通过 GroupWindow 语法来实现
```java
String sql = "select " +  
        " session_start(pt, interval '3' second), " +  
        " session_end(pt, interval '3' second), " +  
        " sum(vc) sumVc " +  
        " from t1 " +  
        " group by session(pt, interval '3' second)";
```
#### 8.4.3.4 累积窗口
- 累积窗口适用于每隔一段固定时间进行计算
![[cumulate_window.png]]
- 每次计算 `window step` 大小的数据，当大小达到 `max window size` 后，从头接着上个窗口的数据继续计算
- 通过 `CUMULATE(TABLE 表名, DESCRIPTOR(时间列), step, size)` 函数定义累积的时间窗口。其中 step 为计算的步长而 size 为累积的最大范围
```java
String sql = "select " +  
        " window_start, " +  
        " window_end, " +  
        " sum(vc) sumVc " +  
        " from table ( " +  
        " cumulate(table t1, descriptor(et), interval '2' second, interval '6' second )) " +  
        " group by window_start, window_end";
```

### 8.4.4 增强聚合
#### 8.4.4.1 GroupingSets
- 场景：需要聚合的字段相同，但是分组方式不同
```java
// select后面加上所有需要分组的字段，在groupby后加上grouping sets
// 每对()中对应一组分组字段
.sqlQuery("select ... from ... group by " +  
	" grouping sets ((a), (b), (c, d, ...))").execute().print();
```
- GroupingSets 也可用在窗口聚合中
```java
String sql = "select id, ts, " +  
        " window_start, " +  
        " window_end, " +  
        " sum(vc) sumVc " +  
        " from table( " +  
        " tumble(table t1, descriptor(pt), interval '3' second)) " +  
        " group by window_start, window_end, grouping sets((id), (ts))";
```
#### 8.4.4.2 RollUp
- 分组字段由右至左逐渐减少
- rollup(a, b, c) -> 按 abc 分组 -> 按 ab 分组 -> 按 a 分组 -> 按 null 分组
```java
.sqlQuery("select ... from ... group by " +  
	" rollup(a, b, c,...)").execute().print();
```
#### 8.4.4.3 Cube
- cube(a, b, c, ...)会按 $$
C_{n}^{n}+C_{n}^{n-1}+\dots+C_{n}^{1}+C_{n}^{0}=2^n
$$
种方式进行分组
```java
.sqlQuery("select ... from ... group by " +  
        " cube(a, b, ...)").execute().print();
```
### 8.4.5 over 窗口
- `hive` 中的 `over()`:
- `窗口函数() over(partition by xxx order by xxx rows | range between ... and)`
- `flink` 的无界流使得下边界是不确定的，因此 `flink` 的 `over()` 强制将下边界规定为 `current row`
- rows
```java
/* rows
 * order by 只能升序
 * order by在流模式下只能定义时间属性(pt或et，ts不可以)
 * 不指定窗口范围会取整个窗口
 * 上限可以向前指定
 */
String rowSql = "select  " +  
        " id, ts, vc, " +  
        // " sum(vc) over(partition by id order by pt rows between unbounded preceding and current row) " +  
        " sum(vc) over(partition by id order by pt rows between 1 preceding and current row) " +  
        " from t1";

// 如果按照事件事件排序，只有当后面的水印>=当前事件时间时才会触发运算
" sum(vc) over(partition by id order by et rows between 1 preceding and current row) "
// 同一个事件时间的数据也会放在不同的窗口中进行计算
```
- range
```java
// range会将事件时间之前的所有数据全部计算
" sum(vc) over(partition by id order by et range between unbounded preceding and current row) " 
/*
 * +------+----+-----+
 * |  ts  | vc | sum |
 * +------+----+-----+
 * | 1000 |   1|    6|
 * | 1000 |   4|    6|
 * | 1000 |   1|    6|
 */

// 也可以通过interval处理processing time
" sum(vc) over(partition by id order by et range between interval '2' second preceding and current row) " 
// 将[ts-2000,ts]范围的通过同一个窗口计算
```
- 窗口的复用
- 当需要通过一个窗口实现多个聚合逻辑时会报错
```java
String multiSql = "select  " +  
        " id, ts, vc, " +  
        " sum(vc) over(partition by id order by pt rows between unbounded preceding and current row) sumVc, " +  
        " min(vc) over(...) minVc, " +  
        " max(vc) over(...) maxVc, " +  
        " from t1";
// 此时可以将窗口进行复用
String multiSql = "select  " +  
        " id, ts, vc, " +  
        " sum(vc) over w sumVc, " +  
        " min(vc) over w minVc, " +  
        " max(vc) over w maxVc " +  
        " from t1 " +  
        " window w as (partition by id order by pt rows between unbounded preceding and current row)";
```
## 8.5 Join
### 8.5.1 CommonJoin
- `commonjoin` 指普通的 `inner join`，`left join` 等，两张表进行 join 时，表中的数据会缓存在状态中，占用资源会越来越大，可以通过配置状态的 `ttl` 来回收
```java
// inner join每关联一条向结果表中增加一条 +I 的数据
tblEnv.sqlQuery("select t1.id, t1.vc, t2.id, t2.vc " +  
        " from t1 join t2 " +  
        " on t1.id = t2.id").execute().print();  
env.execute()

/* 
 * left join取左表全部数据
 * 一开始如过左面没有关联成功，就会增加一条 +I 的数据，等关联成功时，会删除之前关联的结果(-D) ，生成一条新的结果
 * left join 会产生 -D 的数据，只能通过Kafka-upsert写出
 */

// 可以通过表环境设置ttl
.getConfig().setIdleStateRetention(Duration duration);
// 或者通过config设置table.exec.state.ttl
.getConfig().set("table.exec.state.ttl", "毫秒值")
```


### 8.5.2 IntervalJoin
```java
// 不使用join关键字
String sql = "select t1.id, t2.id, t1.ts, t2.ts " +  
        " from t1, t2 " +  
        " where t1.id = t2.id and " +  
        " t2.et " +  // 对面流的时间
        " between t1.et - interval '2' second and t1.et + interval '2' second"; // 在当前表的时间范围中
```
### 8.5.3 LookupJoin
- 维度表一般存储在外部系统，如 `Mysql` 或 `HBase` 中， `lookupjoin` 用于事实表 `join` 维度表的场景，要求维度表中有<font color='skyblue'>处理时间</font>字段
```java
String joinSql = "select t1.id, t2.name " +  
        " from t1 " +  
        // for system_time as of + 处理时间字段
        " left join t2 for system_time as of t1.pt" +  
        " on t1.id = t2.id";
```

## 8.6 Catalog
- Catalog 是库的上一级，用于区分相同库下相同的表名
- Catalog 提供了元数据信息，例如数据库、表、分区、视图以及数据库或其他外部系统中存储的函数和信息
- Catalog 接口
	- `AbstractCatalog` 抽象类
		- `AbstractJdbcCatalog` 
			- `JdbcCatalog`
			- `MySqlCatalog`
			- `PostgresCatalog`
		- `GenericInMemoryCatalog`
		- `HiveCatalog`
### 8.6.1 默认 Catalog
```java
// default_catalog
System.out.println(tblEnv.getCurrentCatalog());  
// default_database
System.out.println(tblEnv.getCurrentDatabase());
// 默认使用GenericInMemoryCatalog，会将Calalog的信息存在内存中
```
### 8.6.2 JdbcCatalog
- `AbstractJdbcCatalog` 有三个子类
- `JdbcCatalog` 可以直接对接 `JDBC` 中的库和表，无需创建 `Flink` 的表映射，即可读取
- 只能读取不能写入数据
```java
// MySqlCatalog的构造器
public AbstractJdbcCatalog(  
        ClassLoader userClassLoader,  
        String catalogName,  
        String defaultDatabase,  
        String username,  
        String pwd,  
        String baseUrl) {  
    super(catalogName, defaultDatabase);  
  
    checkNotNull(userClassLoader);  
    checkArgument(!StringUtils.isNullOrWhitespaceOnly(username));  
    checkArgument(!StringUtils.isNullOrWhitespaceOnly(pwd));  
    checkArgument(!StringUtils.isNullOrWhitespaceOnly(baseUrl));  
  
    JdbcCatalogUtils.validateJdbcUrl(baseUrl);  
  
    this.userClassLoader = userClassLoader;  
    this.username = username;  
    this.pwd = pwd;  
    this.baseUrl = baseUrl.endsWith("/") ? baseUrl : baseUrl + "/";  
    this.defaultUrl = this.baseUrl + defaultDatabase;  
}

// 创建一个MySqlCatalog
MySqlCatalog mySqlCatalog = new MySqlCatalog(  
        CatalogTest.class.getClassLoader(),  
        "mysql",  
        "gmall",  
        "root",  
        "000000",  
        // baseUrl只用写这一段
        "jdbc:mysql://hadoop102:3306"  
);

// 注册到CatalogManager
tblEnv.registerCatalog("catalog", mySqlCatalog);

public void registerCatalog(String catalogName, Catalog catalog) {  
    catalogManager.registerCatalog(catalogName, catalog);  
}

// CatalogManager设置使用MySqlCatalog
// 注册时catalogManager将注册的catalog添加到map中，因此通过注册时的名字进行访问
tblEnv.useCatalog("catalog");
public void useCatalog(String catalogName) {  
    catalogManager.setCurrentCatalog(catalogName);  
}
```
### 8.6.3 HiveCatalog
- `HiveCatalog` 不仅可以直接对接 `Hive` 中的库和表，还可以把 `Flink` 中定义的表的元数据直接存储到 `Hive` 的元数据存储中
- 使用时需要开启元数据服务
```java
// HiveCatalog的构造器
// 将hive-site.xml所在目录传入
public HiveCatalog(String catalogName, @Nullable String defaultDatabase, @Nullable String hiveConfDir) {  
    this(catalogName, defaultDatabase, (String)hiveConfDir, (String)null);  
}

public HiveCatalog(String catalogName, @Nullable String defaultDatabase, @Nullable HiveConf hiveConf, String hiveVersion, boolean allowEmbedded) {  
    super(catalogName, defaultDatabase == null ? "default" : defaultDatabase);  
    this.hiveConf = hiveConf == null ? createHiveConf((String)null, (String)null) : hiveConf; 
    ...
}
```
## 8.7 Module
- `Module` 接口定义了一组元数据，包括函数，用户定义类型等
```java
// Module接口有两个实现类CoreModule和HiveModule
// 默认加载的是CoreModule
tblEnv.listModules();
// 通过ModuleManager列出Module
public String[] listModules() {  
    return moduleManager.listModules().toArray(new String[0]);  
}
// 如果想使用hive中的函数，就要将HiveModule引入
HiveModule hiveModule = new HiveModule();
// 通过ModuleManager加载module，将module放在loadedModules map中
tblEnv.loadModule("hive", hiveModule);
public void loadModule(String moduleName, Module module) {  
    moduleManager.loadModule(moduleName, module);  
}
// 设置要使用的module
// addAll全部添加到集合中
tblEnv.useModules("hive", "core");
```
## 8.8 SqlHint
- `SqlHint` 用于临时修改表中的元数据，只在当前查询有效
- 在需要给出提示的地方使用: `/*+ OPTIONS('k'='v') */`，传入 k, v
```java
// 修改连接器中的参数
env.sqlQuery("select * from t1 /*+ OPTIONS('path'='...') */ ") .execute().print();
```
## 8.9 SqlClient
- flink 提供了一个命令行的客户端，类似 hivecli，使用前必须先启动 Session 集群，再提交 sql：`/bin/sql-client.sh`
- sqlClient 可以设置三种显示模式：`SET sql-client.execution.result-mode=`
	- 默认 table
	- tableau
	- changelog：在 table 的基础上显示+I, +U 等标识
- 设置执行环境：`SET execution.runtime-mode=streaming`
- 执行一个 sql 文件：`sql-client.sh -i xxx.sql`
- 使用 savepoint
```shell
# 停止job，触发savepoint
SET state.savepoints.dir='hdfs://...'
STOP JOB 'jobid' WITH SAVEPOINT

# 从savepoint恢复
SET execution.savepoint.path='...' # 之前保存的路径
# 直接提交sql，就会从之前的状态恢复
```
# 9. 优化
## 9.1 资源调优
### 9.1.1 内存设置
- JobManager 内存模型

![[JM_mem_model.svg]]

- 进程内存= `Flink` 内存 + `JVM` 内存
- `Flink` 内存 = 框架堆内堆外内存 + `Task` 堆内堆外内存 +  网络缓存内存 + 管理内存
- `JVM Overhead = 总 mem * 0.1`
- `Network = Flink内存 * 0.1`
- `Managed Memory` 主要用于 `RocksDBStateBackend` 使用，可以适当调至 `0`
### 9.1.2 合理利用 CPU 资源
- 容量调度器默认使用 `DefaultResourceCalculator`，只根据内存来调度资源，因此资源管理页面上每个容器的 `vcore` 数为 1
- 每个 `TM` 中的 `Slot` 共用一个 `CPU`
- 可修改为 `DominantResourceCalculator`，每个 `Slot` 分一个 `CPU`
```xml
<!-- capacity-scheduler.xml -->
<property>
 <name>yarn.scheduler.capacity.resource-calculator</name>
 <value>org.apache.hadoop.yarn.util.resource.DominantResourceCalculator</value>
</property>
```
- 使用 `DominantResourceCalculator` 在提交任务时可以手动指定 
	- `-Dyarn.containers.vcores=1`
### 9.1.3 并行度设置
- 全局并行度
	- 总 `QPS`(每秒查询率) / 单并行度处理能力 = 并行度
	- 根据高峰时期 `QOS`，并行度 * 1.2 倍 
- Source 对接 `Kafka`，并行度设置为 `Kafka` 对应 `Topic` 的分区数
- Transform
	- `KeyBy` 前一般和 `Source` 保持一致
	- `KeyBy` 后，并行度尽量设置为 2 的整数幂
- Sink
	- 写 `Kafka`：与分区数相同
	- 写 `Doris`：批处理，3s 开窗聚合，数据量小，并行度足够
	- 写 `HBase`：缓慢变化维(`SCD`)
### 9.1.4 最终的资源配置
- JobManager: `1 CPU` `2-4G`
- TaskManager：
	- 根据 `Kafka` 分区决定并行度(一个 `Slot` 共享组)，`Slot 数 = 并行度`
	- 资源充足或数据量大 `cpu : slot = 1 : 1`
	- 反之 `cpu : slot = 1 : 2`
	- `1 CPU` `4G` 
## 9.2 状态及 Checkpoint 调优
### 9.2.1 RocksDB 大状态调优
- `RocksDB` 是内存 + 内盘，可以存储大状态
#### 9.2.1.1 开启 State 访问性能监控
- 可以查看有状态算子的读写时间
- 提交时使用参数 `-Dstate.backend.latency-track.keyed-state-enabled=true`
#### 9.2.1.2 开启增量检查点和本地恢复
- `RocksDB` 是目前唯一可用于支持有状态流处理应用程序增量检查点的状态后端
- 开启参数
	- `-Dstate.backend.incremental=true`
	- `-Dstate.backend.local-recovery=true` 不去 `HDFS` 拉取数据，基于本地状态信息恢复任务
	- `-Dstate.backend.latency-track.keyed-state-enabled=true`
- 预定义选项：
	- `DEFAULT`
	- `SPINNING_DISK_OPTIMIZED`
	- `SPINNING_DISK_OPTIMIZED_HIGH_MEM`
	- `FLASH_SSD_OPTIMIZED`
	- 预定义选项达不到预期后，可以调整下面的参数
- 调整预定义选项
	- 增大 `Block` 缓存
		- `state.backend.rocksdb.block.cache-size 默认8m` 设置到 `64-256MB`
	- 增大 `writeBuffer` 和 `level` 阈值大小
		- 每个 `State` 使用一个列族，每个列族 `writeBuffer` 的大小为 `64MB`
			- `state.backend.rocksdb.writebuffer.size`
		- 调整 `Buffer` 需要适当增加 `L1` 的大小阈值，默认 `256MB`
			- 太小会造成存放的 `SST` 文件过少，层级变多
			- 太大会造成文件过多，合并困难
			- `state.backend.rocksdb.compaction.level.max-size-level-base`
	- 增大 `writeBuffer` 数量，L0 中表的个数，默认 2，可以调大到 5
		- `state.backend.rocksdb.writebuffer.count`
	- 增大后台线程数，默认 1，可以增大到 4
		- `state.backend.rocksdb.thread.num`
	- 增大 `writeBuffer` 合并数，默认 1，可以调成 3
		- `state.backend.rocksdb.writebuffer.number-to-merge`
	- 开启分区索引
		- `state.backend.rocksdb.memory.partitioned-index-filters: true`
### 9.2.2 Checkpoint 设置
- 间隔时间可以设置为分钟级别(`1 - 5min`)
- `Sink` 使用事务写出时，`ck` 的时间可设置为秒级或毫秒级
- 超时时间
- Barrier 的对齐(减少状态大小) + 超时时间，超时后转为非对齐
	- `setCheckpointTimeout()`
	- `enableUnalignedCheckpoints()`
- 重启策略
	- 默认间隔 1s 重试`INTEGER.MAX_VALUE` 次
	- `setRestartStrategy()` 每隔 5s 重试 3 次
## 9.3 反压
### 9.3.1 现象
- `WebUI` -> `BackPressure`
	- 0 - 10% 绿
	- 10 - 50% 黄
	- 50% 以上 红
	- 多个 SubTask 可能全红，也有可能有红有绿

![[bkpressure.png]]
- `WebUI` -> `Metrcis`
	- `outPoolUsage`：发送端 `Buffer` 的使用率
	- `inPoolUsage`：接收端 `Buffer` 的使用率
	- `floatingBuffersUsage`(1.9+)：接收端 `Floating Buffer` 的使用率
	- `exclusiveBuffersUsage`(1.9+)：接收端 `Exclusive Buffer` 的使用率
	- `inPoolUsage = floatingBuffersUsage + exclusiveBuffersUsage`
	- 如果 `SubTask` 发送端 `Buffer` 占用过高，说明受到下游反压限速
	- 如果 `SubTask` 接收端 `Buffer` 占用过高，说明传导反压至上游
- 定位问题
	- 禁用任务链
	- 一般是反压的最后一个 `Task`
	- 可以设置参数开启火焰图
		- `-Drest.flamegraph.enabled=true`
	- 也可以设置 JVM 参数，打印 GC 日志进行分析
		- `-Denv.java.opts="-XX:+PrintGCDetails -XX:+PrintGCDateStamps`

![[flame_graph.png]]
### 9.3.2 危害
- 延迟越来越高
- 主动拉取的 `Source`：数据积压(`Kafka`)
- 被动接收的 `Source`：`OOM`
- 状态过大
- `Checkpoint` 超时失败
### 9.3.3 原因
- 数据倾斜：有红有绿
- 资源不足：全红且繁忙
- 外部交互：全红不繁忙
### 9.3.4 解决方法
- 增加资源：内存，`CPU`，并行度
- 旁路缓存，异步 `IO`
## 9.4 数据倾斜
- 反压且有红有绿一定是数据倾斜，数据倾斜不一定产生反压
- `WebUI` -> `SubTasks` -> `SubtaskMetrics`
	- 观察多个 `SubTask` 之间的 `Bytes Received`
#### 9.4.1 keyBy 前
- 上游数据分布的不均匀，有的算子处理的数据多
- 需要通过重分区算子进行 `shuffle`
- `Kafka` 按 `Key` 写入时可能造成数据倾斜(不按 `Key` 走黏性分区器不会造成倾斜)，`Upsert Kafka left join` 时指定了 `Key`
#### 9.4.2 keyBy 后
- 直接聚合
	- 预聚合
		- 用状态攒批，按时间或条数输出
	- 加随机数双重聚合
		- <font color='red'>会出现计算错误，实时来一条就会处理一条，最终处理的数量没变</font>
- 开窗聚合
	- 预聚合
		- <font color='red'>聚合后窗口的范围不能确定，会丢失数据的开窗属性</font>
	- 加随机数双重聚合
		- 第一阶段 `key` 拼接随机数开窗聚合
		- 第二阶段按 `key` 以及窗口的 `start` 或 `end` 进行分组聚合
## 9.5 Job 优化
### 9.5.1 使用 DataGenerator 造数据
### 9.5.2 指定算子的 UID
- 指定方式 `.uid().name()`
- `checkpoint` 后如果更改了业务逻辑，添加或者删除了算子，重启后状态和算子的映射在不指定 `UID` 时会无法建立，会导致任务无法重启
### 9.5.3 测量链路延迟
- 监测数据输入、计算和输出的及时性
- 开启方式 
	- `metrics.latency.interval: 30000`
	- `metrics.latency.granularity: operator` 粒度，默认算子
### 9.5.4 开启对象重用
-  设置 `enableObjectReuse()`
- 开启后 Flink 会省略深拷贝的步骤，同一个 Slot 中传输时只发送地址值而非对象
- 适用于
	- 一个对象只会被一个下游 Function 处理
	- 所有下游 Function 都不会改变对象内部的值
### 9.5.5 细粒度滑动窗口优化
- 细粒度滑动窗口，指窗口长度远大于滑动步长，重叠的窗口过多，会产生重复计算
- 可以将其转换为滚动窗口，之后进行在线存储和读时聚合
## 9.6 SQL 优化
- 设置 `ttl`
- 开启 `MiniBatch` 攒批
- ![[minibatch_agg.png]]
	- `configuration.setString("table.exec.mini-batch.enabled", "true")`
	- `configuration.setString("table.exec.mini-batch.allow-latency", "5 s")`
	- `configuration.setString("table.exec.mini-batch.size", "20000")`

- LocalGlobal 解决数据倾斜问题，需要和 `MiniBatch` 一起使用
	- `configuration.setString("table.optimizer.agg-phase-strategy", "TWO_PHASE")`
- Split Distinct
	- 加随机数打散再聚合
	- 开启方式(结合 `MiniBatch`)
		- `table.optimizer.distinct-agg.split.enabled`
		- `table.optimizer.distinct-agg.split.bucket-num` 打散的 `bucket` 数量，默认 1024
![[split distinct.png]]
- count distinct 搭配 case when 时可以优化为 filter
```sql
SELECT
 a,
 COUNT(DISTINCT b) AS total_b,
 COUNT(DISTINCT CASE WHEN c IN ('A', 'B') THEN b ELSE NULL END) AS AB_b,
 COUNT(DISTINCT CASE WHEN c IN ('C', 'D') THEN b ELSE NULL END) AS CD_b
FROM T
GROUP BY a

------>

SELECT
 a,
 COUNT(DISTINCT b) AS total_b,
 COUNT(DISTINCT b) FILTER (WHERE c IN ('A', 'B')) AS AB_b,
 COUNT(DISTINCT b) FILTER (WHERE c IN ('C', 'D')) AS CD_b
FROM T
GROUP BY a
```
# 10. 复习
## 10.1 与 SparkStreaming 的对比
- 本质
	- `Flink` 基于事件触发计算，真正意义上的流处理框架
	- `SparkStreaming` 基于时间触发计算，根据批大小进行处理的微批次处理框架
- 时间语义：`SparkStreaming` 处理时间, `Flink` 有进入，事件，处理三种语义
- 状态编程：
	- `SparkStreaming` 只有一个有状态的算子
	- `updateStateByKey`：当任务挂掉重启时，会将挂掉到当前为止的全部任务一次进行计算，会对内存造成很大负担
	- `Flink` 所有的算子都有状态
- Checkpoint
	- `SparkStreaming` 将所有数据全部保存，包括代码
	- `Flink` 只保存状态数据，可以修改业务逻辑
- 吞吐量
	- `SparkStreaming` 的吞吐量大，因为只有一次网络请求
## 10.2 基础概念
- JobManager
	- 负责资源申请
	- 任务切分
	- 任务管理
	- `Checkpoint` 触发
	- `WebUI`
- TaskManager
	- 提供 `Slot` 执行 `Task`
- 算子链
	- 作用：合并 Task
	- 条件
		- One-to-one 操作：
			- `source, map, flatmap, filter`
			- 不需要重新分区，不需要调整数据顺序
			- 类似 `Spark` 的窄依赖
		- 并行度相同
- Slot 共享组

![[slot_group.svg]]
	- 不同算子的 `Task` 可以共享 `Slot`
	- `Job` 所需 `Slot` 数：各个共享组最大并行度之和
	- 只要属于同一个作业，不同任务的并行子任务可以放到同一个 `slot` 上执行
## 10.3 运行模式、提交方式、提交流程
- 运行模式：Yarn
	- yarn-session: 先创建集群，再提交任务
	- yarn-per-job: 先提交任务，再创建集群，`Driver` 在 `SparkSubmit` 中
	- yarn-application：先提交任务，再创建集群，`Driver` 在 `JM` 中
	- 创建四种 `Graph` 的位置不同
- 提交方式：
	- 脚本：封装启动任务命令
		- web 端口号
		- 任务名称：默认 `flinkStreamingJob`
	- `StreamPark`
- 提交流程
	- yarn-per-job
		- 本地提交程序：
			- `StreamGraph`
			- 合并算子链得到 `JobGraph`
			- 向 `RM` 提交任务
		- `RM` 找一个 `NM` 启动 `AM`
		- `AM`
			- 启动 `JobManager`：`JobMaster`，`RM`
				- `JobMaster`：将 `JobGraph` 按并行度合并成 `ExecutionGraph`
				- `RM` 申请资源
				- 启动 `TaskManager`
		- `TaskManager`
			- 物理流图：提供 `Slot` 运行 `Task`
	- yarn-application: `StreamGraph` 和 `JobGraph` 在 `JobMaster` 生成
## 10.4 算子

![[process_funcs.svg]]

- `SQL -> Table API -> DataStream -> ProcessFunction`
- 分类
	- Source
	- Transform
		- 基本转换：`map, flatMap, filter`
		- 聚合：`max, maxBy, min, minBy, sum, reduce`
	- 重分区
		- keyBy
		- reblance：全局轮询
		- rescale: 
			- 只会将数据轮询发送到下游并行任务的一部分中
			- 分小团体，小团体内部轮流
		- forward：一对一，上下游并行度相同
		- shuffle：随机
		- broadcast：广播
		- global：全部发往下游第一个分区
	- Sink
- 如何将一个流变为两个流 -- 侧流
	- `getSideOutput()`
	- 侧流输出：`ctx.output(OutputTag x, T t)`
	- 主流输出：`Collector.collect()`
## 10.5 时间语义与窗口操作
- 时间语义：事件、进入、处理
- 事件时间：`Watermark`
	- 本质：流中传输的特殊时间戳，用于衡量事件进行的机制
	- 作用：处理乱序数据
	- 作用机制：延迟关窗
	- 传输：递增(见 4.2)、广播、短板(见 4.5)
	- 生成策略：
		- 周期型：默认 200ms
			- 数据密集时，周期型的好
			- 数据稀疏，产生的水印不会发送
		- 断点式
			- 数据稀疏时，断点式好
			- 数据密集时会产生许多的水印数据
		- 考虑到流中存在数据稀疏和数据密集的情况，周期型更好
- 窗口操作
	- 分类
		- 时间：滚动、滑动、会话
		- 计数：滚动、滑动
	- 核心组件：
		- windowAssigner：
			- `window`
			- `windowAll`
		- 触发器:
			- 凌晨没有数据，按照事件时间开窗，最后一个窗口到第二天才会关闭，及时关闭如何处理
				- 自定义 Trigger
				- 不属于迟到的数据注册两个定时器(事件，处理)
					- `watermark` 不更新，按处理时间触发
					- 一个定时器触发后关闭另一个定时器
			- 开一个 10s 的滚动时间窗口，希望每 2s 输出一次聚合结果
			- 只有 `KeyedStream` 才可以使用定时器
		- 移除器
		- 事件时间
			- 允许迟到：关窗前来一条数据计算一次
			- 侧输出流：当数据所属的所有窗口均已关闭，数据会进侧输出流
		- 窗口聚合函数
			- 增量
				- 效率高
			- 全量 `process WinowFunction`
				- 特殊需求：百分比指标
				- 可以获取窗口信息
			- 结合使用
## 10.6 状态编程与容错
### 10.6.1 状态编程
- 状态：历史数据
	- OperateState
		- `List`
		- `UnionList`
		- `BroadcastState`
	- KeyedState
		- `Value`
		- `List`
		- `Map`
		- `Reduce`
		- `Aggregate`
	- 使用在 `RichFunction` 中，可以设置 TTL
### 10.6.2 容错
- Checkpoint / Savepoint
	- Barrier 对齐
		- 精确一次
			- 有 `Barrier` 到达后
				- `Barrier` 已到达的 `Task` 在收到数据时进行缓存数据
				- `Barrier` 未到达的 `Task` 来数据时，正常计算输出
			- 所有的 `Barrier` 均到达后将计算结果(状态)保存
			- 将缓存的数据依次处理并输出
		- 至少一次
			- 有 `Barrier` 到达后
				- `Barrier` 已到达的 `Task` 在收到数据时继续计算，不缓存
				- 挂掉之后会重复读
				- `Barrier` 未到达的 `Task` 来数据时，正常计算输出
			- 所有的 `Barrier` 均到达后将状态保存
	- 非 Barrier 对齐
		- 有 `Barrier` 到达后，会将当前计算结果保存，`ck` 为未完成状态
			- `Barrier` 已到达的 `Task` 收到数据时，正常计算输出
			- `Barrier` 未到达的 `Task` 来数据时，正常计算输出，同时将数据本身进行缓存
		- 所有 `Barrier` 都到达后，`ck` 标记完成
		- 缺点：会造成状态过大
	- 端到端精准一次
		- Source：可重复读取数据
		- Flink：精准一次 `checkpoint`
		- Sink：
			- 幂等
			- 事务 2PC
## 10.7 FlinkSQL
- 事件时间，处理时间的提取
	- 建表时创建
		- `pc as proctime()`
		- `et as to_timestamp_ltz(ts, 3) (ts, 3) -> ms (ts, 0) -> s`
	- 流转表
- 窗口
	- TVF: 
		- 优化
		- 多维分析函数
		- 累计窗口
	- GroupWindow：会话窗口
- 自定义函数
	- UDF
	- UDAF
	- UDTF
	- UDTAF: 只用于 `TableAPI`
- Join
	- 常规 Join：内连接，外连接
		- 左外连接：TTL = 10s
			- 左表先来数据，10s 内右表来数据
				- \[+I]    左    null
				- \[-D]    左   null
				- \[+I]    左    右
			- 左表先来数据，10s 后右表来数据
				- \[+I]    左    null
			- 左表先来数据，10s 内右表来数据，之后保持间隔 10s 内来数据
				- \[+I]    左    null
				- \[-D]    左    null
				- \[+I]    左    右
					- 左表 `ttl` 的类型是 `OnReadAndWrite`, 读一次后 `ttl` 的时间会刷新
				- 后续会保持\[+I] 左右
			- 左表先来数据，10s 内右表来数据，之后超过 10s 来数据
				- \[+I]    左    null
				- \[-D]    左    null
				- \[+I]    左    右
				- 之后不打印任何数据
			- 右表先来数据，10s 内左表来数据
				- \[+I]    左    右
			- 右表先来数据，10s 后左表来数据
				- \[+I]    左    null
			- 右表先来数据，10s 内左表来数据，之后保持间隔 10s 内来数据
				- 10s 内：\[+I]    左    右
				- 10s 后：\[+I]    左    null
					- 右表 `ttl` 类型是 `OnCreateAndWrite`，读取并不会刷新 `ttl` 时间
			- 右表先来数据，10s 内左表来数据，之后超过 10s 来数据
				- 同上
	- 时序 Join：事件、处理
	- LookupJoin：时间语义下特殊的时序 `Join`
		- Cache: 维表数据不更新时使用
	- IntervalJoin