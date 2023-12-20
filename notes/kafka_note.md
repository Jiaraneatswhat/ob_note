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
    	// 默认值5
        int maxInflightRequests = producerConfig.getInt(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION);
    	// 客户端等待请求响应的最长时间，默认30ms
        int requestTimeoutMs = producerConfig.getInt(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG);
    	// 发送消息的通道类型
        ChannelBuilder channelBuilder = ClientUtils.createChannelBuilder(producerConfig, time, logContext);
    	// 创建NetworkClient对象用于异步请求/响应网络i/o，用于实现面向用户的生产者和消费者客户端
        KafkaClient client = kafkaClient != null ? kafkaClient : new NetworkClient(
            // nioSelector接口，用于执行非阻塞多连接网络I/O
            // CONNECTIONS_MAX_IDLE_MS_CONFIG默认9 * 60 * 1000
          new Selector(producerConfig.getLong(ProducerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG), this.metrics, time, "producer", channelBuilder, logContext),
     metadata, clientId, maxInflightRequests,
// reconnect.backoff.ms 在尝试重新连接到给定主机之前等待的基本时间量         	  
producerConfig.getLong(ProducerConfig.RECONNECT_BACKOFF_MS_CONFIG),
// reconnect.backoff.max.ms 重新连接到反复连接失败的代理时等待的最长时间(ms)
);
		// 配置acks，默认all
        short acks = Short.parseShort(producerConfig.getString(ProducerConfig.ACKS_CONFIG));
    	// 返回Sender对象
        return new Sender(logContext,
                client,
                metadata,
                this.accumulator,
                maxInflightRequests == 1,
                // 请求的最大大小（以字节为单位）
 producerConfig.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG)...)
}

