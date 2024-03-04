# 1 Producer
- 基本架构
	- ProducerMetadata：`Producer` 与 `Broker` 通讯获取元数据
	- ProducerConfig：设置 `Producer` 的参数
	- NetworkClient：网络通信
	- RecordAccumlator：缓存 `Batch`
	- Sender：发送数据
	- ProducerInterceptors：用于处理拦截器逻辑(可选)

![[producer_components.svg]]

## 1.1 fields
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
## 1.5 生产者的分区器
- Partitioner 接口有三个子类
	- DefaultPartitioner
	- UniformStickyPartitioner
	- RoundRobinPartitioner
### 1.5.1 DefaultPartitioner
- 分区策略
	- `ProducerRecord` 中指定分区直接发往指定分区
	- 未指定分区，但是有 Key，通过 hash 值
	- 分区和 Key 都没有指定，通过粘性分区器处理
```java
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {  
    return partition(topic, key, keyBytes, value, valueBytes, cluster, cluster.partitionsForTopic(topic).size());  
}

// Cluster 中存放了 topic 和 partition 对应的映射关系
// private final Map<String, List<PartitionInfo>> partitionsByTopic;
// private final Map<TopicPartition, PartitionInfo> partitionsByTopicPartition;
// 首先去已经保存过的映射关系中，查找对应的分区
public List<PartitionInfo> partitionsForTopic(String topic) {  
    return partitionsByTopic.getOrDefault(topic, Collections.emptyList());  
}

public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster,  
                     int numPartitions) {  
	 // 未指定 key，通过粘性分区器处理
    if (keyBytes == null) {  
        return stickyPartitionCache.partition(topic, cluster);  
    }  
    // 指定了 key，计算 hash 值
    return BuiltInPartitioner.partitionForKey(keyBytes, numPartitions);  
}

// BuiltInPartitioner.java
// murmur2 算法求 hash
public static int partitionForKey(final byte[] serializedKey, final int numPartitions) {  
    return Utils.toPositive(Utils.murmur2(serializedKey)) % numPartitions;  
}
```
### 1.5.2 UniformStickyPartitioner
```java
// UniformStickyPartitioner 中创建了一个 StickyPartitionCache 用于决定当前主题的分区
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {  
    return stickyPartitionCache.partition(topic, cluster);  
}

public int partition(String topic, Cluster cluster) {
	// 已经缓存过 topic 
    Integer part = indexCache.get(topic);  
    if (part == null) {  
	    // 计算新的分区
        return nextPartition(topic, cluster, -1);  
    }  
    return part;  
}

public int nextPartition(String topic, Cluster cluster, int prevPartition) {  
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic); // null
    Integer oldPart = indexCache.get(topic); // null
    Integer newPart = oldPart; // null  
    if (oldPart == null || oldPart == prevPartition) {  
        // 可用的分区
        List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);  
        if (availablePartitions.size() < 1) {// 产生新的分区号
        } 
        else if (availablePartitions.size() == 1) {  
            // 集合中唯一一个元素的分区  
        } else {  
	        // 可用分区数有多个, 循环直到分区号与之前不同
            while (newPart == null || newPart.equals(oldPart)) {  
                int random = Utils.toPositive(ThreadLocalRandom.current().nextInt());  
                newPart = availablePartitions.get(random % availablePartitions.size()).partition();  
            }  
        }    
        return indexCache.get(topic);  
    }  
    return indexCache.get(topic);  
}

// UniformStickyPartitioner 还定义了 onNewBatch 方法，产生新的批次时会立即更换分区
public void onNewBatch(String topic, Cluster cluster, int prevPartition) {  
    stickyPartitionCache.nextPartition(topic, cluster, prevPartition);  
}
```
### 1.5.3 RoundRobinPartitioner
```java
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {  
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);  
    int numPartitions = partitions.size();  
    int nextValue = nextValue(topic);  
    List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);  
    // 有可用的分区时，用+1的值mod可用分区数
    if (!availablePartitions.isEmpty()) {  
        int part = Utils.toPositive(nextValue) % availablePartitions.size();  
        return availablePartitions.get(part).partition();  
        // 否则不分区
    } else {  
        // no partitions are available, give a non-available partition  
        return Utils.toPositive(nextValue) % numPartitions;  
    }  
}

// +1
private int nextValue(String topic) {  
    AtomicInteger counter = topicCounterMap.computeIfAbsent(topic, k -> new AtomicInteger(0));  
    return counter.getAndIncrement();  
}
```
## 1.6 单分区有序
- 通过幂等实现
- 未开启幂等，Broker 收到数据，还未返回 ack 时挂掉，重新生产数据时就会产生重复
![[nonidempotent.png]]
- 开启幂等，通过 `ProduceId` 来判断是否是同一个 `Producer`

![[idempotent.png]]
```java

```
# 2 Broker
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

![[controller_first_elect.svg]]

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
  // 将 Startup 类型的 ControllerEvent 添加到集合中
  eventManager.put(Startup)  
  // 启动 ControllerEventManager
  eventManager.start()  
}
```
### 2.2.2 启动 ControllerEventManager
```scala
// 定义了ControllerEventThread
// thread = new ControllerEventThread(ControllerEventThreadName)
// 启动 thread
def start(): Unit = thread.start()
// ControllerEventThread 继承了 ShutdownableThread，run方法会调用doWork
override def doWork(): Unit = {
  // 取出一个事件
  val dequeued = pollFromEventQueue()  
  dequeued.event match {  
    case ShutdownEventThread => // The shutting down of the thread has been initiated at this point. Ignore this event.  
    case controllerEvent =>  
      _state = controllerEvent.state   
      try {  
        def process(): Unit = dequeued.process(processor)  

		// 根据 state 调用process
        rateAndTimeMetrics.get(state) match {  
          case Some(timer) => timer.time { process() }  
          case None => process()  
        }  
      } 
      _state = ControllerState.Idle  
  }  
}
```
### 2.2.3 Controller 处理 Startup 事件
```scala
override def process(event: ControllerEvent): Unit = {  
  try {  
    event match {
	    case Startup =>  
			processStartup()
		}
	}
}

private def processStartup(): Unit = {
	// 注册 ControllerChangeHandler 监听器
		zkClient.registerZNodeChangeHandlerAndCheckExistence(controllerChangeHandler) 
	// 选举
	elect()  
}

// Controller 初始化时会创建一个 ControllerChangeHandler，用于处理 controller 的变化
class ControllerChangeHandler(eventManager: ControllerEventManager) extends ZNodeChangeHandler {
  override val path: String = ControllerZNode.path

  override def handleCreation(): Unit = eventManager.put(ControllerChange)
  // controller节点消失时，向eventManager中放入Reelect事件
  override def handleDeletion(): Unit = eventManager.put(Reelect)
  // 节点数据发生变化时，放入ControllerChange事件
  override def handleDataChange(): Unit = eventManager.put(ControllerChange)
}

// case Reelect => processReelect()
// case ControllerChange => processControllerChange()

private def processReelect(): Unit = {
    maybeResign()
    elect()
}

private def processControllerChange(): Unit = {
    maybeResign()
}

// 该broker之前是controller，需要先卸任，再重新竞选，否则直接竞选
private def maybeResign(): Unit = {
    // 判断之前是不是controller
    val wasActiveBeforeChange = isActive
    // 注册ControllerChangeHandler监听器
    zkClient.registerZNodeChangeHandlerAndCheckExistence(controllerChangeHandler)
    activeControllerId = zkClient.getControllerId.getOrElse(-1)
    // 之前是controller现在不是，需要卸任
    if (wasActiveBeforeChange && !isActive) {
        onControllerResignation()
    }
}

private def onControllerResignation(): Unit = {
    debug("Resigning")
    // de-register listeners
    // 取消ZooKeeper监听器的注册
    zkClient.unregisterZNodeChildChangeHandler(isrChangeNotificationHandler.path)
    zkClient.unregisterZNodeChangeHandler(partitionReassignmentHandler.path)
    zkClient.unregisterZNodeChangeHandler(preferredReplicaElectionHandler.path)
    zkClient.unregisterZNodeChildChangeHandler(logDirEventNotificationHandler.path)
    unregisterBrokerModificationsHandler(brokerModificationsHandlers.keySet)

    // shutdown leader rebalance scheduler
    // 关闭Kafka线程调度器，其实就是取消定期的Leader重选举
    // 将统计字段全部清0
    kafkaScheduler.shutdown()
    offlinePartitionCount = 0
    preferredReplicaImbalanceCount = 0
    globalTopicCount = 0
    globalPartitionCount = 0
    topicsToDeleteCount = 0
    replicasToDeleteCount = 0
    ineligibleTopicsToDeleteCount = 0
    ineligibleReplicasToDeleteCount = 0

    // stop token expiry check scheduler
    if (tokenCleanScheduler.isStarted)
      tokenCleanScheduler.shutdown()

    // de-register partition ISR listener for on-going partition reassignment task
    // 取消分区重分配监听器的注册
    unregisterPartitionReassignmentIsrChangeHandlers()
    // shutdown partition state machine
     // 关闭分区状态机
    partitionStateMachine.shutdown()
    // 取消主题变更监听器的注册
    zkClient.unregisterZNodeChildChangeHandler(topicChangeHandler.path)
    // 取消分区变更监听器的注册
    unregisterPartitionModificationsHandlers(partitionModificationsHandlers.keys.toSeq)
    // 取消主题删除监听器的注册
    zkClient.unregisterZNodeChildChangeHandler(topicDeletionHandler.path)
    // shutdown replica state machine
    replicaStateMachine.shutdown()
    zkClient.unregisterZNodeChildChangeHandler(brokerChangeHandler.path)

    controllerChannelManager.shutdown()
    // 清空集群元数据
    controllerContext.resetContext()
}
```
### 2.2.4 第一次选举
```scala
private def elect(): Unit = {
    activeControllerId = zkClient.getControllerId.getOrElse(-1)
    // 其他broker再获取controllerId时，此时不为-1直接返回
    if (activeControllerId != -1) {
      debug(s"Broker $activeControllerId has been elected as the controller, so stopping the election process.")
      return
    }
    try {
      // broker 向 controller 注册自己
      val (epoch, epochZkVersion) = zkClient.registerControllerAndIncrementControllerEpoch(config.brokerId)
      controllerContext.epoch = epoch
      controllerContext.epochZkVersion = epochZkVersion
      activeControllerId = config.brokerId
      
      onControllerFailover()
    }
}

