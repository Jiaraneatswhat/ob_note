# 1. Producer

-  `Producer` 在生产过程中主要使用以下组件：`ProducerMetadata` 用于 `Producer` 与 `Broker` 通讯获取元数据，`ProducerConfig` 用于设置 `Producer` 的参数，`NetworkClient` 用于网络通信，`RecordAccumlator` 用于缓存 `Batch`，`Sender` 用于发送消息，`ProducerInterceptors` 用于处理拦截器逻辑(可选)

![[producer_components.svg]]

## 1.1 Producer 的属性
```java
public class KafkaProducer<K, V> implements Producer<K, V> {

    private final Logger log;
    private static final String JMX_PREFIX = "kafka.producer";
    public static final String NETWORK_THREAD_PREFIX = "kafka-producer-network-thread";
    public static final String PRODUCER_METRIC_GROUP_NAME = "producer-metrics";

    private final String clientId;
    // Visible for testing
    final Metrics metrics;
    private final KafkaProducerMetrics producerMetrics;
    // 分区器
    private final Partitioner partitioner;
    private final int maxRequestSize;
    private final long totalMemorySize;
    // 元数据
    private final ProducerMetadata metadata;
    // RecordAccumulator
    private final RecordAccumulator accumulator;
    // Sender
    private final Sender sender;
    // 运行Sender的线程
    private final Thread ioThread;
    private final CompressionType compressionType;
    private final Sensor errors;
    private final Time time;
    private final Serializer<K> keySerializer;
    private final Serializer<V> valueSerializer;
    // 生产端的配置
    private final ProducerConfig producerConfig;
    private final long maxBlockTimeMs;
    private final boolean partitionerIgnoreKeys;
    private final ProducerInterceptors<K, V> interceptors;
    private final ApiVersions apiVersions;
    private final TransactionManager transactionManager;
}
```
## 1.2 Producer 的初始化
### 1.2.1 脚本
```shell
# kafka-console-producer.sh
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
# 通过kafka-run-ckass.sh 运行ConsoleProducer
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsoleProducer "$@"

# kafka-run-ckass.sh
# 根据命令中是否有-daemon决定是否挂起
# Launch mode
if [ "x$DAEMON_MODE" = "xtrue" ]; then
  nohup $JAVA $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_GC_LOG_OPTS $KAFKA_JMX_OPTS $KAFKA_LOG4J_OPTS -cp $CLASSPATH $KAFKA_OPTS "$@" > "$CONSOLE_OUTPUT_FILE" 2>&1 < /dev/null &
else
  exec $JAVA $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_GC_LOG_OPTS $KAFKA_JMX_OPTS $KAFKA_LOG4J_OPTS -cp $CLASSPATH $KAFKA_OPTS "$@"
fi
```
### 1.2.2 ConsoleProducer.main()
```java
  def main(args: Array[String]): Unit = {

    try {
        val config = new ProducerConfig(args)
        val reader = Class.forName(config.readerClass).getDeclaredConstructor().newInstance().asInstanceOf[MessageReader]
        reader.init(System.in, getReaderProps(config))

        val producer = new KafkaProducer[Array[Byte], Array[Byte]](producerProps(config))

    Exit.addShutdownHook("producer-shutdown-hook", producer.close)
        var record: ProducerRecord[Array[Byte], Array[Byte]] = null
        do {
          record = reader.readMessage()
          if (record != null)
            // 通过LineMessageReader读取record后发送
            send(producer, record, config.sync)
        } while (record != null)
    } 
}
```
### 1.2.3 new KafkaProducer()
```java
// 传入Properties时调用，将KeySerializer和ValueSerializer设置为null，并调用下一个构造器
public KafkaProducer(Properties properties) {
        this(properties, null, null);
}

// Utils.propsToMap从properties中获取属性并返回一个HashMap对象，将Hashtable转为HashMap
public KafkaProducer(Properties properties, Serializer<K> keySerializer, Serializer<V> valueSerializer) {
    this(Utils.propsToMap(properties), keySerializer, valueSerializer);
}

public KafkaProducer(Map<String, Object> configs, Serializer<K> keySerializer, Serializer<V> valueSerializer) {
    // 通过ProducerConfig对象来解析配置
    // 将KV序列化器的配置添加到config中
    this(new ProducerConfig(ProducerConfig.appendSerializerToConfig(configs, keySerializer, valueSerializer)),
         keySerializer, valueSerializer, null, null, null, Time.SYSTEM);
 }

// 创建keySerializer，ValueSerializer，处理拦截器，创建RecordAccumulator对象等初始化工作，最后创建Sender对象，调用doSend()
KafkaProducer(ProducerConfig config,
                  Serializer<K> keySerializer,
                  Serializer<V> valueSerializer,
                  ProducerMetadata metadata,
                  KafkaClient kafkaClient,
                  ProducerInterceptors<K, V> interceptors,
                  Time time) {
        try {
            this.producerConfig = config;
            // 获取ClientID
            String clientId = config.getString(ProducerConfig.CLIENT_ID_CONFIG);
            // 获取事务ID
            String transactionalId = userProvidedConfigs.containsKey(ProducerConfig.TRANSACTIONAL_ID_CONFIG));
            // 这里可以根据配置的 class 选择分区使用的算法
            this.partitioner = config.getConfiguredInstance(
                    ProducerConfig.PARTITIONER_CLASS_CONFIG,
                    Partitioner.class,
                    Collections.singletonMap(ProducerConfig.CLIENT_ID_CONFIG, clientId));
            // retry.backoff.ms 默认100L
            long retryBackoffMs = config.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG);
            if (keySerializer == null) {
            // 序列化key
			this.keySerializer = config.getConfiguredInstance(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,Serializer.class);}
                // 创建一个空的拦截器List
            this.interceptors = new ProducerInterceptors<>(interceptorList);
            ClusterResourceListeners clusterResourceListeners = configureClusterResourceListeners(keySerializer,
                    valueSerializer, interceptorList, reporters);
            this.maxRequestSize = config.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG); // 最大请求size:1048576
            // RecordAccumulator缓存的大小
            // 32 * 1024 * 1024L -> 32M
            this.totalMemorySize = config.getLong(ProducerConfig.BUFFER_MEMORY_CONFIG);)
            this.compressionType = CompressionType.forName(config.getString(ProducerConfig.COMPRESSION_TYPE_CONFIG));// 默认压缩类型none
            // 创建PartitionerConfig对象，对默认的分区器进行设置
            RecordAccumulator.PartitionerConfig partitionerConfig = new RecordAccumulator.PartitionerConfig(
                enableAdaptivePartitioning,
config.getLong(ProducerConfig.PARTITIONER_AVAILABILITY_TIMEOUT_MS_CONFIG)
            );
            // 默认开启了ENABLE_IDEMPOTENCE_CONFIG，会创建一个transactionManager
            this.transactionManager = configureTransactionState(config, logContext);
            // 创建RecordAccumulator对象用于缓存
            this.accumulator = new RecordAccumulator(logContext,
                    config.getInt(ProducerConfig.BATCH_SIZE_CONFIG), 
                    this.compressionType, 
                    // 计算等待时间 min(config.getLong(
                    // ProducerConfig.LINGER_MS_CONFIG), Integer.MAX_VALUE);}	
                    lingerMs(config),
                    retryBackoffMs, // 重试的回退时间
                    deliveryTimeoutMs,
                    partitionerConfig,// 传入上面创建的partitionerConfig对象
                    metrics,
                    PRODUCER_METRIC_GROUP_NAME,
                    time,
                    apiVersions,
                    transactionManager,
                    // 新建一个缓冲池                        
                    new BufferPool(this.totalMemorySize, config.getInt(ProducerConfig.BATCH_SIZE_CONFIG), metrics, time, PRODUCER_METRIC_GROUP_NAME));
            // 获取bootstrap-server地址
            List<InetSocketAddress> addresses = ClientUtils.parseAndValidateAddresses(
    config.getList(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG),
    config.getString(ProducerConfig.CLIENT_DNS_LOOKUP_CONFIG));
            // 生产者元数据对象初始化
            if (metadata != null) {
                this.metadata = metadata;
            } else {
                this.metadata = new ProducerMetadata(retryBackoffMs,
                        config.getLong(ProducerConfig.METADATA_MAX_AGE_CONFIG),
                        config.getLong(ProducerConfig.METADATA_MAX_IDLE_CONFIG),
                        logContext,
                        clusterResourceListeners,
                        Time.SYSTEM);
                // 元数据初始化
                this.metadata.bootstrap(addresses);
            }
            this.errors = this.metrics.sensor("errors");
            // 创建Sender对象用于发送数据
            this.sender = newSender(logContext, kafkaClient, this.metadata);
            String ioThreadName = NETWORK_THREAD_PREFIX + " | " + clientId;
            // 创建并开启io线程
            this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
            this.ioThread.start();
            ...
        }
    }
```
#### 1.2.3.1 new RecordAccumulator()
```java
public class RecordAccumulator {
    private final int batchSize;
    private final CompressionType compression;
    private final int lingerMs;
    private final long retryBackoffMs;
    // 缓冲池
    private final BufferPool free;
	// TopicInfo是KafkaProducer的内部类，维护了一个ConcurrentMap，储存了topic的分区和储存ProducerBatch对象的双端队列
    private final ConcurrentMap<String /*topic*/, TopicInfo> topicInfoMap = new CopyOnWriteMap<>();
}
```
#### 1.2.3.2 new BufferPool()
```java
public BufferPool(long memory, int poolableSize, Metrics metrics, Time time, String metricGrpName) {
        this.poolableSize = poolableSize; // 当前BufferPool管理的单个ByteBuffer大小，通过RecordAccumulator传入batchSize，大小为16k 
        this.free = new ArrayDeque<>(); //ByteBuffer队列缓存固定大小的ByteBuffer对象
        this.waiters = new ArrayDeque<>(); // 申请不到足够空间而阻塞的线程对应的 Condition 对象
        this.totalMemory = memory; // 整个BufferPool总内存大小 默认32M
        this.nonPooledAvailableMemory = memory; // 非池化可用的内存即 totalMemory 减去 free 列表中的全部 ByteBuffer 的大小
}
```
#### 1.2.3.3 new ProducerMetadata()
```java
// Metadata有两个子类ProducerMetadata和ConsumerMetadata
public class Metadata implements Closeable {
    private final Logger log;
    // 两次请求更新元数据的最小间隔，默认100ms
    private final long refreshBackoffMs;
    // 多久更新一次元数据，默认5min
    private final long metadataExpireMs;
}

// 调用父类的bootstrap方法
public synchronized void bootstrap(List<InetSocketAddress> addresses) {
        this.needFullUpdate = true;
        this.updateVersion += 1;
        this.cache = MetadataCache.bootstrap(addresses);
}

static MetadataCache bootstrap(List<InetSocketAddress> addresses) {
        Map<Integer, Node> nodes = new HashMap<>();
        int nodeId = -1;
    	// 将地址转换为Kafka中的Node类型保存在Map中
        for (InetSocketAddress address : addresses) {
            nodes.put(nodeId, new Node(nodeId, address.getHostString(), address.getPort()));
            nodeId--;
        }
   		// 创建元数据缓存对象
    	// 调用Cluster的bootstrap方法
        return new MetadataCache(null, nodes, Collections.emptyList(),
                Collections.emptySet(), Collections.emptySet(),Collections.emptySet(),
                null, Collections.emptyMap(), Cluster.bootstrap(addresses));}

// 用List中的地址创建Cluster对象
public static Cluster bootstrap(List<InetSocketAddress> addresses) {
        List<Node> nodes = new ArrayList<>();
        int nodeId = -1;
        for (InetSocketAddress address : addresses)
            nodes.add(new Node(nodeId--, address.getHostString(), address.getPort()));
        return new Cluster(null, true, nodes, new ArrayList<>(0),
            Collections.emptySet(), Collections.emptySet(), Collections.emptySet(), null, Collections.emptyMap());
}

// Cluster保存了Kafka集群中的nodes，topics和partitions的信息
private Cluster(String clusterId,
                    boolean isBootstrapConfigured,
                	// kafka集群的节点
                    Collection<Node> nodes,
                    Collection<PartitionInfo> partitions,
                	// 内置的topic
                    Set<String> internalTopics,
                    Node controller,
                    Map<String, Uuid> topicIds) {
        // partition有副本
        this.partitionsByTopicPartition = Collections.unmodifiableMap(tmpPartitionsByTopicPartition);
        // 一个topic有哪些副本
        this.partitionsByTopic = Collections.unmodifiableMap(tmpPartitionsByTopic);
        this.availablePartitionsByTopic = Collections.unmodifiableMap(tmpAvailablePartitionsByTopic);
        this.partitionsByNode = Collections.unmodifiableMap(tmpPartitionsByNode);
        this.topicIds = Collections.unmodifiableMap(topicIds);
        Map<Uuid, String> tmpTopicNames = new HashMap<>();
        topicIds.forEach((key, value) -> tmpTopicNames.put(value, key));
        this.topicNames = Collections.unmodifiableMap(tmpTopicNames);
        this.controller = controller;
}

// Node类存储节点信息
public class Node {

    private static final Node NO_NODE = new Node(-1, "", -1);
    private final int id;
    private final String idString;
    private final String host;
    private final int port; // 9092
    private final String rack;
}  
```
#### 1.2.3.4 newSender()
```java
Sender newSender(LogContext logContext, KafkaClient kafkaClient, ProducerMetadata metadata) {
    	// 客户端在阻塞之前将在单个连接上发送的未确认请求的最大数量
        int maxInflightRequests = producerConfig.getInt(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION);
    	// 客户端等待请求响应的最长时间，默认30ms
        int requestTimeoutMs = producerConfig.getInt(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG);
    	// 发送消息的通道类型
        ChannelBuilder channelBuilder = ClientUtils.createChannelBuilder(producerConfig, time, logContext);
    	// 生产者监控指标对象创建
        ProducerMetrics metricsRegistry = new ProducerMetrics(this.metrics);
        Sensor throttleTimeSensor = Sender.throttleTimeSensor(metricsRegistry.senderMetrics);
    	// 创建NetworkClient对象用于异步请求/响应网络i/o，用于实现面向用户的生产者和消费者客户端
        KafkaClient client = kafkaClient != null ? kafkaClient : new NetworkClient(
            // nioSelector接口，用于执行非阻塞多连接网络I/O
            // CONNECTIONS_MAX_IDLE_MS_CONFIG默认9 * 60 * 1000
          new Selector(producerConfig.getLong(ProducerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG),
                        this.metrics, time, "producer", channelBuilder, logContext),
                metadata,
                clientId,
                maxInflightRequests,
            	// reconnect.backoff.ms 在尝试重新连接到给定主机之前等待的基本时间量
                producerConfig.getLong(ProducerConfig.RECONNECT_BACKOFF_MS_CONFIG),
            	// reconnect.backoff.max.ms 重新连接到反复连接失败的代理时等待的最长时间(ms)
                producerConfig.getLong(ProducerConfig.RECONNECT_BACKOFF_MAX_MS_CONFIG),
                producerConfig.getInt(ProducerConfig.SEND_BUFFER_CONFIG),
                producerConfig.getInt(ProducerConfig.RECEIVE_BUFFER_CONFIG),
                requestTimeoutMs,
                producerConfig.getLong(ProducerConfig.SOCKET_CONNECTION_SETUP_TIMEOUT_MS_CONFIG),
                producerConfig.getLong(ProducerConfig.SOCKET_CONNECTION_SETUP_TIMEOUT_MAX_MS_CONFIG),
                time,
                true,
                apiVersions,
                throttleTimeSensor,
                logContext);
		// 配置acks
        short acks = Short.parseShort(producerConfig.getString(ProducerConfig.ACKS_CONFIG));
    	// 返回Sender对象
        return new Sender(logContext,
                client,
                metadata,
                this.accumulator,
                maxInflightRequests == 1,
                // 请求的最大大小（以字节为单位）
                producerConfig.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG),
                acks, // 默认为all
                producerConfig.getInt(ProducerConfig.RETRIES_CONFIG),
                metricsRegistry.senderMetrics,
                time,
                requestTimeoutMs,
                producerConfig.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG),
                this.transactionManager,
                apiVersions);
}

public Selector(int maxReceiveSize,
            long connectionMaxIdleMs,
            int failedAuthenticationDelayMs,
            Metrics metrics,
            Time time,
            String metricGrpPrefix,
            Map<String, String> metricTags,
            boolean metricsPerConnection,
            boolean recordTimePerConnection,
            ChannelBuilder channelBuilder,
            MemoryPool memoryPool,
            LogContext logContext) {
        try {
            // 创建nioSelector对象
            this.nioSelector = java.nio.channels.Selector.open();
        } catch (IOException e) {
            throw new KafkaException(e);
        }
    	// 单次网络最大接收字节数量 
        this.maxReceiveSize = maxReceiveSize;
    	// 连接最大空闲时间 connections.max.idle.ms
        this.time = time;
        ...
}

public NetworkClient(MetadataUpdater metadataUpdater,
                     Metadata metadata,
                     Selectable selector,
                     String clientId,
                     int maxInFlightRequestsPerConnection,
                     long reconnectBackoffMs,
                     long reconnectBackoffMax,
                     int socketSendBuffer,
                     int socketReceiveBuffer,
                     int defaultRequestTimeoutMs,
                     long connectionSetupTimeoutMs,
                     long connectionSetupTimeoutMaxMs,
                     Time time,
                     boolean discoverBrokerVersions,
                     ApiVersions apiVersions,
                     Sensor throttleTimeSensor,
                     LogContext logContext,
                     HostResolver hostResolver) {
   //用于更新和检索集群元数据的工具类 创建
    if (metadataUpdater == null) {
        if (metadata == null)
            throw new IllegalArgumentException("`metadata` must not be null");
        this.metadataUpdater = new DefaultMetadataUpdater(metadata);
    } else {
        this.metadataUpdater = metadataUpdater;
    }
  //IO 选择器
    this.selector = selector;
  //客户端id
    this.clientId = clientId;
  //配置max.in.flight.requests.per.connection  默认配置是5个 客户端单个请求的最大未确认数量
    this.inFlightRequests = new InFlightRequests(maxInFlightRequestsPerConnection);
  //连接集群的状态
    this.connectionStates = new ClusterConnectionStates(
            reconnectBackoffMs, reconnectBackoffMax,
            connectionSetupTimeoutMs, connectionSetupTimeoutMaxMs, logContext, hostResolver);
  //对应配置send.buffer.bytes 这个是TCP发送数据的缓冲区大小，如果为-1将使用操作系统默认的值
    this.socketSendBuffer = socketSendBuffer;
  //对应配置receive.buffer.bytes 这个是TCP通信接收数据的缓冲区大小，如果为-1将使用操作系统默认值
    this.socketReceiveBuffer = socketReceiveBuffer;
  //当前发送server的相关id
    this.correlation = 0;
    this.randOffset = new Random();
  //默认请求超时时间30秒
    this.defaultRequestTimeoutMs = defaultRequestTimeoutMs;
  //对应配置reconnect.backoff.ms 重连回退时间  50毫秒
    this.reconnectBackoffMs = reconnectBackoffMs;
  //SystemTime
    this.time = time;
  //默认为true 是否发现broker的版本
  //发送请求之前会先发送个版本请求的ApiVersionRequest
    this.discoverBrokerVersions = discoverBrokerVersions;
  //ApiVersions类型 用来封装节点的API信息NodeApiVersions
    this.apiVersions = apiVersions;
  //吞吐量传感器
    this.throttleTimeSensor = throttleTimeSensor;
    this.log = logContext.logger(NetworkClient.class);
  //当前状态
    this.state = new AtomicReference<>(State.ACTIVE);
}
```