public NetworkClient(...) {
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
#### 1.2.3.5 run()
```java
@Override
    public void run() {
        log.debug("Starting Kafka producer I/O thread.");
        while (running) {
            try {
                runOnce();
	            }
            }
        }
}

// 每执行一次就会去RecordAccumulator中拉取消息
void runOnce() {
    if (transactionManager != null) {
        try {
            transactionManager.maybeResolveSequences();

            // do not continue sending if the transaction manager is in a failed state
            if (transactionManager.hasFatalError()) {...}
			// 判断有是否正在进行的请求
            if (maybeSendAndPollTransactionalRequest()) {
                return;
            }
        }
    }
    long pollTimeout = sendProducerData(currentTimeMs);
    client.poll(pollTimeout, currentTimeMs);
}
// sender线程启动后，阻塞在底层Selector的select方法
```
## 1.3 发送
### 1.3.1 send()
```java
public Future<RecordMetadata> send(ProducerRecord<K, V> record) {
        return send(record, null);
}

public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
        ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
        return doSend(interceptedRecord, callback);
}

private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
            try {
                // 获取metadata
                // maxBlockTimeMs最大等待时间，未获取到元数据前main线程会阻塞在这里
                clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);
            } 
    		// 最大等待时间 - 拉去元数据花费时间
            long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
    		//从clusterAndWaitTime中获取Cluster对象
            Cluster cluster = clusterAndWaitTime.cluster;
            byte[] serializedKey;
            // 序列化k, v
			// 计算分区
            int partition = partition(record, serializedKey, serializedValue, cluster);
    		int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(), compressionType, serializedKey, serializedValue, headers);
    // 判断消息大小有没有超过maxRequestSize和totalMemorySize        
    ensureValidRecordSize(serializedSize);
            // 将消息发送到RecordAccumulator
	RecordAccumulator.RecordAppendResult result = accumulator.append(record.topic(), partition, timestamp, serializedKey,
                    serializedValue, headers, appendCallbacks, remainingWaitMs, abortOnNewBatch, nowMs, cluster);
            // 处理事务
            if (transactionManager != null) {
transactionManager.maybeAddPartition(appendCallbacks.topicPartition());
            }
			// batch满了或是新batch，唤醒sender
            if (result.batchIsFull || result.newBatchCreated) {
                log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), appendCallbacks.getPartition());
                this.sender.wakeup();
            }
            return result.future;
        }
}	
```
### 1.3.2 waitOnMetadata()
```java
private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long nowMs, long maxWaitMs) throws InterruptedException {
    	// 从缓存中获取Cluster对象
        Cluster cluster = metadata.fetch();
        metadata.add(topic, nowMs);
        do {
            metadata.add(topic, nowMs + elapsed);
            int version = metadata.requestUpdateForTopic(topic);
            // 申请元数据的请求要放在inflightRequests中通过sender去发送，这里需要唤醒sender线程，Sender再去唤醒NetworkClient线程调用poll方法获取元数据
            sender.wakeup();
            try {
                // main线程阻塞在此方法中等待sender线程
                //time.waitObject()
                metadata.awaitUpdate(version, remainingWaitMs);
            }
        return new ClusterAndWaitTime(cluster, elapsed);
}
```
### 1.3.3 poll()
```java
public List<ClientResponse> poll(long timeout, long now) {
        ensureActive();

		// 封装请求
        long metadataTimeout = metadataUpdater.maybeUpdate(now);
        try {
            // 调用selector的poll方法，进行IO操作
            this.selector.poll(Utils.min(timeout, metadataTimeout, defaultRequestTimeoutMs));
        } catch (IOException e) {
            log.error("Unexpected error during I/O", e);
        }

        // process completed actions
        long updatedNow = this.time.milliseconds();
        List<ClientResponse> responses = new ArrayList<>();
    	// 处理返回的响应
        handleCompletedSends(responses, updatedNow);
        handleCompletedReceives(responses, updatedNow);
        handleDisconnections(responses, updatedNow);
        handleConnections();
        handleInitiateApiVersionRequests(updatedNow);
        handleTimedOutConnections(responses, updatedNow);
        handleTimedOutRequests(responses, updatedNow);
        completeResponses(responses);
        return responses;
}

public long maybeUpdate(long now) {
            // should we update our metadata?
    		// 获取下一次更新的时间，needUpdate=true时返回0，立即更新
            long timeToNextMetadataUpdate = metadata.timeToNextUpdate(now);
            long waitForMetadataFetch = hasFetchInProgress() ? defaultRequestTimeoutMs : 0;

            long metadataTimeout = Math.max(timeToNextMetadataUpdate, waitForMetadataFetch);
    		// 大于0说明暂时不用更新
            if (metadataTimeout > 0) {
                return metadataTimeout;
            }
    		// 最小负载节点
            Node node = leastLoadedNode(now);
            if (node == null) {
                log.debug("Give up sending metadata request since no node is available");
                return reconnectBackoffMs;
            }
            return maybeUpdate(now, node);
        }

private long maybeUpdate(long now, Node node) {
	if (canSendRequest(nodeConnectionId, now)) {
		Metadata.MetadataRequestAndVersion requestAndVersion = metadata.newMetadataRequestAndVersion(now);
		// 创建MetadataRequest
		MetadataRequest.Builder metadataRequest = requestAndVersion.requestBuilder;
		// 发送请求
		sendInternalMetadataRequest(metadataRequest, nodeConnectionId, now);
		return defaultRequestTimeoutMs;
	}
}

void sendInternalMetadataRequest(MetadataRequest.Builder builder, String nodeConnectionId, long now) {
        ClientRequest clientRequest = newClientRequest(nodeConnectionId, builder, now, true);
        doSend(clientRequest, true, now);
}

private void doSend(ClientRequest clientRequest, boolean isInternalRequest, long now) {
     doSend(clientRequest, isInternalRequest, now, builder.build(version));
    }
}

private void doSend(ClientRequest clientRequest, boolean isInternalRequest, long now, AbstractRequest request) {
        String destination = clientRequest.destination();
        RequestHeader header = clientRequest.makeHeader(request.version());
        Send send = request.toSend(header);
        InFlightRequest inFlightRequest = new InFlightRequest(
                clientRequest,
                header,
                isInternalRequest,
                request,
                send,
                now);
    	// 将元数据请求放在inFlightRequests中
        this.inFlightRequests.add(inFlightRequest);
        selector.send(new NetworkSend(clientRequest.destination(), send));
}
```
### 1.3.4 handleCompletedReceives()
```java
private void handleCompletedReceives(List<ClientResponse> responses, long now) {
	// 请求元数据的响应
	if (req.isInternalRequest && response instanceof MetadataResponse)
		metadataUpdater.handleSuccessfulResponse(req.header, now, (MetadataResponse) response);
    }
}

public void handleSuccessfulResponse(RequestHeader requestHeader, long now, MetadataResponse response) {

	if (response.brokers().isEmpty()) {}
	else {
		// 获得到含有元数据的响应
		this.metadata.update(inProgress.requestVersion, response, inProgress.isPartialUpdate, now);
        }
}
```
### 1.3.5 partition()
```java
// 默认没有分区器
private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
        if (record.partition() != null)
            return record.partition();

        if (partitioner != null) {
            // 给定partitioner的话通过指定的分区器来分区
            int customPartition = partitioner.partition(
                record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
            return customPartition;
        }

        if (serializedKey != null && !partitionerIgnoreKeys) {
            // hash the keyBytes to choose a partition
            // 用内置分区器分区，底层使用murmur2 hash算法
            return BuiltInPartitioner.partitionForKey(serializedKey, cluster.partitionsForTopic(record.topic()).size());
        } else {
            return RecordMetadata.UNKNOWN_PARTITION;
        }
}
```
### 1.3.6 数据发送到 RecordAccumulator
#### 1.3.6.1 RecordAccumulator 的缓冲池
##### 1.3.6.1.1 缓冲池属性
```java
static final String WAIT_TIME_SENSOR_NAME = "bufferpool-wait-time";
private final long totalMemory;// 整个BufferPool总内存大小 默认32M
private final int poolableSize;// 当前BufferPool管理的单个ByteBuffer大小，通过RecordAccumulator传入batchSize，大小为16k 
private final ReentrantLock lock;
private final Deque<ByteBuffer> free; //ByteBuffer队列缓存固定大小的ByteBuffer对象
private final Deque<Condition> waiters;// 申请不到足够空间而阻塞的线程对应的 Condition 对象
private long nonPooledAvailableMemory; // 非池化可用的内存即 totalMemory 减去 free 列表中的全部 ByteBuffer 的大小
private final Metrics metrics;
private final Time time;
private final Sensor waitTime;
private boolean closed;
```
##### 1.3.6.1.2 allocate()

- **1. 申请的大小为 16kb 且 free 缓存池有缓存可用**

![[bufferpool_allocate1.svg]]
- 从 <font color='red'>free</font> 缓存池的队首拿出一个 <font color='red'>16kb</font> 的 <font color='red'>ByteBuffer</font> 来直接使用，等到 <font color='red'>ByteBuffer</font> 用完之后，通过 <font color='red'>clear()</font> 方法放入 <font color='red'>free</font> 缓存池的尾部，随后唤醒下一个等待分配内存的线程

- **2. 申请的大小为 16kb 且 free 缓存池无缓存可用**

![[bufferpool_allocate2.svg]]

- 此时 free 缓存池无可用内存，从<b>可用内存中获取16kb 内存来分配</b>，用完将<font color='red'>ByteBuffer</font> 添加到 free 缓存池的队尾中，并调用 clear()方法清空数据

- **3. 申请的大小非16kb 且 free 缓存池无可用的内存**

![[bufferpool_allocate3.svg]]

- 从<b>非池化可用内存中获取一部分内存来分配</b>，用完后将申请到的内存空间释放到非池化可用内存中，后序 GC 会进行处理

- **4. 申请非 16k 且 free 缓存池有可用内存，但非池化可用内存不够**

![[bufferpool_allocate4.svg]]

- free 缓存池有可用内存，但<b>申请的是非16kb</b>，先尝试从**free 缓存池中将 ByteBuffer 释放到非池化可用内存中，直到满足申请内存大小(size)，然后从可用内存中获取对应内存大小来分配，用完后将申请到的空间释放到非池化可用内存中，由 GC 进行后续处理**

```java
public ByteBuffer allocate(int size, long maxTimeToBlockMs) throws InterruptedException {
        if (size > this.totalMemory)
            throw new IllegalArgumentException(
            "Attempt to allocate " + size + " bytes, but there is a hard limit of " + this.totalMemory + " on memory allocations.");
        ByteBuffer buffer = null;
        this.lock.lock(); //重入锁加锁
        if (this.closed) {
            this.lock.unlock();
            throw new KafkaException("Producer closed while allocating memory");
        }
        try {
            // check if we have a free buffer of the right size pooled
            //需要分配内存的大小等于poolableSize，且队列不为空，则从free队列取出一个ByteBuffer(case1)
            if (size == poolableSize && !this.free.isEmpty())
                return this.free.pollFirst();
            //free队列的大小
            int freeListSize = freeSize() * this.poolableSize;
            //有足够的内存为size分配，则通过freeUp来分配给availableMemory
            if (this.nonPooledAvailableMemory + freeListSize >= size) {
                freeUp(size);
                this.nonPooledAvailableMemory -= size;
            } else {
                // 当前BufferPool不够提供申请内存大小，则需要阻塞当前线程
                int accumulated = 0;
                Condition moreMemory = this.lock.newCondition();
                try {
                    long remainingTimeToBlockNs = TimeUnit.MILLISECONDS.toNanos(maxTimeToBlockMs);
                    // 把自己添加到等待队列中末尾，保持公平性，先来的先获取内存，防止饥饿
                    this.waiters.addLast(moreMemory);
                    // 循环等待到分配成功或超时
                    while (accumulated < size) {
                        long startWaitNs = time.nanoseconds();
                        long timeNs;
                        boolean waitingTimeElapsed;
                        try {
                            waitingTimeElapsed = !moreMemory.await(remainingTimeToBlockNs, TimeUnit.NANOSECONDS); // 超时返回false
                        } 
                            //申请的大小为16k，free池有空闲的ByteBuffer
                        if (accumulated == 0 && size == this.poolableSize && !this.free.isEmpty()) {
                            // just grab a buffer from the free list
                            buffer = this.free.pollFirst();
                            accumulated = size;
                        } else {
                            // 释放空间给非池化可用内存，并继续等待空闲空间，如果分配多了只取size大小的空间
                            freeUp(size - accumulated);
                            int got = (int) Math.min(size - accumulated, this.nonPooledAvailableMemory);
                            // 释放非池化可用内存大小
                            this.nonPooledAvailableMemory -= got;
                            // 计算累计分配了多少空间
                            accumulated += got;
                        }
                    }
                    // Don't reclaim memory on throwable since nothing was thrown
                    accumulated = 0;
                } finally {
                    // When this loop was not able to successfully terminate don't loose available memory
                    this.nonPooledAvailableMemory += accumulated;
                    this.waiters.remove(moreMemory);
                }
            }
        } finally {
            // signal any additional waiters if there is more memory left
            // over for them
            try {
                if (!(this.nonPooledAvailableMemory == 0 && this.free.isEmpty()) && !this.waiters.isEmpty())
                    // 可用内存或free队列还有空闲，唤醒等待队列的第一个线程
                    this.waiters.peekFirst().signal();
            } finally {
                // Another finally... otherwise find bugs complains
                lock.unlock();
            }
        }
        if (buffer == null)
            // 没有正好的buffer，从缓冲区外(JVM Heap)中直接分配内存
            return safeAllocateByteBuffer(size);
        else
            // 直接复用free缓存池的ByteBuffer
            return buffer;
    }
```
##### 1.3.6.1.3 freeUp()
```java
// free向nonPooledAvailableMemory提供一部分内存
private void freeUp(int size) {
        while (!this.free.isEmpty() && this.nonPooledAvailableMemory < size)
            this.nonPooledAvailableMemory += this.free.pollLast().capacity(); }
```
##### 1.3.6.1.4 deallocate()
```java
public void deallocate(ByteBuffer buffer, int size) {
        lock.lock();
        try {
            if (size == this.poolableSize && size == buffer.capacity()) {
                buffer.clear();
                this.free.add(buffer);
            } else {
                this.nonPooledAvailableMemory += size;
            }
            Condition moreMem = this.waiters.peekFirst();
            if (moreMem != null)
                moreMem.signal();
        } finally {
            lock.unlock();
        }
    }
```
#### 1.3.6.2 append()
```java
// 调用 ProducerBatch.tryAppend() 方法将消息追加到底层 MemoryRecordsBuilder
public RecordAppendResult append(...) throws InterruptedException {
        TopicInfo topicInfo = topicInfoMap.computeIfAbsent(topic, k -> new TopicInfo(logContext, k, batchSize));
		// 累加数值
        appendsInProgress.incrementAndGet();
        ByteBuffer buffer = null;
        if (headers == null) headers = Record.EMPTY_HEADERS;
        try {
            // Loop to retry in case we encounter partitioner's race conditions.
            while (true) {
                //检查是否有正在进行的批次
                Deque<ProducerBatch> dq = topicInfo.batches.computeIfAbsent(effectivePartition, k -> new ArrayDeque<>());
                synchronized (dq) {
                    // 尝试添加
                    RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callbacks, dq, nowMs);
                    if (appendResult != null) {
                        // If queue has incomplete batches we disable switch (see comments in updatePartitionInfo).
                        boolean enableSwitch = allBatchesFull(dq);
                        topicInfo.builtInPartitioner.updatePartitionInfo(partitionInfo, appendResult.appendedBytes, cluster, enableSwitch);
                        return appendResult;
                    }
                }
				// 分配新的batch
                if (buffer == null) {
                    byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
                    int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
                    buffer = free.allocate(size, maxTimeToBlock);
                    nowMs = time.milliseconds();
                }
            }
        } finally {
            // 释放内存
            free.deallocate(buffer);
            appendsInProgress.decrementAndGet();
        }
}
```
#### 1.3.6.3 tryAppend()
```java
// Append the record to the current record set and return the relative offset within that record set
private RecordAppendResult tryAppend(long timestamp, byte[] key, byte[] value, Header[] headers, Callback callback, Deque<ProducerBatch> deque, long nowMs) {
        ProducerBatch last = deque.peekLast();
        if (last != null) {
            int initialBytes = last.estimatedSizeInBytes();
            FutureRecordMetadata future = last.tryAppend(timestamp, key, value, headers, callback, nowMs);
            if (future == null) {...} 
            else {
                int appendedBytes = last.estimatedSizeInBytes() - initialBytes;
                return new RecordAppendResult(future, deque.size() > 1 || last.isFull(), false, false, appendedBytes);
            }
        }
        return null;
}

public FutureRecordMetadata tryAppend(long timestamp, byte[] key, byte[] value, Header[] headers, Callback callback, long now) {
        if (!recordsBuilder.hasRoomFor(timestamp, key, value, headers)) {
            return null;
        } else {
            this.recordsBuilder.append(timestamp, key, value, headers);
            return future;
        }
}

// MemoryRecordsBuilder
public void append(SimpleRecord record) {
        appendWithOffset(nextSequentialOffset(), record);
}

private void appendWithOffset(long offset, boolean isControlRecord, long timestamp, ByteBuffer key,ByteBuffer value, Header[] headers) {
        try {

            if (magic > RecordBatch.MAGIC_VALUE_V1) {
                appendDefaultRecord(offset, timestamp, key, value, headers);
            } else {
                appendLegacyRecord(offset, timestamp, key, value, magic);
            }
        }
}

// 最后通过appendDefaultRecord或是appendLegacyRecord进行最终处理
private void appendDefaultRecord(long offset, long timestamp, ByteBuffer key, ByteBuffer value, Header[] headers) throws IOException {
        int sizeInBytes = DefaultRecord.writeTo(appendStream, offsetDelta, timestampDelta, key, value, headers);
        recordWritten(offset, timestamp, sizeInBytes);
}

public static int writeTo(...) throws IOException {
    	// 消息总大小
        int sizeInBytes = sizeOfBodyInBytes(offsetDelta, timestampDelta, key, value, headers);
        out.write(attributes);

        ByteUtils.writeVarlong(timestampDelta, out);
        ByteUtils.writeVarint(offsetDelta, out);
		// 写入K, V以及header
        if (key == null) {
            ByteUtils.writeVarint(-1, out);
        } else {
            int keySize = key.remaining();
            ByteUtils.writeVarint(keySize, out);
            Utils.writeTo(out, key, keySize);
        }

        if (value == null) {
            ByteUtils.writeVarint(-1, out);
        } else {
            int valueSize = value.remaining();
            ByteUtils.writeVarint(valueSize, out);
            Utils.writeTo(out, value, valueSize);
        }

        if (headers == null)
            throw new IllegalArgumentException("Headers cannot be null");

        for (Header header : headers) {
            String headerKey = header.key();
            if (headerKey == null)
                throw new IllegalArgumentException("Invalid null header key found in headers");

            byte[] utf8Bytes = Utils.utf8(headerKey);
            ByteUtils.writeVarint(utf8Bytes.length, out);
            out.write(utf8Bytes);

            byte[] headerValue = header.value();
            if (headerValue == null) {
                ByteUtils.writeVarint(-1, out);
            } else {
                ByteUtils.writeVarint(headerValue.length, out);
                out.write(headerValue);
            }
        }

        return ByteUtils.sizeOfVarint(sizeInBytes) + sizeInBytes;
}
```
### 1.3.7 向 Broker 发送数据
#### 1.3.7.1 sendProducerData()
```java
// batch满或时间到后唤醒Sender
private long sendProducerData(long now) {
    Cluster cluster = metadata.fetch();
    // 计算需要向哪些节点发送请求
    RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);
    if (!result.unknownLeaderTopics.isEmpty()) {
        // 存在不知道leader的分区，更新metadata
        for (String topic : result.unknownLeaderTopics)
            this.metadata.add(topic, now);
        this.metadata.requestUpdate();
    }
    // remove any nodes we aren't ready to send to
    Iterator<Node> iter = result.readyNodes.iterator();
    long notReadyTimeout = Long.MAX_VALUE;
    while (iter.hasNext()) {
        Node node = iter.next();
        // 检查目标节点是否准备好接收请求，如果未准备好但目标节点允许创建连接，则创建到目标节点的连接
        if (!this.client.ready(node, now)) {...} 
        else {
            // Update both readyTimeMs and drainTimeMs, this would "reset" the node latency.
            this.accumulator.updateNodeLatencyStats(node.id(), now, true);
        }
    }
        // 获取每个节点待发送消息集合，其中key是目标leader副本所在节点
        Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes, this.maxRequestSize, now);
        addToInflightBatches(batches);
        if (guaranteeMessageOrder) {
            // 如果需要保证消息的强顺序性，则缓存对应 topic 分区对象，防止同一时间往同一个 topic 分区发送多条处于未完成状态的消息
            // Mute all the partitions drained
            for (List<ProducerBatch> batchList : batches.values()) {
                for (ProducerBatch batch : batchList)
                    this.accumulator.mutePartition(batch.topicPartition);
            }
        }
    	// 处理本地过期的消息，返回 TimeoutException，并释放空间
        accumulator.resetNextBatchExpiryTime();
        List<ProducerBatch> expiredInflightBatches = getExpiredInflightBatches(now);
        List<ProducerBatch> expiredBatches = this.accumulator.expiredBatches(now);
        expiredBatches.addAll(expiredInflightBatches);

        // 发送请求到服务端，并处理服务端响应
        sendProduceRequests(batches, now);
        return pollTimeout;
    }
```
#### 1.3.7.2 ready()
```java
// RecordAccumulator.java
// Get a list of nodes whose partitions are ready to be sent
public ReadyCheckResult ready(Cluster cluster, long nowMs) {
        Set<Node> readyNodes = new HashSet<>(); // 记录接受请求的节点
        long nextReadyCheckDelayMs = Long.MAX_VALUE; // 记录下次执行 ready 判断的时间间隔
        Set<String> unknownLeaderTopics = new HashSet<>();  // 记录找不到 leader 副本的分区对应的 topic 集合
        for (Map.Entry<String, TopicInfo> topicInfoEntry : this.topicInfoMap.entrySet()) {
            final String topic = topicInfoEntry.getKey();
            // 遍历topic获取队列大小
            nextReadyCheckDelayMs = partitionReady(cluster, nowMs, topic, topicInfoEntry.getValue(), nextReadyCheckDelayMs, readyNodes, unknownLeaderTopics);
        }
        return new ReadyCheckResult(readyNodes, nextReadyCheckDelayMs, unknownLeaderTopics);

private long partitionReady(Cluster cluster, long nowMs, String topic,
                                TopicInfo topicInfo,
                                long nextReadyCheckDelayMs, Set<Node> readyNodes, Set<String> unknownLeaderTopics) {

        // 判断是否有线程在等待BufferPool分配空间
        boolean exhausted = this.free.queued() > 0;
    	// 遍历Topic的分区及其 RecordBatch 队列，对每个分区的 leader 副本所在的节点执行判定
        for (Map.Entry<Integer, Deque<ProducerBatch>> entry : batches.entrySet()) {
            // 创建topicpartition
            TopicPartition part = new TopicPartition(topic, entry.getKey());
            // leader所在节点
            Node leader = cluster.leaderFor(part);
            synchronized (deque) {
                // Deques are often empty in this path, esp with large partition counts,
                // so we exit early if we can.
                ProducerBatch batch = deque.peekFirst();
                if (batch == null) {
                    continue;
                }
            }

            if (leader == null) {
            	// 添加到没有leader的topic里 
                unknownLeaderTopics.add(part.topic());
            } else {
				// leader不为null
                nextReadyCheckDelayMs = batchReady(nowMs, exhausted, part, leader, waitedTimeMs, backingOff,
                    full, nextReadyCheckDelayMs, readyNodes);
            }
        }
        return nextReadyCheckDelayMs;
    }
}

private long batchReady(long nowMs, boolean exhausted, TopicPartition part, Node leader, long waitedTimeMs, boolean backingOff, boolean full,
     long nextReadyCheckDelayMs, Set<Node> readyNodes) {
        if (!readyNodes.contains(leader) && !isMuted(part)) {
            long timeToWaitMs = backingOff ? retryBackoffMs : lingerMs;
            boolean expired = waitedTimeMs >= timeToWaitMs;
            boolean transactionCompleting = transactionManager != null && transactionManager.isCompleting(); 
            // 标记当前节点是否可以接收请求，如果满足其中一个则认为需要往目标节点投递消息：
            boolean sendable = full // 1.队列中有多个RecordBatch，或第一个RecordBatch已满
                    || expired // 2.当前等待重试的时间过长
                    || exhausted // 3.有其他线程在等待BufferPool分配空间，本地消息缓存已满
                    || closed // 4.producer已经关闭
                    || flushInProgress() // 5.有线程正在等待flush操作完成
                    || transactionCompleting;
            // 允许发送消息，且为首次发送，或者重试等待时间已经较长，则记录目标leader副本所在节点
            if (sendable && !backingOff) {
                readyNodes.add(leader);
            } else {
                long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
            }
        }
        return nextReadyCheckDelayMs;
    }
```
#### 1.3.7.3 drain()
```java
// 回到sendProducerData中
//  create produce requests
public Map<Integer, List<ProducerBatch>> drain(Cluster cluster, Set<Node> nodes, int maxSize, long now) {

        Map<Integer, List<ProducerBatch>> batches = new HashMap<>();
        for (Node node : nodes) {
            // 循环处理已经准备好的结点
            List<ProducerBatch> ready = drainBatchesForOneNode(cluster, node, maxSize, now);
            batches.put(node.id(), ready);
        }
        return batches;
}

private List<ProducerBatch> drainBatchesForOneNode(Cluster cluster, Node node, int maxSize, long now) {
        int size = 0;
    	// 获取当前节点上的分区信息
        List<PartitionInfo> parts = cluster.partitionsForNode(node.id());
    	// 记录待发往当前节点的 RecordBatch 集合
        List<ProducerBatch> ready = new ArrayList<>();
        int drainIndex = getDrainIndex(node.idString());
    	// drainIndex 用于记录上次发送停止的位置，本次继续从当前位置开始发送
        int start = drainIndex = drainIndex % parts.size();
            synchronized (deque) {
                // invariant: !isMuted(tp,now) && deque != null
            	// 获取当前分区对应的 RecordBatch 集合
                ProducerBatch first = deque.peekFirst();

            	// 仅发送第一次发送，或重试等待时间较长的消息
                if (size + first.estimatedSizeInBytes() > maxSize && !ready.isEmpty()) {
                    break;
                } else {
                    if (shouldStopDrainBatchesForPartition(first, tp))
                        break;
                }
				// 每次仅获取第一个 RecordBatch，并放入 read 列表中，这样给每个分区一个机会，保证公平，防止饥饿
                batch = deque.pollFirst();
            batch.close();
            size += batch.records().sizeInBytes();
            ready.add(batch);
            batch.drained(now);
        } while (start != drainIndex);
        return ready;
}
```
#### 1.3.7.4 addToInflightBatches()
```java
// 将上面得到的batches添加到inflightBatches中
public void addToInflightBatches(Map<Integer, List<ProducerBatch>> batches) {
        for (List<ProducerBatch> batchList : batches.values()) {
            addToInflightBatches(batchList);
        }
}

private void addToInflightBatches(List<ProducerBatch> batches) {
        for (ProducerBatch batch : batches) {
            List<ProducerBatch> inflightBatchList = inFlightBatches.get(batch.topicPartition);
            inflightBatchList.add(batch);
        }
}
```
#### 1.3.7.5 sendProduceRequests()
```java
private void sendProduceRequests(Map<Integer, List<ProducerBatch>> collated, long now) {
        // 遍历ProducerBatch
        for (Map.Entry<Integer, List<ProducerBatch>> entry : collated.entrySet())
            sendProduceRequest(now, entry.getKey(), acks, requestTimeoutMs, entry.getValue());
}

private void sendProduceRequest(long now, int destination, short acks, int timeout, List<ProducerBatch> batches) {
    	// 遍历 RecordBatch 集合，整理成 produceRecordsByPartition 和 recordsByPartition 
        for (ProducerBatch batch : batches) {
            TopicPartition tp = batch.topicPartition;
            // batch中的records是MemoryRecords，底层是ByteBuffer
            MemoryRecords records = batch.records();
            
            // 让每个batch的数据都有分区
            ProduceRequestData.TopicProduceData tpData = tpd.find(tp.topic());
            if (tpData == null) {
                tpData = new ProduceRequestData.TopicProduceData().setName(tp.topic());
                tpd.add(tpData);
            }
            tpData.partitionData().add(new ProduceRequestData.PartitionProduceData()
                    .setIndex(tp.partition())
                    .setRecords(records));
            recordsByPartition.put(tp, batch);
        }

        String transactionalId = null;
        if (transactionManager != null && transactionManager.isTransactional()) {
            transactionalId = transactionManager.transactionalId();
        }
		// 创建 ProduceRequest 请求构造器
        ProduceRequest.Builder requestBuilder = ProduceRequest.forMagic(minUsedMagic,
                new ProduceRequestData()
                        .setAcks(acks)
                        .setTimeoutMs(timeout)
                        .setTransactionalId(transactionalId)
                        .setTopicData(tpd));
        RequestCompletionHandler callback = response -> handleProduceResponse(response, recordsByPartition, time.milliseconds());

        String nodeId = Integer.toString(destination);
    	// 创建 ClientRequest 请求对象，如果 acks 不等于 0 则表示期望获取服务端响应
        ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0,
                requestTimeoutMs, callback
         // 将请求加入到网络 I/O 通道（KafkaChannel）中。同时将该对象缓存到 InFlightRequests 中 
        // 创建Sender时传进来的创建的network对象             
        client.send(clientRequest, now);
        log.trace("Sent produce request to {}: {}", nodeId, requestBuilder);
}	
```
## 1.4 Broker 进行响应
### 1.4.1 KafkaApis.handle()
```scala
// Broker通过KafkaApis类来处理请求
  override def handle(request: RequestChannel.Request): Unit = {
    try {
      request.header.apiKey match {
        // 处理生产者的请求
        case ApiKeys.PRODUCE => handleProduceRequest(request)
        case ApiKeys.FETCH => handleFetchRequest(request)
        case ApiKeys.METADATA => handleTopicMetadataRequest(request)
        case ApiKeys.UPDATE_METADATA => handleUpdateMetadataRequest(request)
        case ApiKeys.INIT_PRODUCER_ID => handleInitProducerIdRequest(request)
        ...
}
```
### 1.4.2 handleProduceRequest()
```scala
def handleProduceRequest(request: RequestChannel.Request): Unit = {
	val produceRequest = request.body[ProduceRequest]
	 val produceRecords = produceRequest.partitionRecordsOrFail.asScala
	 // 回调函数
    def sendResponseCallback(responseStatus: Map[TopicPartition, PartitionResponse]): Unit = {
      // ack = 0时默认发送成功
      if (produceRequest.acks == 0) {
        // no operation needed if producer request.required.acks = 0; however, if there is any error in handling
        // the request, since no response is expected by the producer, the server will close socket server so that
        // the producer client will know that some error has happened and will refresh its metadata
      } else {
        // ack不为0的情况下发送响应
        sendResponse(request, Some(new ProduceResponse(mergedResponseStatus.asJava, maxThrottleTimeMs)), None)
      }
	// 处理完响应后通过replicaManager写入record
      // call the replica manager to append messages to the replicas
      replicaManager.appendRecords(...)
}
```
#### 1.4.2.1 sendResponse()
```scala
private def sendResponse(request: RequestChannel.Request,
                           responseOpt: Option[AbstractResponse],
                           onComplete: Option[Send => Unit]): Unit = {
	// 创建response
    val response = responseOpt match {
      case Some(response) =>
        val responseSend = request.context.buildResponse(response)
        val responseString =
          if (RequestChannel.isRequestLoggingEnabled) Some(response.toString(request.context.apiVersion))
          else None
        new RequestChannel.SendResponse(request, responseSend, responseString, onComplete)
      case None =>
        new RequestChannel.NoOpResponse(request)
    }
	// 通过requestChannel发送响应
    requestChannel.sendResponse(response)
}
```
#### 1.4.2.2 appendRecords()
```scala
// KafkaServer.scala
// KafkaServer在启动时会启动ReplicaManager
def startup(): Unit = {
    try {
      info("starting")

      val canStartup = isStartingUp.compareAndSet(false, true)
      if (canStartup) {
        brokerState.newState(Starting)
        // 创建replicaManager
        replicaManager = createReplicaManager(isShuttingDown)
        replicaManager.startup()
}

// Append messages to leader replicas of the partition
// 向partition的leader写入数据
def appendRecords(...): Unit = {
    // ack有效(ack=0, -1, 1时)
    if (isValidRequiredAcks(requiredAcks)) {
      val sTime = time.milliseconds
      // 向本地日志写入数据
      val localProduceResults = appendToLocalLog(internalTopicsAllowed = internalTopicsAllowed,
        origin, entriesPerPartition, requiredAcks)
	  // ack为-1时，isr的follower都写入成功后才会返回最终结果
      if (delayedProduceRequestRequired(requiredAcks, entriesPerPartition, localProduceResults)) {
        // create delayed produce operation
        val produceMetadata = ProduceMetadata(requiredAcks, produceStatus)
        // 创建DelayedProduce对象等待isr其他副本的处理结果
        val delayedProduce = new DelayedProduce(timeout, produceMetadata, this, responseCallback, delayedProduceLock)
      } else {
        // 不是-1时可以直接返回append结果
        // we can respond immediately
        val produceResponseStatus = produceStatus.map { case (k, status) => k -> status.responseStatus }
        responseCallback(produceResponseStatus)
      }
    } else {
      // ack无效，报错
      // If required.acks is outside accepted range, something is wrong with the client
      // Just return an error and don't handle the request at all
      val responseStatus = entriesPerPartition.map { case (topicPartition, _) =>
        topicPartition -> new PartitionResponse(Errors.INVALID_REQUIRED_ACKS,
          LogAppendInfo.UnknownLogAppendInfo.firstOffset.getOrElse(-1), RecordBatch.NO_TIMESTAMP, LogAppendInfo.UnknownLogAppendInfo.logStartOffset)
      }
      responseCallback(responseStatus)
    }
}        
```
#### 1.4.2.3 appendToLocalLog()
```java
private def appendToLocalLog(internalTopicsAllowed: Boolean,
                               origin: AppendOrigin,
                               entriesPerPartition: Map[TopicPartition, MemoryRecords],
                               requiredAcks: Short): Map[TopicPartition, LogAppendResult] = {
    val traceEnabled = isTraceEnabled
    def processFailedRecord(topicPartition: TopicPartition, t: Throwable) = {
      val logStartOffset = getPartition(topicPartition) match {
        case HostedPartition.Online(partition) => partition.logStartOffset
        case HostedPartition.None | HostedPartition.Offline => -1L
      }
	...
      // reject appending to internal topics if it is not allowed
      // 是否为内置topic
      if (Topic.isInternal(topicPartition.topic) && !internalTopicsAllowed) {
        (topicPartition, LogAppendResult(
          LogAppendInfo.UnknownLogAppendInfo,
          Some(new InvalidTopicException(s"Cannot append to internal topic ${topicPartition.topic}"))))
      } else {
        try {
          val partition = getPartitionOrException(topicPartition)
          // 向leader添加record
          val info = partition.appendRecordsToLeader(records, origin, requiredAcks)
          val numAppendedMessages = info.numMessages

          // update stats for successfully appended bytes and messages as bytesInRate and messageInRate
         ...
}
       
// Partition.scala
// makeLeader方法中创建了Log对象leaderLog
def appendRecordsToLeader(records: MemoryRecords, origin: AppendOrigin, requiredAcks: Int): LogAppendInfo = {
    val (info, leaderHWIncremented) = inReadLock(leaderIsrUpdateLock) {
      leaderLogIfLocal match {
        case Some(leaderLog) =>
          val minIsr = leaderLog.config.minInSyncReplicas
          val inSyncSize = isrState.isr.size

          // Avoid writing to leader if there are not enough insync replicas to make it safe   
          // isr未达到minisr且ack为-1，抛异常
          if (inSyncSize < minIsr && requiredAcks == -1) {
            throw new NotEnoughReplicasException(s"The size of the current ISR ${isrState.isr} " +
              s"is insufficient to satisfy the min.isr requirement of $minIsr for partition $topicPartition")
          }
		  // 调用Log的appendAsLeader方法写入record
          val info = leaderLog.appendAsLeader(records, leaderEpoch = this.leaderEpoch, origin,
            interBrokerProtocolVersion)

          // we may need to increment high watermark since ISR could be down to 1
          (info, maybeIncrementLeaderHW(leaderLog))

        case None =>
          throw new NotLeaderOrFollowerException("Leader not local for partition %s on broker %d"
            .format(topicPartition, localBrokerId))
      }
    }

    info.copy(leaderHwChange = if (leaderHWIncremented) LeaderHwChange.Increased else LeaderHwChange.Same)
}
```
# 2. Broker
- 基本组件

![[broker_framework.svg]]

- `SocketServer`：监听`9092`端口的 `socket` 读写请求
	- 首先开启一个 `Acceptor` 线程，新的 `Socket` 连接成功建立时会将对应的 `SocketChannel` 以轮询方式转发给 N 个 `Processor` 线程中的某一个，由其处理接下来 `SocketChannel` 的请求
	- 将请求放置在 `RequestChannel` 中的请求队列；当 `Processor` 线程监听到 `SocketChannel` 请求的响应时，会将响应从 `RequestChannel` 中的响应队列中取出来并发给客户端
- `KafkaRequesThandlePool`：从 `Socketserver` 获取客户端请求然后调用 `KafkaApis` 实现业务逻辑处理，将响应结果返回给 `RequestChannel`,再给客户端
- `KafkaApis`：`kafka` 的业务逻辑实现层, 各种 `Request` 的实现类，通过 `match-case` 匹配请求类型调用实现逻辑
- `KafkaController`：`kafka` 的集群管理模块。通过在 `zk` 上注册节点实现监听
- `OffsetManager`：`offset` 管理模块。对偏移量的保存和读取，offset 保存在 `zk` 或 `Kafka` 中，由参数 `offsets.storage=zookeeper/kafka` 决定
- `TopicConfigManager`：`topic` 的配置信息管理模块，副本数等
- `KafkaScheduler`：kafka 的后台定时任务调度的资源池
	- 主要为 `LogManager`, `ReplicaManager` , `OffsetManager` 提供资源调度
- `ReplicaManager`：副本管理
	- 包括有关副本的 `Leader` 和 `ISR` 的状态变化、副本的删除、副本的监测等
- `KafkaHealthCheck`：集群健康检查
- 日志管理模块。脏数据，过期数据删除等，负责提供 `Broker` 上 `Topic` 的分区数据读取和写入功能
## 2.1 启动
### 2.1.1 Kafka.scala
```scala
// kafka-server-start.sh最后会通过kafka-run-class.sh运行Kafka包下的Kafka类
// exec $base_dir/kafka-run-class.sh $EXTRA_ARGS kafka.Kafka "$@"
def main(args: Array[String]): Unit = {
    try {
      val serverProps = getPropsFromArgs(args)
      // 获取一个KafkaServerStartable对象
      val kafkaServerStartable = KafkaServerStartable.fromProps(serverProps)

      // attach shutdown handler to catch terminating signals as well as normal termination
      Exit.addShutdownHook("kafka-shutdown-hook", kafkaServerStartable.shutdown())
		
      // 启动
      kafkaServerStartable.startup()
      kafkaServerStartable.awaitShutdown()
    }
}

object KafkaServerStartable {
  def fromProps(serverProps: Properties): KafkaServerStartable = {
    fromProps(serverProps, None)
  }

  def fromProps(serverProps: Properties, threadNamePrefix: Option[String]): KafkaServerStartable = {
    val reporters = KafkaMetricsReporter.startReporters(new VerifiableProperties(serverProps))
    // 创建KafkaServerStartable对象
    new KafkaServerStartable(KafkaConfig.fromProps(serverProps, false), reporters, threadNamePrefix)
  }
}
```
### 2.1.2 KafkaServerStartable.scala
```scala
class KafkaServerStartable(val staticServerConfig: KafkaConfig, reporters: Seq[KafkaMetricsReporter], threadNamePrefix: Option[String] = None) extends Logging {
  // 创建一个KafkaServer对象
  private val server = new KafkaServer(staticServerConfig, kafkaMetricsReporters = reporters, threadNamePrefix = threadNamePrefix)

  def this(serverConfig: KafkaConfig) = this(serverConfig, Seq.empty)

  def startup(): Unit = {
    // 启动server
    try server.startup()
  }
}
```
### 2.1.3 启动 KafkaServer
```scala
  def startup(): Unit = {
    try {
      info("starting")
      
      if (canStartup) {
        brokerState.newState(Starting)

        /* setup zookeeper */
        // 连接zk，创建根节点
        initZkClient(time)

        /* Get or create cluster_id */
        // 从ZK获取或创建集群id，规则：UUID的mostSigBits、leastSigBits组合转base64
        _clusterId = getOrGenerateClusterId(zkClient)
        info(s"Cluster ID = $clusterId")

        /* load metadata */
        // 获取brokerId及log存储路径，brokerId通过zk生成或者server.properties配置broker.id
        val (preloadedBrokerMetadataCheckpoint, initialOfflineDirs) = getBrokerMetadataAndOfflineDirs

        /* generate brokerId */
        config.brokerId = getOrGenerateBrokerId(preloadedBrokerMetadataCheckpoint)
        logContext = new LogContext(s"[KafkaServer id=${config.brokerId}] ")
        this.logIdent = logContext.logPrefix

        /* start scheduler */
        // 初始化定时任务调度器
        kafkaScheduler = new KafkaScheduler(config.backgroundThreads)
        kafkaScheduler.startup()

        /* create and configure metrics */
        ...

        /* start log manager */
        // 启动LogManager
        logManager = LogManager(config, initialOfflineDirs, zkClient, brokerState, kafkaScheduler, time, brokerTopicStats, logDirFailureChannel)
        logManager.startup()

        metadataCache = new MetadataCache(config.brokerId)
		// 启动socket监听9092等待客户端请求
        // Create and start the socket server acceptor threads so that the bound port is known.
        // Delay starting processors until the end of the initialization sequence to ensure
        // that credentials have been loaded before processing authentications.
        socketServer = new SocketServer(config, metrics, time, credentialProvider)
        socketServer.startup(startProcessingRequests = false)

        /* start replica manager */
        brokerToControllerChannelManager = new BrokerToControllerChannelManagerImpl(metadataCache, time, metrics, config, threadNamePrefix)
        // 启动副本管理器
        replicaManager = createReplicaManager(isShuttingDown)
        replicaManager.startup()
        brokerToControllerChannelManager.start()

        val brokerInfo = createBrokerInfo
        // 向zk注册Broker
        val brokerEpoch = zkClient.registerBroker(brokerInfo)

        // Now that the broker is successfully registered, checkpoint its metadata
checkpointBrokerMetadata(BrokerMetadata(config.brokerId, Some(clusterId)))

        /* start token manager */
        tokenManager = new DelegationTokenManager(config, tokenCache, time , zkClient)
        tokenManager.startup()

        /* start kafka controller */
          // 启动controller，只有leader会与zk建立连接
        kafkaController = new KafkaController(config, zkClient, time, metrics, brokerInfo, brokerEpoch, tokenManager, brokerFeatures, featureCache, threadNamePrefix)
        kafkaController.startup()

        adminManager = new AdminManager(config, metrics, metadataCache, zkClient)
		
        // 创建GroupCoordinator用于和ConsumerCoordinator 交互
        /* start group coordinator */
        groupCoordinator = GroupCoordinator(config, zkClient, replicaManager, Time.SYSTEM, metrics)
        groupCoordinator.startup()

        /* start transaction coordinator, with a separate background thread scheduler for transaction expiration and log loading */
        // Hardcode Time.SYSTEM for now as some Streams tests fail otherwise, it would be good to fix the underlying issue
        transactionCoordinator = TransactionCoordinator(config, replicaManager, new KafkaScheduler(threads = 1, threadNamePrefix = "transaction-log-manager-"), zkClient, metrics, metadataCache, Time.SYSTEM)
        transactionCoordinator.startup()

        /* start processing requests */
        // 初始化数据类请求的KafkaApis，负责数据类请求逻辑处理
        dataPlaneRequestProcessor = new KafkaApis(socketServer.dataPlaneRequestChannel, replicaManager, adminManager, groupCoordinator, transactionCoordinator,
          kafkaController, zkClient, config.brokerId, config, metadataCache, metrics, authorizer, quotaManagers,
          fetchManager, brokerTopicStats, clusterId, time, tokenManager, brokerFeatures, featureCache)
		// 初始化数据类请求处理的线程池  
        dataPlaneRequestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.dataPlaneRequestChannel, dataPlaneRequestProcessor, time,
          config.numIoThreads, s"${SocketServer.DataPlaneMetricPrefix}RequestHandlerAvgIdlePercent", SocketServer.DataPlaneThreadPrefix)

        socketServer.controlPlaneRequestChannelOpt.foreach { controlPlaneRequestChannel =>
          // 初始化控制类请求的 KafkaApis
          controlPlaneRequestProcessor = new KafkaApis(controlPlaneRequestChannel, replicaManager, adminManager, groupCoordinator, transactionCoordinator,
            kafkaController, zkClient, config.brokerId, config, metadataCache, metrics, authorizer, quotaManagers,
            fetchManager, brokerTopicStats, clusterId, time, tokenManager, brokerFeatures, featureCache)
	  	  // 初始化控制类请求的线程池
          controlPlaneRequestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.controlPlaneRequestChannelOpt.get, controlPlaneRequestProcessor, time, 1, s"${SocketServer.ControlPlaneMetricPrefix}RequestHandlerAvgIdlePercent", SocketServer.ControlPlaneThreadPrefix)
        }
        // Create the config manager. start listening to notifications
        dynamicConfigManager = new DynamicConfigManager(zkClient, dynamicConfigHandlers)
        dynamicConfigManager.startup()

    socketServer.startProcessingRequests(authorizerFutures)
		// 更新broker状态
        brokerState.newState(RunningAsBroker)
        shutdownLatch = new CountDownLatch(1)
        startupComplete.set(true)
        isStartingUp.set(false)
        AppInfoParser.registerAppInfo(metricsPrefix, config.brokerId.toString, metrics, time.milliseconds())
        info("started")
      }
    }
}
```
### 2.1.4 连接 zk 创建节点
```scala
private def initZkClient(time: Time): Unit = {
    info(s"Connecting to zookeeper on ${config.zkConnect}")
	
    // 创建ZkClient
    def createZkClient(zkConnect: String, isSecure: Boolean) = {
      KafkaZkClient(zkConnect, isSecure, config.zkSessionTimeoutMs, config.zkConnectionTimeoutMs,
        config.zkMaxInFlightRequests, time, name = Some("Kafka server"), zkClientConfig = Some(zkClientConfig))
    }

    // make sure chroot path exists
    // 确保根节点存在
    chrootOption.foreach { chroot =>
      val zkConnForChrootCreation = config.zkConnect.substring(0, chrootIndex)
      val zkClient = createZkClient(zkConnForChrootCreation, secureAclsEnabled)
      zkClient.makeSurePersistentPathExists(chroot)
      info(s"Created zookeeper path $chroot")
      zkClient.close()
    }

    _zkClient = createZkClient(config.zkConnect, secureAclsEnabled)
    _zkClient.createTopLevelPaths()
}

// KafkaZkClient的apply
def apply(connectString: String,
            isSecure: Boolean,
            sessionTimeoutMs: Int,
            connectionTimeoutMs: Int,
            maxInFlightRequests: Int,
            time: Time,
            metricGroup: String = "kafka.server",
            metricType: String = "SessionExpireListener",
            name: Option[String] = None,
            zkClientConfig: Option[ZKClientConfig] = None) = {
	// 创建一个 ZooKeeperClient 和 KafkaZkClient
    val zooKeeperClient = new ZooKeeperClient(connectString, sessionTimeoutMs, connectionTimeoutMs, maxInFlightRequests,
      time, metricGroup, metricType, name, zkClientConfig)
    new KafkaZkClient(zooKeeperClient, isSecure, time)
}

// KafkaZkClient对ZooKeeperClient进行封装，内部定义了对zk的操作
class KafkaZkClient private[zk] (zooKeeperClient: ZooKeeperClient, isSecure: Boolean, time: Time) extends AutoCloseable with Logging with KafkaMetricsGroup {
	def registerBroker(brokerInfo: BrokerInfo): Long = {...}
    def registerControllerAndIncrementControllerEpoch(controllerId: Int): (Int, Int) = {...}
    def updateLeaderAndIsr(
    leaderAndIsrs: Map[TopicPartition, LeaderAndIsr],
    controllerEpoch: Int,
    expectedControllerEpochZkVersion: Int
  ): UpdateLeaderAndIsrResult = {...}
}

class ZooKeeperClient(connectString: String,
                      sessionTimeoutMs: Int,
                      connectionTimeoutMs: Int,
                      maxInFlightRequests: Int,
                      time: Time,
                      metricGroup: String,
                      metricType: String,
                      name: Option[String],
                      zkClientConfig: Option[ZKClientConfig]) extends Logging with KafkaMetricsGroup {
      // ZooKeeperClient内部定义了ZooKeeperClientWatcher用于监听zookeeper
      private[zookeeper] object ZooKeeperClientWatcher extends Watcher {
    override def process(event: WatchedEvent): Unit = {
      debug(s"Received event: $event")
      Option(event.getPath) match {
        case None =>
          val state = event.getState
          stateToMeterMap.get(state).foreach(_.mark())
          inLock(isConnectedOrExpiredLock) {
            isConnectedOrExpiredCondition.signalAll()
          }
          if (state == KeeperState.AuthFailed) {
            error("Auth failed.")
            stateChangeHandlers.values.foreach(_.onAuthFailure())
          } else if (state == KeeperState.Expired) {
            scheduleSessionExpiryHandler()
          }
        case Some(path) =>
          /*
           * zNodeChildChangeHandlers有以下几个子类：
           * BrokerChangeHandler
           * IsrChangeNotificationHandler
           * TopicDeletionHandler
           * ChangeNotificationHandler
           * LogDirEventNotificationHandler
           * TopicChangeHandler
           */
          // 监听到节点的变化后，根据不同的event调用不同的handler
          (event.getType: @unchecked) match {
            case EventType.NodeChildrenChanged => zNodeChildChangeHandlers.get(path).foreach(_.handleChildChange())
            case EventType.NodeCreated => zNodeChangeHandlers.get(path).foreach(_.handleCreation())
            case EventType.NodeDeleted => zNodeChangeHandlers.get(path).foreach(_.handleDeletion())
            case EventType.NodeDataChanged => zNodeChangeHandlers.get(path).foreach(_.handleDataChange())
          }
      }
    }
  }
}
```
## 2.2 Controller 选举 Leader

![[controller_select.svg]]

### 2.2.1 启动 Controller
```scala
def startup() = {  
  zkClient.registerStateChangeHandler(new StateChangeHandler {  
    override val name: String = StateChangeHandlers.ControllerHandler  
    override def afterInitializingSession(): Unit = {  
      eventManager.put(RegisterBrokerAndReelect)  
    }  
    override def beforeInitializingSession(): Unit = {  
      val queuedEvent = eventManager.clearAndPut(Expire)  
  
      // Block initialization of the new session until the expiration event is being handled,  
      // which ensures that all pending events have been processed before creating the new session      queuedEvent.awaitProcessing()  
    }  
  })  
  // ControllerEventManager 中定义了一个 QueuedEvent
  controllerContext.stats.rateAndTimeMetrics)
  eventManager.put(Startup)  
  eventManager.start()  
}
```