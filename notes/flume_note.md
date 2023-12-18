# 1. 启动
## 1.1 flume-ng
```shell
# agent主类
FLUME_AGENT_CLASS="org.apache.flume.node.Application"
run_flume() {
  # 执行java -cp $FLUME_CLASSPATH
  $EXEC $JAVA_HOME/bin/java $JAVA_OPTS $FLUME_JAVA_OPTS "${arr_java_props[@]}" -cp "$FLUME_CLASSPATH" \
      -Djava.library.path=$FLUME_JAVA_LIBRARY_PATH "$FLUME_APPLICATION_CLASS" $*
}

# flume-ng agent
case "$mode" in
  agent)
    opt_agent=1
    ;;
    ...
esac

# 最终的执行逻辑
if [ -n "$opt_agent" ] ; then
  run_flume $FLUME_AGENT_CLASS $args
exit 0
```
## 1.2 创建 Application
```java
public static void main(String[] args) {  
  try {  
    // Options对象底层维护了LinkedHashMap和ArrayList
    Options options = new Options();
    // 将参数封装在Option对象后添加到Options对象中
	Option option = new Option("n", "name", true, "the name of this agent");
	option.setRequired(true);
	options.addOption(option);

	option = new Option("f", "conf-file", true,
  "specify a config file (required if -c, -u, and -z are missing)");

	DefaultParser parser = new DefaultParser();  
	// 解析参数
	CommandLine commandLine = parser.parse(options, args, initProps);
	Application application;
	// 初始化组件
	application.handleConfigurationEvent(configurationProvider.getConfiguration());
	application.start();
	}
} 
```
## 1.3 从配置中获取组件相关配置
```java
public MaterializedConfiguration getConfiguration() {  
  // 获取 flume 的配置
  FlumeConfiguration fconfig = getFlumeConfiguration();  
  // 从 fconfig 中获取 agent 的配置
  AgentConfiguration agentConf = fconfig.getConfigurationFor(getAgentName());  
  if (agentConf != null) {  
    // 创建三个 Map 用于存放 Source，Channel 和 Sink 的参数
    Map<String, ChannelComponent> channelComponentMap = Maps.newHashMap();  
    Map<String, SourceRunner> sourceRunnerMap = Maps.newHashMap();  
    Map<String, SinkRunner> sinkRunnerMap = Maps.newHashMap();  
    try {  
      // 创建 channel，存入 channelComponentMap 中
      loadChannels(agentConf, channelComponentMap);  
      // 读取配置文件生成 source , 然后创建 sourceRunner, 并注册到 channel
      loadSources(agentConf, channelComponentMap, sourceRunnerMap); 
      // 创建sink，并存入 sinkRunnerMap 中，并注册到 channel
      loadSinks(agentConf, channelComponentMap, sinkRunnerMap);  
      Set<String> channelNames = new HashSet<String>(channelComponentMap.keySet());  
      for (String channelName : channelNames) {  
        ChannelComponent channelComponent = channelComponentMap.get(channelName);  
        if (channelComponent.components.isEmpty()) {...} 
        else {   
          conf.addChannel(channelName, channelComponent.channel);  
        }  
      }  
      for (Map.Entry<String, SourceRunner> entry : sourceRunnerMap.entrySet()) { 
        // SourceRunner 有 PollableSourceRunner 和 EventDrivenSourceRunner 两个子类
        conf.addSourceRunner(entry.getKey(), entry.getValue());  
      }  
      for (Map.Entry<String, SinkRunner> entry : sinkRunnerMap.entrySet()) {  
        // SinkRunner 中会创建 PollingRunner
        conf.addSinkRunner(entry.getKey(), entry.getValue());  
      }  
    } 
  }  
  return conf;  
}
```
## 1.4 Channel 的初始化
```java
private void loadChannels(AgentConfiguration agentConf,  
    Map<String, ChannelComponent> channelComponentMap)  
        throws InstantiationException {  
        
  // channel 名
  Set<String> channelNames = agentConf.getChannelSet();  
  Map<String, ComponentConfiguration> compMap = agentConf.getChannelConfigMap();  
  /*  
   * Components which have a ComponentConfiguration object   */  
   for (String chName : channelNames) {  
    ComponentConfiguration comp = compMap.get(chName);  
    if (comp != null) {  
      // 创建 channel
      Channel channel = getOrCreateChannel(channelsNotReused,  
          comp.getComponentName(), comp.getType());  
      try {  
        Configurables.configure(channel, comp); 
        // 添加到 map 中 
        channelComponentMap.put(comp.getComponentName(),  
            new ChannelComponent(channel));  
      }  
    }  
  }  
}

private Channel getOrCreateChannel(  
    ListMultimap<Class<? extends Channel>, String> channelsNotReused,  
    String name, String type)  
    throws FlumeException {  

  Class<? extends Channel> channelClass = channelFactory.getClass(type);   
 
  Map<String, Channel> channelMap = channelCache.get(channelClass);  
  Channel channel = channelMap.get(name);  
  if (channel == null) {
    // 通过反射创建 channel  
    channel = channelFactory.create(name, type);  
    channel.setName(name);  
    channelMap.put(name, channel);  
  }
  return channel;  
}
```
## 1.5 Source 的初始化
```java
private void loadSources(AgentConfiguration agentConf,  
    Map<String, ChannelComponent> channelComponentMap,  
    Map<String, SourceRunner> sourceRunnerMap)  
    throws InstantiationException {  
  
  Set<String> sourceNames = agentConf.getSourceSet();  
  Map<String, ComponentConfiguration> compMap =  
      agentConf.getSourceConfigMap();  

   for (String sourceName : sourceNames) {  
    ComponentConfiguration comp = compMap.get(sourceName);  
    if (comp != null) {  
      SourceConfiguration config = (SourceConfiguration) comp;  
  
      Source source = sourceFactory.create(comp.getComponentName(),  
          comp.getType());  
      try {  
        Configurables.configure(source, config);  
        Set<String> channelNames = config.getChannels();  
        List<Channel> sourceChannels =  
                getSourceChannels(channelComponentMap, source, channelNames);  
        if (sourceChannels.isEmpty()) {...}
         // 为 source 配置 channel 选择器
         // Channel 选择器决定 Source 接收的一个特定事件写入哪些 Channel 中,它将选择的结果告知 Channel 处理器,然后由 Channel 处理器将event 写入指定的 Channel
        ChannelSelectorConfiguration selectorConfig =  
            config.getSelectorConfiguration();  

        ChannelSelector selector = ChannelSelectorFactory.create(  
            sourceChannels, selectorConfig);  
  
        ChannelProcessor channelProcessor = new ChannelProcessor(selector);  
        Configurables.configure(channelProcessor, config);  
  
        source.setChannelProcessor(channelProcessor);  
        sourceRunnerMap.put(comp.getComponentName(),  
            SourceRunner.forSource(source));  
        for (Channel channel : sourceChannels) {  
          ChannelComponent channelComponent =  

          channelComponent.components.add(sourceName);  
        }  
      } 
    }  
  }
}
```
## 1.6 Sink 的初始化
```java
private void loadSinks(AgentConfiguration agentConf,  
    Map<String, ChannelComponent> channelComponentMap, Map<String, SinkRunner> sinkRunnerMap)  
    throws InstantiationException {  
    
  Set<String> sinkNames = agentConf.getSinkSet();  
  Map<String, ComponentConfiguration> compMap =  
      agentConf.getSinkConfigMap();  
  Map<String, Sink> sinks = new HashMap<String, Sink>();  
  /*  
   * Components which have a ComponentConfiguration object   */  
   for (String sinkName : sinkNames) {  
    ComponentConfiguration comp = compMap.get(sinkName);  
    if (comp != null) {  
      SinkConfiguration config = (SinkConfiguration) comp;  
      Sink sink = sinkFactory.create(comp.getComponentName(), comp.getType());  
      try {  
        Configurables.configure(sink, config);  
        ChannelComponent channelComponent = channelComponentMap.get(config.getChannel());  
        // 检查 sink 和 channel 的兼容性
        checkSinkChannelCompatibility(sink, channelComponent.channel);  
        sink.setChannel(channelComponent.channel);  
        sinks.put(comp.getComponentName(), sink);  
        channelComponent.components.add(sinkName);  
      }  
    }  
  }
}

private void checkSinkChannelCompatibility(Sink sink, Channel channel)
    throws InstantiationException {
    // 开启事务
    if (sink instanceof BatchSizeSupported && channel instanceof TransactionCapacitySupported) {
        // FileChannel和MemoryChannel都实现了getTransactionCapacity方法
        long transCap = ((TransactionCapacitySupported) channel).getTransactionCapacity();
        // 调用不同sink的getBatchSize方法
        long batchSize = ((BatchSizeSupported) sink).getBatchSize();
        // 事务大小不能小于批大小
        if (transCap < batchSize) {...}
    }
}
```
## 1.7 组件的运行
```java
public void handleConfigurationEvent(MaterializedConfiguration conf) { 
  try {  
    // 先停止之前的组件
    stopAllComponents();  
    // 初始化所有组件
    initializeAllComponents(conf);  
    // 启动组件
    startAllComponents(conf);  
  }
}

private void startAllComponents(MaterializedConfiguration materializedConfiguration) {  

  // channel start
  for (Entry<String, Channel> entry :  
      materializedConfiguration.getChannels().entrySet()) {  
    try {  
      logger.info("Starting Channel " + entry.getKey());  
      supervisor.supervise(entry.getValue(),  
          new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);  
    } 
  }  
  // sink start
  for (Entry<String, SinkRunner> entry : materializedConfiguration.getSinkRunners().entrySet()) {  
    try {  
      logger.info("Starting Sink " + entry.getKey());  
      supervisor.supervise(entry.getValue(),  
          new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);  
    } 
  }  
  // source start
  for (Entry<String, SourceRunner> entry :  
       materializedConfiguration.getSourceRunners().entrySet()) {  
    try {  
      logger.info("Starting Source " + entry.getKey());  
      supervisor.supervise(entry.getValue(),  
          new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);  
    } 
  }  
  this.loadMonitoring();  
}

// application.start()
public void start() {  
  lifecycleLock.lock();  
  try {  
    for (LifecycleAware component : components) {  
      supervisor.supervise(component,  
          new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);  
    }  
  }
}

public synchronized void supervise(LifecycleAware lifecycleAware,  
    SupervisorPolicy policy, LifecycleState desiredState) {    
  // monitorService 是一个线程池用于执行程序, 调用对应的 runner 来启动组件
  ScheduledFuture<?> future = monitorService.scheduleWithFixedDelay(  
      monitorRunnable, 0, 3, TimeUnit.SECONDS);  
  monitorFutures.put(lifecycleAware, future);  
}
```

# 复习

## .1 基本组成
- `Source`
	- `TailDirSource`
	- `KafkaSource`
	- `AvroSource`：用于 `Flume` 之间的直接对接，构建 `Flume` 复杂拓扑结构
- `Channel`
	- `MemoryChannel`
	- `FileChannel`
	- `KafkaChannel`
- `Sink`
	- `HDFSSink`