---
layout: article
title: Flink-Kafka-Producer配置
date: 2019/10/7 17:12
categories: Flink Kafka
---



# 前言
这里因一个线上异常，而对flink-connection-kafka的源码的进一步理解。

## `TimeoutException`
```java
org.apache.flink.streaming.connectors.kafka.FlinkKafkaException: Failed to send data to Kafka: Expiring 43 record(s) for xxxxx-task-name-7: 32322 ms has passed since last append
	at org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer.checkErroneous(FlinkKafkaProducer.java:1000)
	at org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer.invoke(FlinkKafkaProducer.java:617)
	at org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer.invoke(FlinkKafkaProducer.java:95)
	at org.apache.flink.streaming.api.functions.sink.TwoPhaseCommitSinkFunction.invoke(TwoPhaseCommitSinkFunction.java:230)
	at org.apache.flink.streaming.api.operators.StreamSink.processElement(StreamSink.java:56)
	at org.apache.flink.streaming.runtime.io.StreamInputProcessor.processInput(StreamInputProcessor.java:202)
	at org.apache.flink.streaming.runtime.tasks.OneInputStreamTask.run(OneInputStreamTask.java:105)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.invoke(StreamTask.java:300)
	at org.apache.flink.runtime.taskmanager.Task.run(Task.java:704)
	at java.lang.Thread.run(Thread.java:745)
Caused by: org.apache.kafka.common.errors.TimeoutException: Expiring 43 record(s) for xxxxx-task-name-7: 32322 ms has passed since last append
```

上述问题的原因是kafka消息的callback超过的默认的`request-timeout-ms`，所有增加即可。

### 解决办法

在flink 的producer同样也需要配置`request.timeout.ms`，默认值是30秒。
```java
  // 增大为5分钟。
  properties.setProperty("request.timeout.ms", "600000");
```

## 版本
下面的flink代码是Flink-1.9-SNAPSHOT。

# Flink-Producer
## FlinkKafkaProducer初始化
`FlinkKafkaProducer`继承了`TwoPhaseCommitSinkFunction`用于实现`exactly-once`语义，下面是其初始化代码，因为在其初始化代码中仅用到了`key.serializer`,`value.serializer`,`bootstrap.servers`,`transaction.timeout.ms`这四个配置属性，所以就天真认为flink kafka producer并不需要配置`request.timeout.ms`了。
```java
  public FlinkKafkaProducer(String defaultTopicId, KeyedSerializationSchema<IN> serializationSchema, Properties producerConfig, Optional<FlinkKafkaPartitioner<IN>> customPartitioner, FlinkKafkaProducer.Semantic semantic, int kafkaProducersPoolSize) {
    super(new FlinkKafkaProducer.TransactionStateSerializer(), new FlinkKafkaProducer.ContextStateSerializer());
    this.availableTransactionalIds = new LinkedBlockingDeque();
    this.writeTimestampToKafka = false;
    this.pendingRecords = new AtomicLong();
    this.previouslyCreatedMetrics = new HashMap();
    this.defaultTopicId = (String)Preconditions.checkNotNull(defaultTopicId, "defaultTopicId is null");
    this.schema = (KeyedSerializationSchema)Preconditions.checkNotNull(serializationSchema, "serializationSchema is null");
    this.producerConfig = (Properties)Preconditions.checkNotNull(producerConfig, "producerConfig is null");
    this.flinkKafkaPartitioner = (FlinkKafkaPartitioner)((Optional)Preconditions.checkNotNull(customPartitioner, "customPartitioner is null")).orElse((Object)null);
    this.semantic = (FlinkKafkaProducer.Semantic)Preconditions.checkNotNull(semantic, "semantic is null");
    this.kafkaProducersPoolSize = kafkaProducersPoolSize;
    Preconditions.checkState(kafkaProducersPoolSize > 0, "kafkaProducersPoolSize must be non empty");
    ClosureCleaner.clean(this.flinkKafkaPartitioner, true);
    ClosureCleaner.ensureSerializable(serializationSchema);
    if (!producerConfig.containsKey("key.serializer")) {
      this.producerConfig.put("key.serializer", ByteArraySerializer.class.getName());
    } else {
      LOG.warn("Overwriting the '{}' is not recommended", "key.serializer");
    }

    if (!producerConfig.containsKey("value.serializer")) {
      this.producerConfig.put("value.serializer", ByteArraySerializer.class.getName());
    } else {
      LOG.warn("Overwriting the '{}' is not recommended", "value.serializer");
    }

    if (!this.producerConfig.containsKey("bootstrap.servers")) {
      throw new IllegalArgumentException("bootstrap.servers must be supplied in the producer config properties.");
    } else {
      if (!producerConfig.containsKey("transaction.timeout.ms")) {
        long timeout = DEFAULT_KAFKA_TRANSACTION_TIMEOUT.toMilliseconds();
        Preconditions.checkState(timeout < 2147483647L && timeout > 0L, "timeout does not fit into 32 bit integer");
        this.producerConfig.put("transaction.timeout.ms", (int)timeout);
        LOG.warn("Property [{}] not specified. Setting it to {}", "transaction.timeout.ms", DEFAULT_KAFKA_TRANSACTION_TIMEOUT);
      }

      if (semantic == FlinkKafkaProducer.Semantic.EXACTLY_ONCE) {
        Object object = this.producerConfig.get("transaction.timeout.ms");
        long transactionTimeout;
        if (object instanceof String && StringUtils.isNumeric((String)object)) {
          transactionTimeout = Long.parseLong((String)object);
        } else {
          if (!(object instanceof Number)) {
            throw new IllegalArgumentException("transaction.timeout.ms must be numeric, was " + object);
          }

          transactionTimeout = ((Number)object).longValue();
        }

        super.setTransactionTimeout(transactionTimeout);
        super.enableTransactionTimeoutWarnings(0.8D);
      }

      this.topicPartitionsMap = new HashMap();
    }
  }
```

