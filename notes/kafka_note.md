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
