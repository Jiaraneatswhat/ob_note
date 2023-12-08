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
- **申请的大小为 16kb 且 free 缓存池有缓存可用**

![[bufferpool_allocate1.svg]]
- 从 <font color='red'>free</font> 缓存池的队首拿出一个<font color='red'>16kb</font>的 ByteBuffer 来直接使用，等到 ByteBuffer 用完之后，通过 clear()方法放入 free 缓存池的尾部，随后唤醒下一个等待分配内存的线程