## Kafka producer初始化

### `FlinkKafkaInternalProducer`
这个是个flink继承kafka produce接口实现的flink内部的kafka producer实现类。

```java
  public class FlinkKafkaInternalProducer<K, V> implements Producer<K, V> {
    ...
    public FlinkKafkaInternalProducer(Properties properties) {
      transactionalId = properties.getProperty(ProducerConfig.TRANSACTIONAL_ID_CONFIG);
      kafkaProducer = new KafkaProducer<>(properties);
	}
    ...
}
```

### `KafkaProducer`
A Kafka client that publishes records to the Kafka cluster.这是kafka的producer实现，下面是其构造函数.

```java
  public class KafkaProducer<K, V> implements Producer<K, V> {
    ...
    KafkaProducer(Map<String, Object> configs,
                  Serializer<K> keySerializer,
                  Serializer<V> valueSerializer,
                  Metadata metadata,
                  KafkaClient kafkaClient,
                  ProducerInterceptors interceptors,
                  Time time) {
    ProducerConfig config = new ProducerConfig(ProducerConfig.addSerializerToConfig(configs, keySerializer,
                valueSerializer));
    try { 
      Map<String, Object> userProvidedConfigs = config.originals();
      // 给到了一个实例属性变量中
      this.producerConfig = config;
      this.time = time;
      ...
      // 在这里配置包括request-timeout-ms在内部分配置。
      int deliveryTimeoutMs = configureDeliveryTimeout(config, log);
      ...
    } catch (Throwable t) {
      // call close methods if internal objects are already constructed this is to prevent resource leak. see KAFKA-2121
      close(Duration.ofMillis(0), true);
      // now propagate the exception
      throw new KafkaException("Failed to construct kafka producer", t);
      }
    }
    
    private static int configureDeliveryTimeout(ProducerConfig config, Logger log) {
      int deliveryTimeoutMs = config.getInt(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG);
      int lingerMs = config.getInt(ProducerConfig.LINGER_MS_CONFIG);
      // 配置timeoutMs
      int requestTimeoutMs = config.getInt(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG);
      if (deliveryTimeoutMs < Integer.MAX_VALUE && deliveryTimeoutMs < lingerMs + requestTimeoutMs) {
        if (config.originals().containsKey(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG)) {
          // throw an exception if the user explicitly set an inconsistent value
          throw new ConfigException(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG
              + " should be equal to or larger than " + ProducerConfig.LINGER_MS_CONFIG
              + " + " + ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG);
        } else {
          // override deliveryTimeoutMs default value to lingerMs + requestTimeoutMs for backward compatibility
          deliveryTimeoutMs = lingerMs + requestTimeoutMs;
          log.warn("{} should be equal to or larger than {} + {}. Setting it to {}.",
               ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, ProducerConfig.LINGER_MS_CONFIG,
               ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, deliveryTimeoutMs);
        }
      }
      return deliveryTimeoutMs;
   }
   ...
}
```

## Flink kafka 配置初始化的流程

1. `TwoPhaseCommitSinkFunction#initializeState`

```java
  public void initializeState(FunctionInitializationContext context) throws Exception {
    ...
    // 开始一个事务
    currentTransactionHolder = beginTransactionInternal();
    LOG.debug("{} - started new transaction '{}'", name(), currentTransactionHolder);
  }
```

2. `TwoPhaseCommitSinkFunction#beginTransactionInternal`

```java
  private TransactionHolder<TXN> beginTransactionInternal() throws Exception {
    // beginTransaction 仅仅是一个抽象方法具体实现在FlinkKafkaProducer
  	return new TransactionHolder<>(beginTransaction(), clock.millis());
  }
```