def registerControllerAndIncrementControllerEpoch(controllerId: Int): (Int, Int) = {
	val (curEpoch, curEpochZkVersion) = getControllerEpoch
    .map(e => (e._1, e._2.getVersion))
    // 在zk上创建节点
    .getOrElse(maybeCreateControllerEpochZNode()
	// epoch + 1
    val newControllerEpoch = curEpoch + 1
    val expectedControllerEpochZkVersion = curEpochZkVersion
}
```
### 2.2.5 去 Zk 创建节点
```java
private def maybeCreateControllerEpochZNode(): (Int, Int) = {  
  createControllerEpochRaw(KafkaController.InitialControllerEpoch).resultCode match {...}

// ControllerEpochZNode.path = "/controller_epoch"
// 创建持久节点
def createControllerEpochRaw(epoch: Int): CreateResponse = {  
  val createRequest = CreateRequest(ControllerEpochZNode.path, ControllerEpochZNode.encode(epoch),  
    defaultAcls(ControllerEpochZNode.path), CreateMode.PERSISTENT)  
  retryRequestUntilConnected(createRequest)  
}

private def retryRequestsUntilConnected[Req <: AsyncRequest](requests: Seq[Req]): Seq[Req#Response] = {  
  while (remainingRequests.nonEmpty) {  
    val batchResponses = zooKeeperClient.handleRequests(remainingRequests)  
  responses  
}

// ZooKeeperClient 处理请求
def handleRequests[Req <: AsyncRequest](requests: Seq[Req]): Seq[Req#Response] = {  
  if (requests.isEmpty) 
  else {  
    requests.foreach { request =>  
      inFlightRequests.acquire()  
      try {  
        inReadLock(initializationLock) {  
          // 向 ZooKeeperClient 发送请求
          send(request) { response =>  
            responseQueue.add(response)  
            inFlightRequests.release()  
            countDownLatch.countDown()  
          }  
        }  
      } 
    }  
  }  
}

private[zookeeper] def send[Req <: AsyncRequest](request: Req)(processResponse: Req#Response => Unit): Unit = {  
  // ZooKeeperClient中创建了org.apache.zookeeper包中的zk对象
  // zooKeeper = new ZooKeeper
  request match {  
    // 创建节点请求
    case CreateRequest(path, data, acl, createMode, ctx) =>  
      zooKeeper.create(path, data, acl.asJava, createMode,  
        (rc, path, ctx, name) =>  
          callback(CreateResponse(Code.get(rc), path, Option(ctx), name, responseMetadata(sendTimeMs))),  
        ctx.orNull)  
  }  
}

// ZooKeeper.java
public void create(  
    final String path,  
    byte[] data,  
    List<ACL> acl,  
    CreateMode createMode,  
    StringCallback cb,  
    Object ctx) {  
    final String clientPath = path;  
    PathUtils.validatePath(clientPath, createMode.isSequential());  
    EphemeralType.validateTTL(createMode, -1);  
  
    final String serverPath = prependChroot(clientPath);  
    // 请求头
    RequestHeader h = new RequestHeader();  
    h.setType(createMode.isContainer() ? ZooDefs.OpCode.createContainer : ZooDefs.OpCode.create);  
    CreateRequest request = new CreateRequest();  
    CreateResponse response = new CreateResponse();  
    ReplyHeader r = new ReplyHeader();  
    request.setData(data);  
    request.setFlags(createMode.toFlag());  
    request.setPath(serverPath);  
    request.setAcl(acl);  
    // cnxn 发送 Packet
    cnxn.queuePacket(h, r, request, response, cb, clientPath, serverPath, ctx, null);  
}
```
## 2.3 启动 ReplicaManager
```java
// 开启两个定时任务：一个管理 isr 的过期情况(5s)，另一个广播 isr 的变化(2.5s)
def startup(): Unit = {  
  scheduler.schedule("isr-expiration", maybeShrinkIsr _, period = config.replicaLagTimeMaxMs / 2, unit = TimeUnit.MILLISECONDS)  
  // If using AlterIsr, we don't need the znode ISR propagation  
  if (!config.interBrokerProtocolVersion.isAlterIsrSupported) {  
    scheduler.schedule("isr-change-propagation", maybePropagateIsrChanges _,  
      period = isrChangeNotificationConfig.checkIntervalMs, unit = TimeUnit.MILLISECONDS)  
  } 
}
```
### 2.3.1 已过期的副本移除 isr 列表
#### 2.3.1.1 判断 isr 是否过期
```java
private def maybeShrinkIsr(): Unit = {  
  trace("Evaluating ISR list of partitions to see which replicas can be removed from the ISR")  
  
  // Shrink ISRs for non offline partitions  
  allPartitions.keys.foreach { topicPartition =>  
    nonOfflinePartition(topicPartition).foreach(_.maybeShrinkIsr())  
  }  
}

// Partition.scala
def maybeShrinkIsr(): Unit = {  
  val leaderHWIncremented = needsIsrUpdate && inWriteLock(leaderIsrUpdateLock) {  
    leaderLogIfLocal.exists { leaderLog =>  
      // 获取过期的 isr
      val outOfSyncReplicaIds = getOutOfSyncReplicas(replicaLagTimeMaxMs)  
      if (outOfSyncReplicaIds.nonEmpty) {  
        shrinkIsr(outOfSyncReplicaIds)  
        // we may need to increment high watermark since ISR could be down to 1  
        // 检查 HW 是否需要更新
        maybeIncrementLeaderHW(leaderLog)  
      }
    }  
  }  
}

def getOutOfSyncReplicas(maxLagMs: Long): Set[Int] = {  
  val current = isrState  
  if (!current.isInflight) {  
    // LEO
    val leaderEndOffset = localLogOrException.logEndOffset  
    candidateReplicaIds.filter(replicaId => isFollowerOutOfSync(replicaId, leaderEndOffset, currentTimeMs, maxLagMs))  
  }
}

private def isFollowerOutOfSync(replicaId: Int,  
                                leaderEndOffset: Long,  
                                currentTimeMs: Long,  
                                maxLagMs: Long): Boolean = {  
  val followerReplica = getReplicaOrException(replicaId)  
  // LEO 与 Leader 不同，且上次更新时间超过阈值(10s)
  followerReplica.logEndOffset != leaderEndOffset &&  
    (currentTimeMs - followerReplica.lastCaughtUpTimeMs) > maxLagMs  
}

```
#### 2.3.1.2 移除过期的 isr
```java
// 移除
private[cluster] def shrinkIsr(outOfSyncReplicas: Set[Int]): Unit = {  
	// 从 Set 中移除过期的 isr
    shrinkIsrWithZk(isrState.isr -- outOfSyncReplicas)  
  }  
}

private def shrinkIsrWithZk(newIsr: Set[Int]): Unit = {  
  val newLeaderAndIsr = new LeaderAndIsr(localBrokerId, leaderEpoch, newIsr.toList, zkVersion)  
  val zkVersionOpt = stateStore.shrinkIsr(controllerEpoch, newLeaderAndIsr)  
  if (zkVersionOpt.isDefined) { 
	// 记录 shrink 的操作 
    isrChangeListener.markShrink()  
  }  
  maybeUpdateIsrAndVersionWithZk(newIsr, zkVersionOpt)  
}

private def maybeUpdateIsrAndVersionWithZk(isr: Set[Int], zkVersionOpt: Option[Int]): Unit = {  
  zkVersionOpt match {  
    case Some(newVersion) =>  
      isrState = CommittedIsr(isr)  
      zkVersion = newVersion  
      info("ISR updated to [%s] and zkVersion updated to [%d]".format(isr.mkString(","), zkVersion))  
  }  
}
```
#### 2.3.1.3 判断是否需要更新 HW
```java
// 需要更新的情况: isr 变化或副本的 LEO 变化
private def maybeIncrementLeaderHW(leaderLog: Log, curTime: Long = time.milliseconds): Boolean = {  
  inReadLock(leaderIsrUpdateLock) {  
    // maybeIncrementLeaderHW is in the hot path, the following code is written to  
    // avoid unnecessary collection generation    
    var newHighWatermark = leaderLog.logEndOffsetMetadata  
    // 比较 leader 和 每个副本的 leo 的大小，计算出最小的作为新的 HW
    remoteReplicasMap.values.foreach { replica =>  
      // Note here we are using the "maximal", see explanation above  
      if (replica.logEndOffsetMetadata.messageOffset < newHighWatermark.messageOffset &&  
        (curTime - replica.lastCaughtUpTimeMs <= replicaLagTimeMaxMs || isrState.maximalIsr.contains(replica.brokerId))) {  
        newHighWatermark = replica.logEndOffsetMetadata  
      }  
    }  
  
    leaderLog.maybeIncrementHighWatermark(newHighWatermark) match { 
      // 更改成功后的回调 
      case Some(oldHighWatermark) =>  
        debug(s"High watermark updated from $oldHighWatermark to $newHighWatermark")  
        true  
    }  
  }  
}

def maybeIncrementHighWatermark(newHighWatermark: LogOffsetMetadata): Option[LogOffsetMetadata] = {   
  lock.synchronized {  
    val oldHighWatermark = fetchHighWatermarkMetadata  
   
    if (oldHighWatermark.messageOffset < newHighWatermark.messageOffset ||  
      (oldHighWatermark.messageOffset == newHighWatermark.messageOffset && oldHighWatermark.onOlderSegment(newHighWatermark))) {  
      updateHighWatermarkMetadata(newHighWatermark)  
      Some(oldHighWatermark)  
    }
  }  
}
```
### 2.3.2 有新的 follower 时，更新 isr 集合
```java
// ReplicaManager 中定义了 ReplicaFetcherManager
protected def createReplicaFetcherManager(metrics: Metrics, time: Time, threadNamePrefix: Option[String], quotaManager: ReplicationQuotaManager) = {  
  new ReplicaFetcherManager(config, this, metrics, time, threadNamePrefix, quotaManager)  
}

// ReplicaFetcherManager 中定义了 ReplicaFetcherThread 对象
// 调用 ReplicaManager 中的 fetchMessages 方法
def fetchMessages(...): Unit = {  
  val isFromFollower = Request.isValidBrokerId(replicaId)  
  
  val fetchOnlyFromLeader = isFromFollower || (isFromConsumer && clientMetadata.isEmpty)  
  def readFromLog(): Seq[(TopicPartition, LogReadResult)] = {  
    val result = readFromLocalLog(...)
    // 如果是 Follower 的 fetch 请求, 更新远程的状态  
    if (isFromFollower) updateFollowerFetchState(replicaId, result)  
    else result  
  }  
}

private def updateFollowerFetchState(followerId: Int,  
                                     readResults: Seq[(TopicPartition, LogReadResult)]): Seq[(TopicPartition, LogReadResult)] = {
	 
  nonOfflinePartition(topicPartition) match {  
	case Some(partition) =>  
	  if (partition.updateFollowerFetchState(followerId,  
		followerFetchOffsetMetadata = readResult.info.fetchOffsetMetadata,  
		followerStartOffset = readResult.followerLogStartOffset,  
		followerFetchTimeMs = readResult.fetchTimeMs,  
		leaderEndOffset = readResult.leaderLogEndOffset)) {  
		readResult  
		}
	 }
}

// Partition.scala
def updateFollowerFetchState(followerId: Int,  
		followerFetchOffsetMetadata: LogOffsetMetadata,  
		 followerStartOffset: Long,  
		 followerFetchTimeMs: Long,  
		 leaderEndOffset: Long): Boolean = { 
      maybeExpandIsr(followerReplica, followerFetchTimeMs) 
}

private def maybeExpandIsr(followerReplica: Replica, followerFetchTimeMs: Long): Unit = {  
  if (needsIsrUpdate) {  
    inWriteLock(leaderIsrUpdateLock) {  
      // check if this replica needs to be added to the ISR  
      if (needsExpandIsr(followerReplica)) {  
        expandIsr(followerReplica.brokerId)  
      }  
    }  
  }  
}

// LEO >= HW
private def needsExpandIsr(followerReplica: Replica): Boolean = {  
  canAddReplicaToIsr(followerReplica.brokerId) && isFollowerAtHighwatermark(followerReplica)  
}

private def isFollowerAtHighwatermark(followerReplica: Replica): Boolean = {  
  leaderLogIfLocal.exists { leaderLog =>  
    val followerEndOffset = followerReplica.logEndOffset  
    followerEndOffset >= leaderLog.highWatermark && leaderEpochStartOffsetOpt.exists(followerEndOffset >= _)  
  }  
}

private[cluster] def expandIsr(newInSyncReplica: Int): Unit = {  
  if (useAlterIsr) {  
    expandIsrWithAlterIsr(newInSyncReplica)  
  } else {  
    expandIsrWithZk(newInSyncReplica)  
  }  
}
```
### 2.3.3 广播 isr 变化
```java
def maybePropagateIsrChanges(): Unit = {  
  val now = time.milliseconds()  
  isrChangeSet synchronized {  
    if (isrChangeSet.nonEmpty &&  
      (lastIsrChangeMs.get() + isrChangeNotificationConfig.lingerMs < now ||  
        lastIsrPropagationMs.get() + isrChangeNotificationConfig.maxDelayMs < now)) {  
      // 非空则广播变化
      zkClient.propagateIsrChanges(isrChangeSet)  
      isrChangeSet.clear()  
      lastIsrPropagationMs.set(now)  
    }  
  }  
}
```
#### 2.3.3.1 shrink 操作
```java
// 2.3.1.2
// 通知 ReplicaManager isr 的 shrink
override def markShrink(): Unit = {  
  replicaManager.recordIsrChange(topicPartition)  
  replicaManager.isrShrinkRate.mark()  
}

def recordIsrChange(topicPartition: TopicPartition): Unit = {  
  if (!config.interBrokerProtocolVersion.isAlterIsrSupported) {  
    isrChangeSet synchronized {  
      // 加入到 isrChangeSet 中
      isrChangeSet += topicPartition  
      // 更新变更时间
      lastIsrChangeMs.set(time.milliseconds())  
    }  
  }  
}
```
#### 2.3.3.2 expand 操作
```java
override def markExpand(): Unit = {  
  replicaManager.recordIsrChange(topicPartition)  
  replicaManager.isrExpandRate.mark()  
}
```
## 2.4 创建启动 LogManager
- LogManager 负责日志的创建、检索、清理
- Kafka 中的日志对应多个 Segment
- 每个 Segment 对应
	- 一个 .log 文件
	- 一个 .index 文件
	- 一个 .timeindex 文件
- Segment 分段时机
	- `Segment` 中消息的最大时间戳与当前系统的时间戳的差值 > `log.roll.ms`
	- 当 `Segment` 的大小超过了 `log.segment.bytes` 默认 `1G`
	- 偏移量索引文件或时间戳索引文件的大小达到了 `log.index.size.max.bytes 默认 10MB`
- 日志查找步骤
	- 根据时间戳
		- 先去 timeindex 文件中找不大于时间戳的最大偏移量
		- 根据偏移量去偏移量索引中找到不大于该值的最大偏移量对应的物理文件位置
		- 去物理文件中找对应消息
	- 根据偏移量
### 2.4.1 创建 LogManager
```java
logManager = LogManager(...)

class LogManager(logDirs: Seq[File],  
                 initialOfflineDirs: Seq[File],  
                 val topicConfigs: Map[String, LogConfig], // note that this doesn't get updated after creation  
                 val initialDefaultConfig: LogConfig,  
                 val cleanerConfig: CleanerConfig,  
                 recoveryThreadsPerDataDir: Int,  
                 val flushCheckMs: Long,  
                 val flushRecoveryOffsetCheckpointMs: Long,  
                 val flushStartOffsetCheckpointMs: Long,  
                 val retentionCheckMs: Long,  
                 val maxPidExpirationMs: Int,  
                 scheduler: Scheduler,  
                 val brokerState: BrokerState,  
                 brokerTopicStats: BrokerTopicStats,  
                 logDirFailureChannel: LogDirFailureChannel,  
                 time: Time) extends Logging with KafkaMetricsGroup { 
  // 创建有效的 log 目录
  private val _liveLogDirs: ConcurrentLinkedQueue[File] = createAndValidateLogDirs(logDirs, initialOfflineDirs)  
	//恢复并且载入所给定目录的log对象，如果有子目录，子目录中的文件就是Segment
    loadLogs()
}
```
### 2.4.2 载入 log
```java
private def loadLogs(): Unit = {  
  val threadPools = ArrayBuffer.empty[ExecutorService]
  // 遍历 liveLogDirs，每一个 dir 产生一个线程池
  for (dir <- liveLogDirs) {  
    val logDirAbsolutePath = dir.getAbsolutePath  
    try {  
      // 添加到池中
      val pool = Executors.newFixedThreadPool(numRecoveryThreadsPerDataDir)  
      threadPools.append(pool)  
      val jobsForDir = logsToLoad.map { logDir =>  
        val runnable: Runnable = () => {  
          try {  
  
            val logLoadStartMs = time.hiResClockMs()  
            // 加载 log
            val log = loadLog(logDir, recoveryPoints, logStartOffsets)  
          }  
        }  
        runnable  
      }  
	  // 向线程池提交执行 Runable 对象
      jobs(cleanShutdownFile) = jobsForDir.map(pool.submit)  
    }
  }  
}

private def loadLog(logDir: File,  
                    recoveryPoints: Map[TopicPartition, Long], 
                    logStartOffsets: Map[TopicPartition, Long]): Log = { 
  // 从目录路径可以解析出主题分区信息                   
  val topicPartition = Log.parseTopicPartitionName(logDir)  
  // 创建 Log 对象
  val log = Log(...)  
  log  
}

def apply(dir: File,  
          config: LogConfig,  
          logStartOffset: Long,  
          recoveryPoint: Long,  
          scheduler: Scheduler,  
          brokerTopicStats: BrokerTopicStats,  
          time: Time = Time.SYSTEM,  
          maxProducerIdExpirationMs: Int,  
          producerIdExpirationCheckIntervalMs: Int,  
          logDirFailureChannel: LogDirFailureChannel): Log = {  
  val topicPartition = Log.parseTopicPartitionName(dir)  
  val producerStateManager = new ProducerStateManager(topicPartition, dir, maxProducerIdExpirationMs)  
  new Log(...)  
}

class Log(...) extends Logging with KafkaMetricsGroup {
	locally {  
  // 加载 Segment
  val nextOffset = loadSegments()
	}
}

private def loadSegments(): Long = {  
  // 首先移除交换文件 
  val swapFiles = removeTempFilesAndCollectSwapFiles()  
  
  // 二次读取
  retryOnOffsetOverflow {     
    logSegments.foreach(_.close())  
    segments.clear()  
    loadSegmentFiles()  
  }  
}

private def loadSegmentFiles(): Unit = {  
  // load segments in ascending order because transactional data from one segment may depend on the  
  // segments that come before it  
  for (file <- dir.listFiles.sortBy(_.getName) if file.isFile) { 
    // .index
    if (isIndexFile(file)) {  
      // if it is an index file, make sure it has a corresponding .log file  
      val offset = offsetFromFile(file)  
      val logFile = Log.logFile(dir, offset)   
    } else if (isLogFile(file)) {  
      // if it's a log file, load the corresponding log segment  
      val baseOffset = offsetFromFile(file)  
      val timeIndexFileNewlyCreated = !Log.timeIndexFile(dir, baseOffset).exists()  
      val segment = LogSegment.open(dir = dir,  
        baseOffset = baseOffset,  
        config,  
        time = time,  
        fileAlreadyExists = true)  
      // 添加到 ConcurrentSkipListMap 中
      addSegment(segment)  
    }  
  }  
}
```
### 2.4.3 载入 LogSegment
```java
// LogSegment.scala
def open(dir: File, baseOffset: Long, config: LogConfig, time: Time, fileAlreadyExists: Boolean = false,  
         initFileSize: Int = 0, preallocate: Boolean = false, fileSuffix: String = ""): LogSegment = {  
  new LogSegment(...)  
}

// LogSegment 在调用 append 方法时，会判断是否需要 roll
private def append(records: MemoryRecords,  
                   origin: AppendOrigin,  
                   interBrokerProtocolVersion: ApiVersion,  
                   assignOffsets: Boolean,  
                   leaderEpoch: Int,  
                   ignoreRecordSize: Boolean): LogAppendInfo = {
	lock synchronized {
		val segment = maybeRoll(validRecords.sizeInBytes, appendInfo)
	}
}

private def maybeRoll(messagesSize: Int, appendInfo: LogAppendInfo): LogSegment = {  
  val segment = activeSegment  
  val now = time.milliseconds  
  
  val maxTimestampInMessages = appendInfo.maxTimestamp  
  val maxOffsetInMessages = appendInfo.lastOffset  
  
  if (segment.shouldRoll(RollParams(config, appendInfo, messagesSize, now))) {  
		appendInfo.firstOffset match {  
	      case Some(firstOffset) => roll(Some(firstOffset))  
	      case None => roll(Some(maxOffsetInMessages - Integer.MAX_VALUE))  
    }  
  } else {  
    segment  
  }  
}

// Segment 字节数 > config.segmentSize - messagesSize
// time.milliseconds - segment.created > config.segmentMs - segment.rollJitterMs
// 超过 maxIndexSize
def shouldRoll(rollParams: RollParams): Boolean = {  
  val reachedRollMs = timeWaitedForRoll(rollParams.now, rollParams.maxTimestampInMessages) > rollParams.maxSegmentMs - rollJitterMs  
  size > rollParams.maxSegmentBytes - rollParams.messagesSize ||  
    (size > 0 && reachedRollMs) ||  
    offsetIndex.isFull || timeIndex.isFull || !canConvertToRelativeOffset(rollParams.maxOffsetInMessages)  
}
```
### 2.4.5 启动 LogManager
```java
def startup(): Unit = {  
  /* Schedule the cleanup task to delete old logs */  
  if (scheduler != null) {  
    // 定时清理过期的日志，维护日志大小
    scheduler.schedule("kafka-log-retention",  
                       cleanupLogs _,  
                       delay = InitialTaskDelayMs,  
                       period = retentionCheckMs,  
                       TimeUnit.MILLISECONDS)  
    // 定时刷写还没有写到磁盘上的日志
    scheduler.schedule("kafka-log-flusher",  
                       flushDirtyLogs _,  
                       delay = InitialTaskDelayMs,  
                       period = flushCheckMs,  
                       TimeUnit.MILLISECONDS)  
    scheduler.schedule("kafka-recovery-point-checkpoint",  
                       checkpointLogRecoveryOffsets _,  
                       delay = InitialTaskDelayMs,  
                       period = flushRecoveryOffsetCheckpointMs,  
                       TimeUnit.MILLISECONDS)  
    scheduler.schedule("kafka-log-start-offset-checkpoint",  
                       checkpointLogStartOffsets _,  
                       delay = InitialTaskDelayMs,  
                       period = flushStartOffsetCheckpointMs,  
                       TimeUnit.MILLISECONDS)  
    // 定时删除标记为 delete 的日志文件
    scheduler.schedule("kafka-delete-logs", // will be rescheduled after each delete logs with a dynamic period  
                       deleteLogs _,  
                       delay = InitialTaskDelayMs,  
                       unit = TimeUnit.MILLISECONDS)  
  }  
  if (cleanerConfig.enableCleaner)  
    cleaner.startup()  
}
```
#### 2.4.5.1 清理日志
```java
def cleanupLogs(): Unit = {  
  debug("Beginning log cleanup...")  
  try {  
    deletableLogs.foreach {  
      case (topicPartition, log) =>  
        debug(s"Garbage collecting '${log.name}'")  
        // 通过 deleteOldSegments 删除日志
        total += log.deleteOldSegments()  
    }  
  }  
}

def deleteOldSegments(): Int = {  
  if (config.delete) {  
     // 删除到达保存时间阈值的
    deleteRetentionMsBreachedSegments() +    
    // 删除到达保存大小阈值的
    deleteRetentionSizeBreachedSegments() + 
    // 删除到达 offset 阈值的
    deleteLogStartOffsetBreachedSegments()  
  }
}
// Log.scala
// 删除超过 retention.ms 的日志
private def deleteRetentionMsBreachedSegments(): Int = {  
  val startMs = time.milliseconds  
  
  def shouldDelete(segment: LogSegment, nextSegmentOpt: Option[LogSegment]): Boolean = {  
    startMs - segment.largestTimestamp > config.retentionMs  
  }  
  
  deleteOldSegments(shouldDelete, RetentionMsBreach)  
}

// 删除超过 retention.bytes 的日志
private def deleteRetentionSizeBreachedSegments(): Int = {  
  var diff = size - config.retentionSize  
  def shouldDelete(segment: LogSegment, nextSegmentOpt: Option[LogSegment]): Boolean = {  
    if (diff - segment.size >= 0) {  
      diff -= segment.size  
      true  
    } else {  
      false  
    }  
  }  
  deleteOldSegments(shouldDelete, RetentionSizeBreach)  
}

// delete any log segments that are before the log start offset
private def deleteLogStartOffsetBreachedSegments(): Int = {  
  def shouldDelete(segment: LogSegment, nextSegmentOpt: Option[LogSegment]): Boolean = {  
    nextSegmentOpt.exists(_.baseOffset <= logStartOffset)  
  }  
  deleteOldSegments(shouldDelete, StartOffsetBreach)  
}
```
#### 2.4.5.2 刷写日志
```java
private def flushDirtyLogs(): Unit = {  
  
  for ((topicPartition, log) <- currentLogs.toList ++ futureLogs.toList) {  
    try {  
      val timeSinceLastFlush = time.milliseconds - log.lastFlushTime  
	  // 超过 flush.ms 则刷写
      if(timeSinceLastFlush >= log.config.flushMs)  
        log.flush()  
    } 
  }  
}

// Log.scala
def flush(): Unit = flush(this.logEndOffset)

def flush(offset: Long): Unit = {  
  maybeHandleIOException(s"Error while flushing log for $topicPartition in dir ${dir.getParent} with offset $offset") {  
	// 调用 Segment 的 flush 方法
    for (segment <- logSegments(this.recoveryPoint, offset))  
      segment.flush()  
    }  
}
```
## 2.5 Controller 选举 Partition 的 leader
### 2.5.1 Partition 的状态
```java
// PartitionStateMachine 中定义了 Partition 的状态
sealed trait PartitionState {  
  def state: Byte  
  // 有效的上个状态
  def validPreviousStates: Set[PartitionState]  
}

// 正在创建新的分区，只在 Controller 中保存了状态信息
case object NewPartition extends PartitionState {  
  val state: Byte = 0  
  val validPreviousStates: Set[PartitionState] = Set(NonExistentPartition)  
}

// 可提供服务的状态
case object OnlinePartition extends PartitionState {  
  val state: Byte = 1  
  val validPreviousStates: Set[PartitionState] = Set(NewPartition, OnlinePartition, OfflinePartition)  
}

// Broker 宕机或要删除 Topic
case object OfflinePartition extends PartitionState {  
  val state: Byte = 2  
  val validPreviousStates: Set[PartitionState] = Set(NewPartition, OnlinePartition, OfflinePartition)  
}

// Topic 删除成功
case object NonExistentPartition extends PartitionState {  
  val state: Byte = 3  
  val validPreviousStates: Set[PartitionState] = Set(OfflinePartition)  
}
```
### 2.5.2 处理分区状态变化
```java
private def doHandleStateChanges(  
  partitions: Seq[TopicPartition],  
  targetState: PartitionState,  
  partitionLeaderElectionStrategyOpt: Option[PartitionLeaderElectionStrategy]  
): Map[TopicPartition, Either[Throwable, LeaderAndIsr]] = {  

  targetState match {  
    case NewPartition =>  
        controllerContext.putPartitionState(partition, NewPartition)  
      }  
      Map.empty  
    case OnlinePartition =>  
      // NewPartition -> OnlinePartition 初始化
      val uninitializedPartitions = validPartitions.filter(partition => partitionState(partition) == NewPartition)  
      // OfflinePartition | OnlinePartition -> OnlinePartition 选举
      val partitionsToElectLeader = validPartitions.filter(partition => partitionState(partition) == OfflinePartition || partitionState(partition) == OnlinePartition)  
      if (uninitializedPartitions.nonEmpty) {  
        val successfulInitializations = initializeLeaderAndIsrForPartitions(uninitializedPartitions)  
        successfulInitializations.foreach { partition =>  
          controllerContext.putPartitionState(partition, OnlinePartition)  
        }  
      }  
      if (partitionsToElectLeader.nonEmpty) {  
        val electionResults = electLeaderForPartitions(  
          partitionsToElectLeader,  
          partitionLeaderElectionStrategyOpt.getOrElse(  
            throw new IllegalArgumentException("Election strategy is a required field when the target state is OnlinePartition")  
          )  
        )  
  
        electionResults.foreach {  
          case (partition, Right(leaderAndIsr)) =>  
            controllerContext.putPartitionState(partition, OnlinePartition)  
          case (_, Left(_)) => // Ignore; no need to update partition state on election error  
        } 
        electionResults  
      } else {  
        Map.empty  
      }  
    // 下线或删除只需向集合加入新状态即可
    case OfflinePartition =>  
      validPartitions.foreach { partition =>  
        controllerContext.putPartitionState(partition, OfflinePartition)  
      }  
      Map.empty  
    case NonExistentPartition =>  
      validPartitions.foreach { partition =>  
        controllerContext.putPartitionState(partition, NonExistentPartition)  
      }  
      Map.empty  
  }  
}
```
### 2.5.3 初始化 Partition 的 leader
```java
private def initializeLeaderAndIsrForPartitions(partitions: Seq[TopicPartition]): Seq[TopicPartition] = {  
  
  val leaderIsrAndControllerEpochs = partitionsWithLiveReplicas.map { case (partition, liveReplicas) =>  
    // liveReplicas 的第一个作为 leader，liveReplicas 作为 isr
    val leaderAndIsr = LeaderAndIsr(liveReplicas.head, liveReplicas.toList)  
    val leaderIsrAndControllerEpoch = LeaderIsrAndControllerEpoch(leaderAndIsr, controllerContext.epoch)  
    partition -> leaderIsrAndControllerEpoch  
  }.toMap  
// 写入 zk 节点中 /brokers/topics/{topic_name}/partitions/{partition_num}/state
  val createResponses = try {  
zkClient.createTopicPartitionStatesRaw(leaderIsrAndControllerEpochs, controllerContext.epochZkVersion)  
  }  
  successfulInitializations  
}
```
### 2.5.4 leader 宕机重选举
- 状态机中定义了选举策略 `
```java
sealed trait PartitionLeaderElectionStrategy  
final case class OfflinePartitionLeaderElectionStrategy(allowUnclean: Boolean) extends PartitionLeaderElectionStrategy  
final case object ReassignPartitionLeaderElectionStrategy extends PartitionLeaderElectionStrategy  
final case object PreferredReplicaPartitionLeaderElectionStrategy extends PartitionLeaderElectionStrategy  
final case object ControlledShutdownPartitionLeaderElectionStrategy extends PartitionLeaderElectionStrategy

// Broker 下线时，会调用 controlledShutdown 方法
private def controlledShutdown(): Unit = {  
  
  def doControlledShutdown(retries: Int): Boolean = {   
    try {       
        // 1. 获取到一个 Controller 
        zkClient.getControllerId match {  
          case Some(controllerId) =>  
            zkClient.getBroker(controllerId) match {  
              case Some(broker) =>  
                  prevController = broker  
                }  
        }  
  
        // 2. issue a controlled shutdown to the controller  
        if (prevController != null) {  
          try {  
            // send the controlled shutdown request  
            val controlledShutdownRequest = new ControlledShutdownRequest.Builder(...)  
            // 向 Controller 发送停止请求
            val request = networkClient.newClientRequest(node(prevController).idString, controlledShutdownRequest,  
              time.milliseconds(), true)  
            val clientResponse = NetworkClientUtils.sendAndReceive(networkClient, request, time)  
    }  
    finally  
      networkClient.close()  
    shutdownSucceeded  
  }   
}

// Controller 进行处理
override def process(event: ControllerEvent): Unit = {  
  try {  
    event match {
	    case ControlledShutdown(id, brokerEpoch, callback) =>  
		  processControlledShutdown(id, brokerEpoch, callback)
	  }
}

private def processControlledShutdown(id: Int, brokerEpoch: Long, controlledShutdownCallback: Try[Set[TopicPartition]] => Unit): Unit = { 
  val controlledShutdownResult = Try { doControlledShutdown(id, brokerEpoch) }  
  controlledShutdownCallback(controlledShutdownResult)  
}

private def doControlledShutdown(id: Int, brokerEpoch: Long): Set[TopicPartition] = {
	partitionStateMachine.handleStateChanges(partitionsLedByBroker.toSeq, OnlinePartition, Some(ControlledShutdownPartitionLeaderElectionStrategy))
}

private def doHandleStateChanges(  
  partitions: Seq[TopicPartition],  
  targetState: PartitionState,  
  partitionLeaderElectionStrategyOpt: Option[PartitionLeaderElectionStrategy]  
): Map[TopicPartition, Either[Throwable, LeaderAndIsr]] = {  
  
  targetState match {  
    case OnlinePartition =>  
      val partitionsToElectLeader = validPartitions.filter(partition => partitionState(partition) == OfflinePartition || partitionState(partition) == OnlinePartition)  
      if (partitionsToElectLeader.nonEmpty) {
        // 选举  
        val electionResults = electLeaderForPartitions(  
          partitionsToElectLeader,  
          partitionLeaderElectionStrategyOpt.getOrElse(  
            throw new IllegalArgumentException("Election strategy is a required field when the target state is OnlinePartition")  
          )  
        )   
        electionResults  
      } else {  
        Map.empty  
      }  
  }  
}

private def electLeaderForPartitions(  
  partitions: Seq[TopicPartition],  
  partitionLeaderElectionStrategy: PartitionLeaderElectionStrategy  
): Map[TopicPartition, Either[Throwable, LeaderAndIsr]] = {  
  var remaining = partitions  
  val finishedElections = mutable.Map.empty[TopicPartition, Either[Throwable, LeaderAndIsr]]  
  
  while (remaining.nonEmpty) {  
    val (finished, updatesToRetry) = doElectLeaderForPartitions(remaining, partitionLeaderElectionStrategy)  
    remaining = updatesToRetry  
  finishedElections.toMap  
}

private def doElectLeaderForPartitions(...) = {
	// 去 zk 获取当前 partition 的 isr 和 leader 信息，封装进validLeaderAndIsrs
	val getDataResponses = try {  
  zkClient.getTopicPartitionStatesRaw(partitions)
  }
	val (partitionsWithoutLeaders, partitionsWithLeaders) = partitionLeaderElectionStrategy match {  
  case ControlledShutdownPartitionLeaderElectionStrategy =>  
    leaderForControlledShutdown(controllerContext, validLeaderAndIsrs).partition(_.leaderAndIsr.isEmpty)  
}

def leaderForControlledShutdown(controllerContext: ControllerContext,  
	leaderAndIsrs: Seq[(TopicPartition, LeaderAndIsr)]): 
	Seq[ElectionResult] = {  
	
  leaderAndIsrs.map { case (partition, leaderAndIsr) =>  
    leaderForControlledShutdown(partition, leaderAndIsr, shuttingDownBrokerIds, controllerContext)  
  }  
}

private def leaderForControlledShutdown(partition: TopicPartition,  
	leaderAndIsr: LeaderAndIsr,  
	shuttingDownBrokerIds: Set[Int],  
	controllerContext: ControllerContext): ElectionResult = {  
	  
	  val isr = leaderAndIsr.isr  
	  val leaderOpt = PartitionLeaderElectionAlgorithms.controlledShutdownPartitionLeaderElection(assignment, 
	  isr,
	  liveOrShuttingDownReplicas.toSet, shuttingDownBrokerIds)  
  val newIsr = isr.filter(replica => !shuttingDownBrokerIds.contains(replica))  
  val newLeaderAndIsrOpt = leaderOpt.map(leader => leaderAndIsr.newLeaderAndIsr(leader, newIsr))  
  ElectionResult(partition, newLeaderAndIsrOpt, liveOrShuttingDownReplicas)  
}

// 找到第一个在 AR 中且在 isr 中的 partition 作为 leader
def controlledShutdownPartitionLeaderElection(assignment: Seq[Int], isr: Seq[Int], liveReplicas: Set[Int], shuttingDownBrokers: Set[Int]): Option[Int] = {  
  assignment.find(id => liveReplicas.contains(id) && isr.contains(id) && !shuttingDownBrokers.contains(id))  
}
```
# 3 Consumer
- 基本架构
	- SubscriptionState：管理消费者订阅的主题分区，记录消费的各种状态
	- ConsumerCoordinator：负责和 `Server` 的 `GroupCoordinator` 通讯
	- Fetcher：负责发送拉取消息的请求
	- PartitionAssignor：分区分配策略
	- ConsumerNetworkClient：负责消费者的网络 IO，在 `NetworkClient` 上进行封装

![[kafka_consumer.svg]]

## 3.1 fields
```java
public class KafkaConsumer<K, V> implements Consumer<K, V> {
	private final String clientId;  
	private final Optional<String> groupId;  
	// 与 Server 的 coordinator 交互
	private final ConsumerCoordinator coordinator;  
	private final Deserializer<K> keyDeserializer;  
	private final Deserializer<V> valueDeserializer;  
	// 拉取数据
	private final Fetcher<K, V> fetcher;  
	private final ConsumerInterceptors<K, V> interceptors;
	// 保存消费信息
	private final SubscriptionState subscriptions;  
	private final ConsumerMetadata metadata;
	// 分配消费分区
	private List<ConsumerPartitionAssignor> assignors
}
```
## 3.2 初始化
```java
KafkaConsumer(ConsumerConfig config, Deserializer<K> keyDeserializer, Deserializer<V> valueDeserializer) {  
    try {
		this.groupId = Optional.ofNullable(groupRebalanceConfig.groupId);  
		this.clientId = config.getString(CommonClientConfigs.CLIENT_ID_CONFIG);
		// 拦截器
		// 序列化器
		OffsetResetStrategy offsetResetStrategy = OffsetResetStrategy.valueOf(config.getString(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG).toUpperCase(Locale.ROOT));  
		this.subscriptions = new SubscriptionState(logContext, offsetResetStrategy);
		this.metadata = new ConsumerMetadata(...);
	    NetworkClient netClient = new NetworkClient(...);  
		this.client = new ConsumerNetworkClient(...);
		// partition.assignment.strategy, 默认 RangeAssignor
		this.assignors = ConsumerPartitionAssignor.getAssignorInstances(  
	    config.getList(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG), config.originals(Collections.singletonMap(ConsumerConfig.CLIENT_ID_CONFIG, clientId)) 
	);

		if (!groupId.isPresent()) {...} 
		// 指定 groupId 时会创建 ConsumerCoordinator
		else {  
		    this.coordinator = new ConsumerCoordinator(...);  

		this.fetcher = new Fetcher<>(...);
}	
```
## 3.3 消费者组重平衡
- 触发重平衡的条件
	- 消费者组内消费者数量变化
	- 订阅 `Topic` 发生变化
	- 消费者组对应 `GroupCoordinator` 组件所在 `Broker` 变化
### 3.3.1 消费者组状态

![[consumer_status.svg]]

```java
// GroupState 定义了消费者组的状态空间，包含一个检验前置状态的方法
private[group] case object Dead extends GroupState {  
  val validPreviousStates: Set[GroupState] = Set(Stable, PreparingRebalance, CompletingRebalance, Empty, Dead)  
}

// 正在执行加入组操作的消费者组
private[group] case object PreparingRebalance extends GroupState {  
  val validPreviousStates: Set[GroupState] = Set(Stable, CompletingRebalance, Empty)  
}

// 等待 leader 成员指定分配方案的消费者组
private[group] case object CompletingRebalance extends GroupState {  
  val validPreviousStates: Set[GroupState] = Set(PreparingRebalance)  
}

// 已完成 Rebalance 可以正常工作的消费者组
private[group] case object Stable extends GroupState {  
  val validPreviousStates: Set[GroupState] = Set(CompletingRebalance)  
}

// 没有成员且元数据信息被删除的消费者组
private[group] case object Dead extends GroupState {  
  val validPreviousStates: Set[GroupState] = Set(Stable, PreparingRebalance, CompletingRebalance, Empty, Dead)  
}
```
### 3.3.2 ConsumerCoordinator 寻找 GroupCoordinator
```java
// ConsumerCoordinator.java
public boolean poll(Timer timer, boolean waitForJoinGroup) {  

// 判断是否为订阅Topic或者订阅pattern模式
    if (subscriptions.hasAutoAssignedPartitions()) {  
	    // 检查心跳
		pollHeartbeat(timer.currentTimeMs());  
		// 检查是否需要加入消费者组
        if (rejoinNeededOrPending()) {  
			// 确保消费者组处于 active
            if (!ensureActiveGroup(waitForJoinGroup ? timer : time.timer(0L))) {  
                return false;  
            }  
        }  
    }
    return true;  
}

boolean ensureActiveGroup(final Timer timer) {   
	// 确保 ConsumerCoordinator能够正常接收服务端请求
    if (!ensureCoordinatorReady(timer)) {  
        return false;  
    }  
    // 启动心跳线程
    startHeartbeatThreadIfNeeded();  
    // 发送 joinGroup 请求
    return joinGroupIfNeeded(timer);  
}

// AbstractCoordinator.java
private synchronized boolean ensureCoordinatorReady(final Timer timer, boolean disableWakeup) {  
	// 找到了 coordinator 直接返回
    if (!coordinatorUnknown())  
        return true;  
  
    do {  
	    // 循环给服务器端发送 FindCoordinatorRequest 请求，直到找到 coordinator
        final RequestFuture<Void> future = lookupCoordinator();  
        // 阻塞等待 Broker 请求返回或超时
        client.poll(future, timer, disableWakeup);  
    } while (coordinatorUnknown() && timer.notExpired());  
    return !coordinatorUnknown();  
}

protected synchronized RequestFuture<Void> lookupCoordinator() {  
    if (findCoordinatorFuture == null) {  
        // 找负载最小的节点
        Node node = this.client.leastLoadedNode();  
        if (node == null) {...} 
        else {  
            findCoordinatorFuture = sendFindCoordinatorRequest(node);  
        }  
    }  
    return findCoordinatorFuture;  
}

private RequestFuture<Void> sendFindCoordinatorRequest(Node node) {   
    FindCoordinatorRequest.Builder requestBuilder = new FindCoordinatorRequest.Builder(data);  
    return client.send(node, requestBuilder)  
            .compose(new FindCoordinatorResponseHandler());  
}
```
### 3.3.3 Server 端处理请求
```java
override def handle(request: RequestChannel.Request): Unit = {  
  try {
		request.header.apiKey match {
			case ApiKeys.FIND_COORDINATOR => handleFindCoordinatorRequest(request)
			}
	}
}

def handleFindCoordinatorRequest(request: RequestChannel.Request): Unit = {  

    // get metadata (and create the topic if necessary)  
    val (partition, topicMetadata) = CoordinatorType.forId(findCoordinatorRequest.data.keyType) match {  
      case CoordinatorType.GROUP =>  
	    // 确定 GroupCoordinator 在哪个 Broker
        val partition = groupCoordinator.partitionFor(findCoordinatorRequest.data.key)  
        // 获取对应的元数据
        val metadata = getOrCreateInternalTopic(GROUP_METADATA_TOPIC_NAME, request.context.listenerName)  
        (partition, metadata)  
  
    def createResponse(requestThrottleMs: Int): AbstractResponse = {  
      def createFindCoordinatorResponse(error: Errors, node: Node): FindCoordinatorResponse = {  
        new FindCoordinatorResponse(  
            new FindCoordinatorResponseData()  
              .setErrorCode(error.code)  
              .setErrorMessage(error.message)  
              // 返回 node.id
              .setNodeId(node.id)  
              // 返回 node.host
              .setHost(node.host)  
              // 返回 node.port
              .setPort(node.port)  
              .setThrottleTimeMs(requestThrottleMs))  
      }  
      val responseBody = if (topicMetadata.errorCode != Errors.NONE.code) {  
        createFindCoordinatorResponse(Errors.COORDINATOR_NOT_AVAILABLE, Node.noNode)  
      } else {  
        val coordinatorEndpoint = topicMetadata.partitions.asScala  
          .find(_.partitionIndex == partition)  
          .filter(_.leaderId != MetadataResponse.NO_LEADER_ID)  
          .flatMap(metadata => metadataCache.getAliveBroker(metadata.leaderId))  
          .flatMap(_.getNode(request.context.listenerName))  
          .filterNot(_.isEmpty)  
  
        coordinatorEndpoint match {  
          case Some(endpoint) =>  
            // 调用上面定义的方法
            createFindCoordinatorResponse(Errors.NONE, endpoint)  
          case _ =>  
            createFindCoordinatorResponse(Errors.COORDINATOR_NOT_AVAILABLE, Node.noNode)  
        }  
      }   
      responseBody  
    }  
    sendResponseMaybeThrottle(request, createResponse)  
  }  
}

def partitionFor(group: String): Int = groupManager.partitionFor(group)
// groupId 的 hash 值 % __cousumer_offsets 的分区数(50)
def partitionFor(groupId: String): Int = Utils.abs(groupId.hashCode) % groupMetadataTopicPartitionCount
```
### 3.3.4 Consumer 回调
```java
// lookupCoordinator() 最后调用 compose()
// 传入 FindCoordinatorResponseHandler
public <S> RequestFuture<S> compose(final RequestFutureAdapter<T, S> adapter) {  
    final RequestFuture<S> adapted = new RequestFuture<>();  
    addListener(new RequestFutureListener<T>() {  
        @Override  
        public void onSuccess(T value) {  
            adapter.onSuccess(value, adapted);  
        }  
    });  
    return adapted;  
}

public void onSuccess(ClientResponse resp, RequestFuture<Void> future) {  
    log.debug("Received FindCoordinator response {}", resp);  
    
    if (error == Errors.NONE) {  
        synchronized (AbstractCoordinator.this) {  
            AbstractCoordinator.this.coordinator = new Node(  
                    coordinatorConnectionId,  
                    coordinatorData.host(),  
                    coordinatorData.port());  
            // 建立连接
            client.tryConnect(coordinator);  
            heartbeat.resetSessionTimeout();  
        }  
        future.complete(null);  
    }   
}
```
### 3.3.4 Consumer 发起 JoinGroupRequest
```java
// ensureActiveGroup() 返回 joinGroupIfNeeded()
boolean joinGroupIfNeeded(final Timer timer) {  
    while (rejoinNeededOrPending()) {  
	    // 确保已经找到 GroupCoordinator
        if (!ensureCoordinatorReady(timer)) {  
            return false;  
        }  
		// Rebalance正在执行的过程中，如果再次执行（元数据刷新导致重平衡）则不需要准备工作
        if (needsJoinPrepare) {            
            needsJoinPrepare = false;  
            if (!onJoinPrepare(timer, generation.generationId, generation.memberId)) {  
                needsJoinPrepare = true;  
                return false;  
            }  
        }  
	    // 初始化 JoinGroup，发送请求
        final RequestFuture<ByteBuffer> future = initiateJoinGroup();  
        client.poll(future, timer);  
    }  
    return true;  
}

private synchronized RequestFuture<ByteBuffer> initiateJoinGroup() {  
    if (joinFuture == null) {  
	    // 开始准备重平衡
        state = MemberState.PREPARING_REBALANCE;   
        // 发送请求
        joinFuture = sendJoinGroupRequest();  
    }  
    return joinFuture;  
}

RequestFuture<ByteBuffer> sendJoinGroupRequest() {  
  
    JoinGroupRequest.Builder requestBuilder = new JoinGroupRequest.Builder(  
            new JoinGroupRequestData()  
	        .setGroupId(rebalanceConfig.groupId)  
	        // 客户端与 Broker 最大会话有效期，默认 10s
		    .setSessionTimeoutMs(this.rebalanceConfig.sessionTimeoutMs)  
            .setMemberId(this.generation.memberId)  
	.setGroupInstanceId(this.rebalanceConfig.groupInstanceId.orElse(null))  
            .setProtocolType(protocolType())  
            .setProtocols(metadata())  
	.setRebalanceTimeoutMs(this.rebalanceConfig.rebalanceTimeoutMs)                .setReason(JoinGroupRequest.maybeTruncateReason(this.rejoinReason))  
    );  

	// joinGroup 的请求超时时间
	// 取 request.timeout.ms 默认 30s 
	// 和 JOIN_GROUP_TIMEOUT_LAPSE = 5s + request.timeout.ms 最大值
    int joinGroupTimeoutMs = Math.max(  
        client.defaultRequestTimeoutMs(),  
        Math.max(  
            rebalanceConfig.rebalanceTimeoutMs + JOIN_GROUP_TIMEOUT_LAPSE,  
            rebalanceConfig.rebalanceTimeoutMs)
        );  
    // 发请求
    return client.send(coordinator, requestBuilder, joinGroupTimeoutMs)  
            .compose(new JoinGroupResponseHandler(generation));  
}
```
### 3.3.5 GroupCoordinator 处理 JoinGroupRequest 请求
```java
override def handle(request: RequestChannel.Request): Unit = {  
  try {
		request.header.apiKey match {
			case ApiKeys.JOIN_GROUP => handleJoinGroupRequest(request)
			}
	}
}

def handleJoinGroupRequest(request: RequestChannel.Request): Unit = {  
    groupCoordinator.handleJoinGroup(...)  
  }  
}

def handleJoinGroup(groupId: String,  
					// 新加入成员该字段为""
                    memberId: String,  
                    groupInstanceId: Option[String],  
                    requireKnownMemberId: Boolean,  
					// Coordinator 使用它来生成 memberId
                    clientId: String,  
                    clientHost: String,  
                    rebalanceTimeoutMs: Int,  
                    sessionTimeoutMs: Int,  
                    protocolType: String,  
                    protocols: List[(String, Array[Byte])],  
                    responseCallback: JoinCallback): Unit = {  
	// 验证消费者组状态的合法性
	// sessionTimeoutMs参数校验
	
    // 获取组信息
    groupManager.getOrMaybeCreateGroup(groupId, isUnknownMember) match {  
      case Some(group) =>  
        group.inLock {  
          // 判断是否接收该成员
          if (!acceptJoiningMember(group, memberId)) {  
            group.remove(memberId)  
            group.removeStaticMember(groupInstanceId)  
          } else if (isUnknownMember) {  
	        // 新的 member 执行 doUnknownJoinGroup
            doUnknownJoinGroup(group, groupInstanceId, requireKnownMemberId, clientId, clientHost, rebalanceTimeoutMs, sessionTimeoutMs, protocolType, protocols, responseCallback)  
          } else { 
	        // 已存在的 member 请求加入 
            doJoinGroup(group, memberId, groupInstanceId, clientId, clientHost, rebalanceTimeoutMs, sessionTimeoutMs, protocolType, protocols, responseCallback)  
          }  
        }  
    }  
  }  
}

private def acceptJoiningMember(group: GroupMetadata, member: String): Boolean = {  
  group.currentState match {  
    // empty 或 dead 可以加入成员  
    case Empty | Dead =>  
      true  
	// PreparingRebalance 状态需要：之前已有的正在等待加入组的成员
	//      当前等待加入组的成员数小于 Broker 端参数 group.max.size 值
    case PreparingRebalance =>  
      (group.has(member) && group.get(member).isAwaitingJoin) ||  
        group.numAwaiting < groupConfig.groupMaxSize  
  
    // 其他状态：需要是已有成员，或当前组总成员数小于 Broker 端参数 group.max.size 值
    case CompletingRebalance | Stable =>  
      group.has(member) || group.size < groupConfig.groupMaxSize  
  }  
}
```
### 3.3.6 新成员入组
```java
private def doUnknownJoinGroup(group: GroupMetadata,  
                               groupInstanceId: Option[String],  
                               requireKnownMemberId: Boolean,  
                               clientId: String,  
                               clientHost: String,  
                               rebalanceTimeoutMs: Int,  
                               sessionTimeoutMs: Int,  
                               protocolType: String,  
                               protocols: List[(String, Array[Byte])],  
                               responseCallback: JoinCallback): Unit = {  
  group.inLock {  
    if (group.is(Dead)) {  
	  // 封装异常调用回调返回  
      responseCallback(JoinGroupResult(JoinGroupRequest.UNKNOWN_MEMBER_ID, Errors.COORDINATOR_NOT_AVAILABLE))  
    } else if (!group.supportsProtocols(protocolType, MemberMetadata.plainProtocolSet(protocols))) { 
      // 协议不匹配 
      responseCallback(JoinGroupResult(JoinGroupRequest.UNKNOWN_MEMBER_ID, Errors.INCONSISTENT_GROUP_PROTOCOL))  
    } else {  
	  // 创建 memberId
	  // 动态Member ：cliend.id-UUID.toString
      // 静态Member： group.instance.id-UUID.toString
      val newMemberId = group.generateMemberId(clientId, groupInstanceId)  
      if (group.hasStaticMember(groupInstanceId)) {
      } else if (requireKnownMemberId) {
      } else {  
	    // 重平衡
        addMemberAndRebalance(rebalanceTimeoutMs, sessionTimeoutMs, newMemberId, groupInstanceId, clientId, clientHost, protocolType, protocols, group, responseCallback)  
      }  
    }  
  }  
}
```
### 3.3.7 旧成员入组
```java
private def doJoinGroup(group: GroupMetadata,  
                        memberId: String,  
                        groupInstanceId: Option[String],  
                        clientId: String,  
                        clientHost: String,  
                        rebalanceTimeoutMs: Int,  
                        sessionTimeoutMs: Int,  
                        protocolType: String,  
                        protocols: List[(String, Array[Byte])],  
                        responseCallback: JoinCallback): Unit = {  
  group.inLock {  
    if (group.is(Dead)) {   
      responseCallback(JoinGroupResult(memberId, Errors.COORDINATOR_NOT_AVAILABLE))  
    } else if (!group.supportsProtocols(protocolType, MemberMetadata.plainProtocolSet(protocols))) {  
      responseCallback(JoinGroupResult(memberId, Errors.INCONSISTENT_GROUP_PROTOCOL))  
    } else if (group.isPendingMember(memberId)) {  
      if (groupInstanceId.isDefined) {...} 
      else {  
	    // 如果是待决成员，由于这次分配了成员ID，故允许加入组
        addMemberAndRebalance(rebalanceTimeoutMs, sessionTimeoutMs, memberId, groupInstanceId,  
          clientId, clientHost, protocolType, protocols, group, responseCallback)  
      }  
    } else {  
	  // 处理非 pending 状态成员的入组请求
      val groupInstanceIdNotFound = groupInstanceId.isDefined && !group.hasStaticMember(groupInstanceId)  
        // 获取元数据信息
        val member = group.get(memberId)  
  
        group.currentState match {  
          // 将要开启重平衡时直接加入
          case PreparingRebalance =>  
            updateMemberAndRebalance(group, member, protocols, s"Member ${member.memberId} joining group during ${group.currentState}", responseCallback)  
		  // 判断 member 的分区消费分配策略与订阅分区列表是否和记录中一致
          case CompletingRebalance =>  
            // 相同，说明已经发起过加入组的操作，没有收到返回信息
            // 则创建一个 JoinGroupResult 对象返回给 member
            if (member.matches(protocols)) {  
              responseCallback(JoinGroupResult(  
                members = if (group.isLeader(memberId)) {  
                  group.currentMemberMetadata  
                } else {  
                  List.empty  
                },  
                memberId = memberId,  
                generationId = group.generationId,  
                protocolType = group.protocolType,  
                protocolName = group.protocolName,  
                leaderId = group.leaderOrNull,  
                error = Errors.NONE))  
            } else {  
              // 成员变更了订阅信息或分配策略，需要强制重平衡
              // member has changed metadata, so force a rebalance  
              updateMemberAndRebalance(group, member, protocols, s"Updating metadata for member ${member.memberId} during ${group.currentState}", responseCallback)  
            }  
  
          case Stable =>  
            val member = group.get(memberId)  
            if (group.isLeader(memberId)) {  
		      // 如果是 leader 
              updateMemberAndRebalance(group, member, protocols, s"leader ${member.memberId} re-joining group during ${group.currentState}", responseCallback)  
              // 或成员变更了分区分配策略，则开启一轮 rebalance
            } else if (!member.matches(protocols)) {  
              updateMemberAndRebalance(group, member, protocols, s"Updating metadata for member ${member.memberId} during ${group.currentState}", responseCallback)  
            } else {  
              // 否则返回当前组信息给该成员即可
              responseCallback(JoinGroupResult(...))  
            }  
          case Empty | Dead =>  
            // 封装异常
            responseCallback(JoinGroupResult(memberId, Errors.UNKNOWN_MEMBER_ID))  
        }  
      }  
    }  
  }  
}
```
### 3.3.8 添加成员重平衡
```java
private def addMemberAndRebalance(rebalanceTimeoutMs: Int,  
                                  sessionTimeoutMs: Int,  
                                  memberId: String,  
                                  groupInstanceId: Option[String],  
                                  clientId: String,  
                                  clientHost: String,  
                                  protocolType: String,  
                                  protocols: List[(String, Array[Byte])],  
                                  group: GroupMetadata,  
                                  callback: JoinCallback): Unit = {  
  // 创建 member 对象
  val member = new MemberMetadata(memberId, group.groupId, groupInstanceId,  
    clientId, clientHost, rebalanceTimeoutMs,  
    sessionTimeoutMs, protocolType, protocols)  
  // 标记为新成员
  member.isNew = true  
  
  // 如果消费者组准备开启首次Rebalance，设置newMemberAdded为True
  if (group.is(PreparingRebalance) && group.generationId == 0)  
    group.newMemberAdded = true  
  // 入组
  group.add(member, callback)  
  
 // 设置下次心跳过期时间(member 的 session 过期时间)
  completeAndScheduleNextExpiration(group, member, NewMemberJoinTimeoutMs)  
  // 准备开启 rebalance
  maybePrepareRebalance(group, s"Adding new member $memberId with group instance id $groupInstanceId")  
}
```
#### 3.3.8.1 添加成员
```java
def add(member: MemberMetadata, callback: JoinCallback = null): Unit = {
  // 如果是第一个添加的消费者组成员 
  if (members.isEmpty)  
    // 将该成员的 protocolType 设置为组的 protocolType，以及分区分配策略与组相同
    this.protocolType = Some(member.protocolType)  
  // 确保成员的组ID，protocolType
  assert(groupId == member.groupId)  
  assert(this.protocolType.orNull == member.protocolType)  
  assert(supportsProtocols(member.protocolType, MemberMetadata.plainProtocolSet(member.supportedProtocols)))  

  // 判断有没有 leader
  if (leaderId.isEmpty)  
    // 把该成员设定为Leader成员
    leaderId = Some(member.memberId) 
  // 否则添加进成员 
  members.put(member.memberId, member)  
  member.supportedProtocols.foreach{ case (protocol, _) => supportedProtocols(protocol) += 1 }  
  member.awaitingJoinCallback = callback  
// 更新已加入组的成员数
  if (member.isAwaitingJoin)  
    numMembersAwaitingJoin += 1  
}
```
#### 3.3.8.2 重平衡
- 每个消费者向 `GroupCoordinator` 发送入组请求，包含了各自提案的分配策略和订阅信息
- 入组响应中包含了投票选举出的分配策略的信息，并且只有 leader 消费者的回执中包含各个消费者的订阅信息
```java
private def maybePrepareRebalance(group: GroupMetadata, reason: String): Unit = {  
  group.inLock 
  { 
	// 判断前置状态是否合法 
    if (group.canRebalance)  
      prepareRebalance(group, reason)  
  }  
}

private[group] def prepareRebalance(group: GroupMetadata, reason: String): Unit = {  
  if (group.is(CompletingRebalance))  
    resetAndPropagateAssignmentError(group, Errors.REBALANCE_IN_PROGRESS)  
  // 为空说明 GroupCoordinator 第一次接收客户端入组请求
  val delayedRebalance = if (group.is(Empty))  
    new InitialDelayedJoin(this,  
      joinPurgatory,  
      group,  
      groupConfig.groupInitialRebalanceDelayMs,  
      groupConfig.groupInitialRebalanceDelayMs,  
      max(group.rebalanceTimeoutMs - groupConfig.groupInitialRebalanceDelayMs, 0))  
  else  
    // 客户端 poll 间隔
    new DelayedJoin(this, group, group.rebalanceTimeoutMs)  
  // 尝试将group状态流转为  PreparingRebalance，并且尝试完成Join
  group.transitionTo(PreparingRebalance)  
  
  val groupKey = GroupKey(group.groupId)  
  // 延迟操作执行delayedRebalance。如果是DelayedJoin类型，会执行对应的onComplete方法
  joinPurgatory.tryCompleteElseWatch(delayedRebalance, Seq(groupKey))  
}

def onCompleteJoin(group: GroupMetadata): Unit = {  
  group.inLock {  
    if (group.is(Dead)) {  
      info(s"Group ${group.groupId} is dead, skipping rebalance stage")  
    } else if (!group.maybeElectNewJoinedLeader() && group.allMembers.nonEmpty) {  
      // 如果所有成员都没有重新加入，我们将推迟重新平衡准备阶段的完成，并发出另一个延迟操作，直到会话超时删除所有未响应的成员
      joinPurgatory.tryCompleteElseWatch(  
        new DelayedJoin(this, group, group.rebalanceTimeoutMs),  
        Seq(GroupKey(group.groupId)))  
    } else {  
      // 初始化下个 generation，generationId + 1，组不为空时进入 CompletingRebalance 状态
      // 选择组的分配协议，所有 member 支持的协议中票数最多的
      group.initNextGeneration()  
      if (group.is(Empty)) {  
		// Group是空的话，我们则在 __consumer_offset 里面写入一条Group的元数据,状态是Empty
        groupManager.storeGroup(group, Map.empty, error => {...})  
      } else {  
      
        // trigger the awaiting join group response callback for all the members after rebalancing  
        // rebalance 后触发所有等待Join group的members的的回调接口
        for (member <- group.allMemberMetadata) {  
          val joinResult = JoinGroupResult(  
            // 只有 leader 会返回 member 元数据信息
            members = if (group.isLeader(member.memberId)) {  
              group.currentMemberMetadata  
            } else {  
              List.empty  
            },  
            memberId = member.memberId,  
            generationId = group.generationId,  
            protocolType = group.protocolType,  
            protocolName = group.protocolName,  
            leaderId = group.leaderOrNull,  
            error = Errors.NONE)  
		  // member 的回调函数
          group.maybeInvokeJoinCallback(member, joinResult)  
          completeAndScheduleNextHeartbeatExpiration(group, member)  
          member.isNew = false  
        }  
      }  
    }  
  }  
}

// GroupMetadata.scala
def initNextGeneration(): Unit = {  
  if (members.nonEmpty) {  
    generationId += 1  
    protocolName = Some(selectProtocol)  
    subscribedTopics = computeSubscribedTopics()  
    transitionTo(CompletingRebalance)  
  } else {  
    generationId += 1  
    protocolName = None  
    subscribedTopics = computeSubscribedTopics()  
    transitionTo(Empty)  
  }  
  receivedConsumerOffsetCommits = false  
  receivedTransactionalOffsetCommits = false  
}
```
### 3.3.9 处理 JoinGroupResponse
```java
// CoordinatorResponseHandler 处理响应
public void onSuccess(ClientResponse clientResponse, RequestFuture<T> future) {  
    try {  
        this.response = clientResponse;  
        R responseObj = (R) clientResponse.responseBody();  
        handle(responseObj, future);  
    }
}

// 调用 JoinGroupResponseHandler 的 handle 方法
public void handle(JoinGroupResponse joinResponse, RequestFuture<ByteBuffer> future) {
	
	synchronized (AbstractCoordinator.this) {  
			if (heartbeatThread != null) 
			    // 开启心跳 
			    heartbeatThread.enable();
	        if (joinResponse.isLeader()) {  
	            onLeaderElected(joinResponse).chain(future);  
	        } else {  
	            onJoinFollower().chain(future);  
	        }  
	    }  
	}
}
```
### 3.3.10 Leader 制定消费计划
```java
// AbstractCoordinator.java
private RequestFuture<ByteBuffer> onLeaderElected(JoinGroupResponse joinResponse) {  
    try {  
        // 调用 ConsumerCoordinator的方法
        Map<String, ByteBuffer> groupAssignment = onLeaderElected(  
            joinResponse.data().leader(),  
            joinResponse.data().protocolName(),  
            joinResponse.data().members(),  
            joinResponse.data().skipAssignment()  
        );  
  
        List<SyncGroupRequestData.SyncGroupRequestAssignment> groupAssignmentList = new ArrayList<>();  
	  // 循环向集合中添加分区策略
  
        SyncGroupRequest.Builder requestBuilder =  
                new SyncGroupRequest.Builder(...);  
		// 创建并发送 SyncGroupRequest
        return sendSyncGroupRequest(requestBuilder);  
    }
}

protected Map<String, ByteBuffer> onLeaderElected(String leaderId,  
   String assignmentStrategy,  
  List<JoinGroupResponseData.JoinGroupResponseMember> allSubscriptions,  
  boolean skipAssignment) {  

    // 获取对应的分区的分配策略
    // this.assigner
    ConsumerPartitionAssignor assignor = lookupAssignor(assignmentStrategy);  
    String assignorName = assignor.name();  
  
    Set<String> allSubscribedTopics = new HashSet<>();  
    Map<String, Subscription> subscriptions = new HashMap<>();  
  
    // collect all the owned partitions 

    // 分配策略
    Map<String, Assignment> assignments = assignor.assign(metadata.fetch(), new GroupSubscription(subscriptions)).groupAssignment();  
    return groupAssignment;  
}

// AbstractPartitionAssigner.java
public GroupAssignment assign(Cluster metadata, GroupSubscription groupSubscription) {  
    // 获取消费者组的订阅关系
    Map<String, Subscription> subscriptions = groupSubscription.groupSubscription();  
    Set<String> allSubscribedTopics = new HashSet<>();  
    for (Map.Entry<String, Subscription> subscriptionEntry : subscriptions.entrySet())  
        allSubscribedTopics.addAll(subscriptionEntry.getValue().topics());  

	// 遍历订阅的Topic，根据Topic从元数据中获取对应的partation信息
    Map<String, Integer> partitionsPerTopic = new HashMap<>();  
    for (String topic : allSubscribedTopics) {  
        Integer numPartitions = metadata.partitionCountForTopic(topic);  
        if (numPartitions != null && numPartitions > 0)  
            partitionsPerTopic.put(topic, numPartitions);  
    }  
    // 分配策略
    Map<String, List<TopicPartition>> rawAssignments = assign(partitionsPerTopic, subscriptions);  
  
    return new GroupAssignment(assignments);  
}

// leader Consumer 发送 SYNC_GROUP 请求
private RequestFuture<ByteBuffer> sendSyncGroupRequest(SyncGroupRequest.Builder requestBuilder) {  
    if (coordinatorUnknown())  
        return RequestFuture.coordinatorNotAvailable();  
    return client.send(coordinator, requestBuilder)  
            .compose(new SyncGroupResponseHandler(generation));  
}
```
### 3.3.11 Server 处理同步组请求，发送消费方案
```java
case ApiKeys.SYNC_GROUP => handleSyncGroupRequest(request)

def handleSyncGroupRequest(request: RequestChannel.Request): Unit = {  
  
  def sendResponseCallback(syncGroupResult: SyncGroupResult): Unit = {  
    sendResponseMaybeThrottle(request, requestThrottleMs =>  
      new SyncGroupResponse(...)  
  }  
  
  groupCoordinator.handleSyncGroup(  
      syncGroupRequest.data.groupId,  
      syncGroupRequest.data.generationId,  
      syncGroupRequest.data.memberId,  
      Option(syncGroupRequest.data.protocolType),  
      Option(syncGroupRequest.data.protocolName),  
      Option(syncGroupRequest.data.groupInstanceId),  
      assignmentMap.result(),  
      sendResponseCallback  
    )  
  }  
}

def handleSyncGroup(groupId: String,  
                    generation: Int,  
                    memberId: String,  
                    protocolType: Option[String],  
                    protocolName: Option[String],  
                    groupInstanceId: Option[String],  
                    groupAssignment: Map[String, Array[Byte]],  
                    responseCallback: SyncCallback): Unit = {  
  validateGroupStatus(groupId, ApiKeys.SYNC_GROUP) match {  
    case None =>  
      groupManager.getGroup(groupId) match {  
        // 同步
        case Some(group) => doSyncGroup(group, generation, memberId, protocolType, protocolName,  
          groupInstanceId, groupAssignment, responseCallback)  
      }  
  }  
}		

private def doSyncGroup(group: GroupMetadata,  
                        generationId: Int,  
                        memberId: String,  
                        protocolType: Option[String],  
                        protocolName: Option[String],  
                        groupInstanceId: Option[String],  
                        groupAssignment: Map[String, Array[Byte]],  
                        responseCallback: SyncCallback): Unit = {  
  group.inLock {  
    // 判断消费者组状态是否为 dead
    if (group.is(Dead)) {  
      responseCallback(SyncGroupResult(Errors.COORDINATOR_NOT_AVAILABLE))  
      // 静态成员相关
    } else if (group.isStaticMemberFenced(memberId, groupInstanceId, "sync-group")) {  
      responseCallback(SyncGroupResult(Errors.FENCED_INSTANCE_ID))  
      // 通过 memberId 判断是否该组成员
    } else if (!group.has(memberId)) {  
      responseCallback(SyncGroupResult(Errors.UNKNOWN_MEMBER_ID)) 
      // 判断成员的 generationId 是否与组相同
    } else if (generationId != group.generationId) {  
      responseCallback(SyncGroupResult(Errors.ILLEGAL_GENERATION))  
      // 判断协议类型
    } else if (protocolType.isDefined && !group.protocolType.contains(protocolType.get)) {  
      responseCallback(SyncGroupResult(Errors.INCONSISTENT_GROUP_PROTOCOL))
      // 判断分区分配策略  
    } else if (protocolName.isDefined && !group.protocolName.contains(protocolName.get)) {  
      responseCallback(SyncGroupResult(Errors.INCONSISTENT_GROUP_PROTOCOL))  
    } else {  
      group.currentState match { 
        // 封装异常，回调 
        case Empty =>  
          responseCallback(SyncGroupResult(Errors.UNKNOWN_MEMBER_ID))  
		// 封装异常，回调 
        case PreparingRebalance =>  
          responseCallback(SyncGroupResult(Errors.REBALANCE_IN_PROGRESS))  
	    // 设置回调
        case CompletingRebalance =>  
          group.get(memberId).awaitingSyncCallback = responseCallback  
  
          // leader 需要处理
          if (group.isLeader(memberId)) {  
            val missing = group.allMembers.diff(groupAssignment.keySet) 
            // 如果没有消费方案，创建一个空方案 
            val assignment = groupAssignment ++ missing.map(_ -> Array.empty[Byte]).toMap  
			// 保存消费者组信息至元数据，写入内部 offset 中
            groupManager.storeGroup(group, assignment, (error: Errors) => {
	            group.inLock {  
				  if (group.is(CompletingRebalance) && generationId == group.generationId) {  
				    if (error != Errors.NONE) {...}
				    else {  
				      // 在消费者组元数据中保存分配方案并发送给所有成员
				      setAndPropagateAssignment(group, assignment)  
				      group.transitionTo(Stable)  
				    }  
				  }  
				}
            }
  
        case Stable =>  
          val memberMetadata = group.get(memberId)  
          responseCallback(SyncGroupResult(group.protocolType, group.protocolName, memberMetadata.assignment, Errors.NONE))  
          completeAndScheduleNextHeartbeatExpiration(group, group.get(memberId))  
  
        case Dead =>  
          throw new IllegalStateException(s"Reached unexpected condition for Dead group ${group.groupId}")  
      }  
    }  
  }  
}

private def setAndPropagateAssignment(group: GroupMetadata, assignment: Map[String, Array[Byte]]): Unit = {  
  assert(group.is(CompletingRebalance))  
  // 遍历 member 发送分区分配方案
  group.allMemberMetadata.foreach(member => member.assignment = assignment(member.memberId))  
  propagateAssignment(group, Errors.NONE)  
}
```
### 3.3.15 重平衡成功后消费者更新自己的分区
```java
// 3.3.4 joinGroupIfNeeded()
boolean joinGroupIfNeeded(final Timer timer) {
	if (future.succeeded()) {
		if (!hasGenerationReset(generationSnapshot) && stateSnapshot == MemberState.STABLE) {  
    ByteBuffer memberAssignment = future.value().duplicate();  
    onJoinComplete(...);
	    }
	}
}

protected void onJoinComplete(int generation,  
                              String memberId,  
                              String assignmentStrategy,  
                              ByteBuffer assignmentBuffer) {  

  
    ConsumerPartitionAssignor assignor = lookupAssignor(assignmentStrategy);  
    // 已经订阅的 topic 及分区信息
    SortedSet<TopicPartition> ownedPartitions = new TreeSet<>(COMPARATOR);  
    ownedPartitions.addAll(subscriptions.assignedPartitions());  
    // 将 GroupCoordinator 响应数据反序列化为 Assignment
    Assignment assignment = ConsumerProtocol.deserializeAssignment(assignmentBuffer);  
    // 获取GroupCoordinator发给消费者对应消费的TopicPartition信息
    SortedSet<TopicPartition> assignedPartitions = new TreeSet<>(COMPARATOR); 
    // 消费者更新自己的分区
    subscriptions.assignFromSubscribed(assignedPartitions);  
}
```
## 3.4 消费者分区策略
### 3.4.1 RangeAssigner
```java
public Map<String, List<TopicPartition>> assign(
	Map<String, Integer> partitionsPerTopic,  
    Map<String, Subscription> subscriptions) {  

	// topic 和订阅的消费者集合信息，例如{t0:[c0, c1], t1:[C0, C1]}
    Map<String, List<MemberInfo>> consumersPerTopic = consumersPerTopic(subscriptions); 
	// 保存 topic 分区和订阅该 topic 的消费者关系结果
    Map<String, List<TopicPartition>> assignment = new HashMap<>();  

	// range策略分配结果在各个topic之间互不影响
    for (Map.Entry<String, List<MemberInfo>> topicEntry : consumersPerTopic.entrySet()) {  
        String topic = topicEntry.getKey();  
        List<MemberInfo> consumersForTopic = topicEntry.getValue();  
        Integer numPartitionsForTopic = partitionsPerTopic.get(topic);  
		// 消费者集合根据字典排序
        Collections.sort(consumersForTopic);  
		// 每个topic分区数量除以消费者数量，得出每个消费者分配到的分区数量
        int numPartitionsPerConsumer = numPartitionsForTopic / consumersForTopic.size();  
        // 不能整除的数量
        int consumersWithExtraPartition = numPartitionsForTopic % consumersForTopic.size();  
		
        List<TopicPartition> partitions = AbstractPartitionAssignor.partitions(topic, numPartitionsForTopic);  
        // 整除分配到的分区数量，加上1个无法整除分配到的分区
        for (int i = 0, n = consumersForTopic.size(); i < n; i++) {  
            int start = numPartitionsPerConsumer * i + Math.min(i, consumersWithExtraPartition);  
            // 整除后余数为 m，排序后的消费者集合中前 m 个都能分配到一个额外的分区
            int length = numPartitionsPerConsumer + (i + 1 > consumersWithExtraPartition ? 0 : 1);  assignment.get(consumersForTopic.get(i).memberId).addAll(partitions.subList(start, start + length));  
        }  
    }  
    return assignment;  
}
```
### 3.4.2 RoundRobinAssignor
```java
public Map<String, List<TopicPartition>> assign(
	Map<String, Integer> partitionsPerTopic,  
    Map<String, Subscription> subscriptions) {  
    Map<String, List<TopicPartition>> assignment = new HashMap<>();  
    List<MemberInfo> memberInfoList = new ArrayList<>();  
    for (Map.Entry<String, Subscription> memberSubscription : subscriptions.entrySet()) {  
        assignment.put(memberSubscription.getKey(), new ArrayList<>());  
        memberInfoList.add(new MemberInfo(memberSubscription.getKey(),                   memberSubscription.getValue().groupInstanceId()));  
    }  
	// 环形迭代器：迭代器中没有元素时返回第一个元素继续迭代
	// 对消费者进行排序
    CircularIterator<MemberInfo> assigner = new CircularIterator<>(Utils.sorted(memberInfoList));  
  
    for (TopicPartition partition : allPartitionsSorted(partitionsPerTopic, subscriptions)) {  
        final String topic = partition.topic();  
        // 如果遍历得到的消费者没有订阅当前 topic，则继续循环
        while (!subscriptions.get(assigner.peek().memberId).topics().contains(topic))  
            assigner.next();  
        assignment.get(assigner.next().memberId).add(partition);  
    }  
    return assignment;  
}
```
### 3.4.3 AbstractStickyAssignor
- AbstractStickyAssignor 有两个实现类
	- `CooperativeStickyAssignor`
	- `StickyAssignor`：分区的分配尽量均匀，尽可能与上次保持相同
```java
// todo
```
## 3.5 消费流程
### 3.5.1 poll()
```java
private ConsumerRecords<K, V> poll(final Timer timer, final boolean includeMetadataInTimeout) {  
    try {   
	    // 未过期
        do {  
			// 拉取 Fetch 
            final Fetch<K, V> fetch = pollForFetches(timer);  
            if (!fetch.isEmpty()) {               
                if (fetcher.sendFetches() > 0 || client.hasPendingRequests()) {  
                    // 向 Server 发送请求
                    client.transmitSends();  
                }  
                return this.interceptors.onConsume(new ConsumerRecords<>(fetch.records()));  
            }  
        } while (timer.notExpired());  
        return ConsumerRecords.empty();  
    } 
}
```
### 3.5.2 sendFetches()
```java
// Fetcher.java
public synchronized int sendFetches() {  
    Map<Node, FetchSessionHandler.FetchRequestData> fetchRequestMap = prepareFetchRequests();  
    for (Map.Entry<Node, FetchSessionHandler.FetchRequestData> entry : fetchRequestMap.entrySet()) {  
		// 创建请求
		// maxWaitMs 最大等待时间，默认500ms 
		// minBytes 最少抓取一个字节 
		// maxBytes 最大抓取多少数据 默认50m
        final FetchRequest.Builder request = FetchRequest.Builder  
                .forConsumer(maxVersion, this.maxWaitMs, this.minBytes, data.toSend())  
                .isolationLevel(isolationLevel)  
                .setMaxBytes(this.maxBytes)  
                .metadata(data.metadata())  
                .removed(data.toForget())  
                .replaced(data.toReplace())  
                .rackId(clientRackId);  
  
        RequestFuture<ClientResponse> future = client.send(fetchTarget, request);    
        future.addListener(new RequestFutureListener<ClientResponse>() {  
            @Override  
            public void onSuccess(ClientResponse resp) {  
				completedFetches.add(new CompletedFetch(partition, partitionData,metricAggregator, batches, fetchOffset, responseVersion));  
	               }  
              }  
          } 
    return fetchRequestMap.size();  
}
```
### 3.5.3 Server 处理 Fetch 请求
```java
// KafkaApis
case ApiKeys.FETCH => handleFetchRequest(request)

def handleFetchRequest(request: RequestChannel.Request): Unit = {
	val interesting = mutable.ArrayBuffer[(TopicPartition, FetchRequest.PartitionData)]() 
	// interesting += (topicPartition -> data)
	if (interesting.isEmpty)  
  processResponseCallback(Seq.empty)  
else {  
  // call the replica manager to fetch messages from the local replica  
  replicaManager.fetchMessages(  
    fetchRequest.maxWait.toLong,  
    fetchRequest.replicaId,  
    fetchMinBytes,  
    fetchMaxBytes,  
    versionId <= 2,  
    interesting,  
    replicationQuota(fetchRequest),  
    // 创建响应
    processResponseCallback,  
    fetchRequest.isolationLevel,  
    clientMetadata)  
	}
}

// ReplicaManager 拉取数据
def fetchMessages(...): Unit = {
	val logReadResults = readFromLog()
	def readFromLog(): Seq[(TopicPartition, LogReadResult)] = {  
  val result = readFromLocalLog(  
    replicaId = replicaId,  
    fetchOnlyFromLeader = fetchOnlyFromLeader,  
    fetchIsolation = fetchIsolation,  
    fetchMaxBytes = fetchMaxBytes,  
    hardMaxBytesLimit = hardMaxBytesLimit,  
    readPartitionInfo = fetchInfos,  
    quota = quota,  
    clientMetadata = clientMetadata)  
  if (isFromFollower) updateFollowerFetchState(replicaId, result)  
  else result  
	}
}

def readFromLocalLog(...): Seq[(TopicPartition, LogReadResult)] = {
	var limitBytes = fetchMaxBytes  
	val result = new mutable.ArrayBuffer[(TopicPartition, LogReadResult)]  
	readPartitionInfo.foreach { case (tp, fetchInfo) =>  
	  val readResult = read(tp, fetchInfo, limitBytes, minOneMessage)  
	  val recordBatchSize = readResult.info.records.sizeInBytes  
  if (recordBatchSize > 0)  
	  limitBytes = math.max(0, limitBytes - recordBatchSize)  
	  result += (tp -> readResult)  
}

def read(tp: TopicPartition, fetchInfo: PartitionData, limitBytes: Int, minOneMessage: Boolean): LogReadResult = {
	// 调用 Partition.scala 的 readRecords()
	val readInfo: LogReadInfo = partition.readRecords(...)
	// 返回 LogReadResult
	LogReadResult(...)
}

// Server 创建响应
def processResponseCallback(responsePartitionData: Seq[(TopicPartition, FetchPartitionData)]): Unit = {
	if (fetchRequest.isFromFollower) {...}
	else {
	    // 都会调用 sendResponse()
		sendResponse(request, Some(response), onComplete)	
	}
}
```
### 3.5.4 Consumer 处理 Fetch 响应
```java
// ConcurrentLinkedQueue<CompletedFetch> completedFetches
public void onSuccess(ClientResponse resp) {  
	// 创建 CompletedFetch 加入到队列中
	completedFetches.add(new CompletedFetch(partition, partitionData,metricAggregator, batches, fetchOffset, responseVersion));  
}
// poll()
// 返回拦截器处理后的 ProducerRecord
return this.interceptors.onConsume(new ConsumerRecords<>(fetch.records()));
```
# 4 时间轮
# 5 Shell 操作
```shell
kafka-topics.sh --bootstrap-server hadoop102:9092
# 查看所有主题
--list 
# 创建主题
--create --topic [topic_name] --partitions 3 --replication-factor 3 
# 修改分区数
--alter --topic [topic_name] --partitions # 分区数只能增加
# 删除 topic
--delete --topic [topic_name]
```
# 6 复习
## 6.1 基本信息
### 6.1.1 生产流程
- 两个线程：`main`， `sender`
- 拦截器
- 序列化器
- 分区器
	- 默认 `DefaultPartitioner`
		- `ProducerRecord` 中指定分区直接发往指定分区
		- 未指定分区，但是有 Key，通过 hash 值
	- 粘性
	- 轮询：每次加 1 后取余，避免数据每次第一次发送给同一个分区
- ack
	- -1(all): `Leader` 和 `ISR` 队列中全部的副本收到数据，进行响应
	- 0：生产者发送过来的数据不需要等待数据落盘应答
	- 1：生产者发送过来的数据，`Leader`收到数据后应答
### 6.1.2 Broker 相关
- 主题：逻辑上的分类
- 主题名-分区号作为目录名进行物理存储
- 分区：提高吞吐量，增加了负载均衡的能力
- 副本：提高可靠性
	- `ISR + OSR = AR`
	- `LEO`：每个副本最大的 `offset + 1`
	- `HW`: 同一个分区中，多个副本最小的 `LEO`
		- 消费者可见的最大 `offset`
		- 当需要消费的 `offset` 超过记录的 `offset` 时，会触发  `auto.offset.reset`
		- 第一次启动时(新的消费者组), 没有初始的 `offset` 时也会触发
		- 保证了存储数据的一致性

### 6.1.3 消费者
- 分区分配策略
	- `RangeAssignor`：按范围分
	- `RoundRobinAssignor`：对消费者进行排序后，轮询消费
	- `StickyAssignor`：在重新分区时，尽量保证原有分区与消费者之间的绑定关系不变
- Offset 存储
	- 老版本：`zk`
	- 新版本：`__consumer_offsets` 50 个分区
	- 手动维护：`Flink：Checkpoint`
### 6.1.4 生产环境相关
- 数据量(高峰期以 100W 为例)
	- 100W 日活，每个用户 100 条数据共计 1 亿条数据
	- 1 条 / k  共计 100G 左右
	- 离线最大的表是行为日志
	- 实时关心的是高峰期数据
	- 平均 `1亿 / (3600 * 24) = 1150 条/s`
	- 高峰期速度是均值的 2-20 倍  2000-20000 条  20M/s
- 主题数
	- ods: 2
	- dwd: 
		- 日志分流: 启动、页面、曝光、错误、行为产生 5 个
		- 业务主题分流产生 4 个
		- 大概 20-30 个
- 副本数 2-3
- 分区数 按主题分
	- 日志数据主题数据量最大，6 个分区
	- 其他主题，2-3 个分区
- 压测
	- `Kafka` 自带压测脚本
	- `kafka-producer-perf-test.sh` (<font color='red'>perf</font>ormance)
		- --record-size: 一条消息的大小(byte)
		- --num-records
		- --throughput: 每秒的吞吐量
	- `kafka-consumer-perf-test.sh` 
		- --broker-list
		- --topic
		- --fetch-size
		- --messages
- 机器台数: `峰值速度 / min(生产速率，消费速率)`
- 监控：通过 eagle
	- 生产、消费速度
	- lag: 消费数据与 `Kafka` 中 `offset` 的差值，是否存在数据积压问题
- 保存天数更改为 3 天
## 6.2 挂了
- 多副本可以保证数据不丢
- Flume，日志文件解耦
- 大部分情况下是内存的原因
## 6.3 数据丢失
- 生产者丢数据
	- `ack` 不是 -1
	- 解决：将 `ack` 设置为 -1，如果此时 `follower` 副本超过了同步时间，进入 `osr` 队列时，此时只会有 `leader` 进行响应，相当于 `ack` 变为了 1
	- 因此还需要 `isr` 中副本数 >= 2
- 消费者丢数据
	- 消费者消费到数据后，先保存 `offset`，再保存数据
	- 保存 `offset` 后，任务挂掉，重启后会接着 `offset` 进行消费，导致数据丢失
	- 解决：先保存数据，再保存 `offset`
## 6.4 数据重复
- 生产者
	- `ack` 设置为-1 时有可能导致数据重复
	- 解决：幂等，事务
- 消费者
	- 先保存数据，再 保存 offset 有可能导致数据重复
	- 解决：事务写出，下游去重(幂等性)
	- `doris` `replace`
## 6.5 数据积压
- 生产速度 > 消费速度
- 生产环境只能提高消费速度
	- 提高单次拉取 `ConsumerRecord` 的个数 `max.poll.records, 默认500`
	- 提高单次拉取的数据量大小 `fetch.max.bytes, 默认50 * 1024 * 1024`
	- 数据的处理速度跟不上(数据倾斜，反压，关联维表)，不会去 fetch 下一次数据
	- 提高下游处理能力：增大资源，旁路缓存，异步 IO
## 6.6 其他
- Kafka 使用了两个系统调用
	- 直接内存映射 MMAP
	- sendFile 零拷贝
- 内存：10-15G
- 高效的原因：
	- 分区
	- 稀疏索引
	- 页缓存
	- 零拷贝
		- 传统的网络 I/O:
			- 操作系统从磁盘把数据读到内核区
			- 用户进程把数据从内核区拷贝到用户区
			- 用户进程将数据写入 `Socket`, 数据到内核区的 `Socket Buffer` 上
			- 最后将数据从 `Socket Buffer` 发送到网卡
		- `sendFile` 优化：
			- 直接把数据从内核区拷贝到 `Socket`
			- 然后发送到网卡
			- 避免了在内核 `Buffer` 和用户 `Buffer` 来回拷贝的弊端
	- 顺序写磁盘
- 能不能消费到 7 天前的数据
	- `Kafka` `segment` 的默认大小是 `1GB`
	- 到达默认的删除时间是，会按照文件进行删除，需要文件中的最后一条数据到达 7 天才会删除
	- 有可能一个文件中同时存在 7 天和 7 天前的数据
- 如果新增了一条维度信息，马上进行了修改，修改数据比新增数据先写出到 `HBase`，如何解决
	- 考察如何保证 `Kafka`  的数据有序性 
	- 将新增和更新的数据发往同一个分区 -> 保证 `key` 相同 -> 按主键作为 `Kafka` 的 `key` 值
	- `maxwell` 的配置文件中，通过 `producer_partition_by=primary_key` 设置
	- 仅限于 `HBase`：将事件时间作为版本写入到 `HBase` 中
- 来一条 2M 的数据，`Kafka` 会产生什么现象
	- 卡死，默认的最大的数据大小为 1M
## 6.7 生产流程
- 创建一个 `KafkaProducer` 对象，给 `Producer` 配置拦截器，序列化器，分区器，创建一个 `RecordAccumulator` 对象以及一个线程池，开启 `Sender` 线程，创建 `NetworkClient` 用于连接 `Broker`
- 调用 `send` 方法，数据经过拦截器，序列化器，分区器后发送到 `RecordAccumulator` 中，未达到数据发送条件时，`sender` 线程会阻塞
- 当数据达到 `batchSize` (`batch.size 默认16k`) 或到达等待时间(`linger.ms 默认0s`)，会唤醒 `Sender` 线程，通过 `Selector` 选择 `Channel` 向 `Leader` 发送 `ProducerRequest`
- `Leader` 响应 `Producer` 的请求，完成写入，返回 `ack`
## 6.8 Broker 工作流程
- `KafkaServer` 在启动时会创建 `Controller`，`ReplicaManager`，`LogManager`，`GroupCoordinator` 等组件并启动
- `Controller` 启动后会去 `Zk` 创建节点并注册 `Watcher`，先创建节点的成为 `leader`，如果 `leader` 节点挂了，`zk` 监听到节点的变化，就会触发重选举
- `Leader` `Controller` 所在节点的 `ReplicaManager` 会选举 `Partition` 的 `leader`，以 `isr` 中存活为前提，按照 `AR` 排在前面的优先
- `ReplicaManager` 会获取 `isr`，由 `controller` 将节点信息上传到 `zk`
## 6.9 消费者组消费流程
- `Consumer` 向 `GroupCoordinator` 发起 `joinGroup` 请求，先发送的成为 `leader` `consumer`
- `GroupCoordinator` 向 `leader consumer` 发送待消费主题的信息
- `leader consumer` 制定消费方案发给 `GroupCoordinator`
- `GroupCoordinator` 将消费方案发送给每个 `Consumer`
- 每个 `Consumer` 和 `GroupCoordinator` 保持心跳，一旦超时，或者处理消息时间过长，就会被移除并触发再平衡
# 6 面试
- 1 Kafka 创建过多的 topic 会有什么影响
	- topic 数量过多，总分区数也会增加，影响磁盘读写
	- 存储在 zk 中的元数据会占很大空间
	- 日志文件过多，选举分区的 leader 时所需要的时间也会大大增加
- 2 leader 和 follower 的数据同步
	- 每个 partition 有多个 replica
	- partition 接收消息和消费消息在 leader 中进行，follower 的作用是备份 leader 的消息
	- partition 中的所有副本称为 AR，所有与 leader 副本保持同步的副本组成 ISR
	- follower 数据落后太多或长时间未向 leader 发起同步的，移除 ISR
- 3 消费者组中的消费者数量超过分区数怎么办
	- 同一消费者组的消费者数量超过分区数时，多余的消费者会处于空闲状态，只有当其中一个消费者离开消费者组后，其他消费者才会重新进行分区分配
- 4 Kafka 事务机制
	- `KafkaProducer` 初始化时会创建 `TransactionManager`
	- `TransactionManager` 通过 `transactional.id` 设置事务 id
	- `TransactionManager` 通过 `initializeTransactions()` 方法创建一个 `InitProducerIdRequest`
	- `Broker` 通过 `TransactionCoordinator` 处理请求，生成 `producerId`
	- `TransactionManager` 通过 `beginTransaction()` 方法开启事务
	- `TransactionManager` 通过 `beginCommit()` 方法提交事务，生成一个 `EndTxnRequest` 发送给 `Broker`
	- `TransactionCoordinator` 通过 `endTransaction` () 方法进行提交，结束事务
- 5 新增消费者如何执行再平衡
	- 新增消费者，消费者组状态变为 `PreparingRebalance`
	- `ConsumerCoordinator` 寻找到 `GroupCoordinator`
	- `consumer` 发起 `JoinGroup` 请求
	- `GroupCoordinator` 处理请求，新的消费者的 `memberId` 为 `""`，通过 `doUnknownJoinGroup()` 方法处理
	- 为 `consumer` 创建 `memberId`，加入组，第一个入组的成员成为 `leader`，执行重平衡
	- 选择消费者组的分配协议，取 `member` 中票数最多的，完成平衡
- 6 Kafka 和传统消息队列的区别
	- 存储方式：
		- 传统消息队列采用 FIFO 的方式存储消息
		- Kafka 采用日志形式存储消息
	- 扩展性
	- 消息保证：Kafka 可以实现精准一次
	- 消息存储时间