3. `FlinkKafkaProducer#beginTransaction`
```java
  protected FlinkKafkaProducer.KafkaTransactionState beginTransaction() throws FlinkKafkaException {
    switch(this.semantic) {
    case EXACTLY_ONCE:
      // 这里
      FlinkKafkaInternalProducer<byte[], byte[]> producer = this.createTransactionalProducer();
      producer.beginTransaction();
      return new FlinkKafkaProducer.KafkaTransactionState(producer.getTransactionalId(), producer);
    case AT_LEAST_ONCE:
    case NONE:
      FlinkKafkaProducer.KafkaTransactionState currentTransaction = (FlinkKafkaProducer.KafkaTransactionState)this.currentTransaction();
      if (currentTransaction != null && currentTransaction.producer != null) {
        return new FlinkKafkaProducer.KafkaTransactionState(currentTransaction.producer);
      }

      // 这里
      return new FlinkKafkaProducer.KafkaTransactionState(this.initNonTransactionalProducer(true));
    default:
      throw new UnsupportedOperationException("Not implemented semantic");
    }
  }
```

4. `FlinkKafkaProducer#initProducer`

```java
  // 开启事务
  private FlinkKafkaInternalProducer<byte[], byte[]> initTransactionalProducer(String transactionalId, boolean registerMetrics) {
    this.producerConfig.put("transactional.id", transactionalId);
    return this.initProducer(registerMetrics);
  }

  // 不开启事务
  private FlinkKafkaInternalProducer<byte[], byte[]> initNonTransactionalProducer(boolean registerMetrics) {
    this.producerConfig.remove("transactional.id");
    return this.initProducer(registerMetrics);
  }
  
  private FlinkKafkaInternalProducer<byte[], byte[]> initProducer(boolean registerMetrics) {
    // 初始化 FlinkKafkaInternalProducer，参见上小节
    FlinkKafkaInternalProducer<byte[], byte[]> producer = this.createProducer();
    ...
    }
```

4. 

# kafka procured config
## Kafka源码中的配置解释
详细procuder的配置参见[kafka官方文档](http://kafka.apache.org/documentation.html#producerconfigs)。
也就是说flink的kafka producer所需要配置或者说是能够配置的参数就是kafka自身producer提供的配置参数。flink仅仅是基于kafka自己实现了可以满足分布式事务的exactly-once语义的扩展。

```java
  public class ProducerConfig extends AbstractConfig {
    /** <code>batch.size</code> */
    public static final String BATCH_SIZE_CONFIG = "batch.size";
    private static final String BATCH_SIZE_DOC = "The producer will attempt to batch records together into fewer requests whenever multiple records are being sent"
                                                 + " to the same partition. This helps performance on both the client and the server. This configuration controls the "
                                                 + "default batch size in bytes. "
                                                 + "<p>"
                                                 + "No attempt will be made to batch records larger than this size. "
                                                 + "<p>"
                                                 + "Requests sent to brokers will contain multiple batches, one for each partition with data available to be sent. "
                                                 + "<p>"
                                                 + "A small batch size will make batching less common and may reduce throughput (a batch size of zero will disable "
                                                 + "batching entirely). A very large batch size may use memory a bit more wastefully as we will always allocate a "
                                                 + "buffer of the specified batch size in anticipation of additional records.";

    ...
```

## 日志打印的配置
下面是日志大打印的Producer的所有Config，推测下面所有打印而出的配置应该都是可以配置的参数。
```bash
16:25:18,162 INFO  org.apache.kafka.clients.producer.ProducerConfig              - ProducerConfig values: 
	acks = 1
	batch.size = 16384
	bootstrap.servers = [xxx,xxx,xxx]
	buffer.memory = 33554432
	client.id = kafka-client
	compression.type = none
	connections.max.idle.ms = 540000
	enable.idempotence = false
	interceptor.classes = []
	key.serializer = class org.apache.kafka.common.serialization.ByteArraySerializer
	linger.ms = 0
	max.block.ms = 60000
	max.in.flight.requests.per.connection = 5
	max.request.size = 1048576
	metadata.max.age.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partitioner.class = class org.apache.kafka.clients.producer.internals.DefaultPartitioner
	receive.buffer.bytes = 32768
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 600000
	retries = 120
	retry.backoff.ms = 500
	sasl.client.callback.handler.class = null
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.login.callback.handler.class = null
	sasl.login.class = null
	sasl.login.refresh.buffer.seconds = 300
	sasl.login.refresh.min.period.seconds = 60
	sasl.login.refresh.window.factor = 0.8
	sasl.login.refresh.window.jitter = 0.05
	sasl.mechanism = GSSAPI
	security.protocol = PLAINTEXT
	send.buffer.bytes = 131072
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	ssl.endpoint.identification.algorithm = https
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLS
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	transaction.timeout.ms = 3600000
	transactional.id = null
	value.serializer = class org.apache.kafka.common.serialization.ByteArraySerializer
```

