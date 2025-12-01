# Kafka 消息队列最佳实践

## 角色设定

你是一位精通 Apache Kafka 的消息中间件专家，擅长高吞吐消息处理、分区策略和消费者组管理。

## 提示词模板

### Kafka 方案设计

```
请帮我设计 Kafka 消息方案：
- 业务场景：[描述业务]
- 吞吐量要求：[预估 QPS]
- 消息大小：[消息体大小]
- 顺序要求：[全局有序/分区有序/无序]
- 可靠性要求：[至少一次/至多一次/精确一次]

请提供：
1. Topic 设计
2. 分区策略
3. 生产者配置
4. 消费者配置
```

## 核心代码示例

### Spring Kafka 配置

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      batch-size: 16384
      buffer-memory: 33554432
      properties:
        linger.ms: 10
        enable.idempotence: true
    consumer:
      group-id: ${spring.application.name}
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: "*"
        max.poll.records: 500
        max.poll.interval.ms: 300000
    listener:
      ack-mode: manual_immediate
      concurrency: 3
```

### 生产者

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class KafkaProducerService {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    /**
     * 同步发送
     */
    public void sendSync(String topic, String key, Object message) {
        try {
            SendResult<String, Object> result = kafkaTemplate.send(topic, key, message).get();
            RecordMetadata metadata = result.getRecordMetadata();
            log.info("Sent message to topic={}, partition={}, offset={}",
                metadata.topic(), metadata.partition(), metadata.offset());
        } catch (Exception e) {
            log.error("Failed to send message", e);
            throw new RuntimeException("Send message failed", e);
        }
    }

    /**
     * 异步发送
     */
    public void sendAsync(String topic, String key, Object message) {
        CompletableFuture<SendResult<String, Object>> future =
            kafkaTemplate.send(topic, key, message);

        future.whenComplete((result, ex) -> {
            if (ex == null) {
                RecordMetadata metadata = result.getRecordMetadata();
                log.info("Async sent to partition={}, offset={}",
                    metadata.partition(), metadata.offset());
            } else {
                log.error("Failed to send message", ex);
                // 补偿逻辑
            }
        });
    }

    /**
     * 发送到指定分区
     */
    public void sendToPartition(String topic, int partition, String key, Object message) {
        kafkaTemplate.send(topic, partition, key, message);
    }

    /**
     * 带回调的发送
     */
    public void sendWithCallback(String topic, Object message,
            Consumer<SendResult<String, Object>> onSuccess,
            Consumer<Throwable> onFailure) {
        kafkaTemplate.send(topic, message).whenComplete((result, ex) -> {
            if (ex == null) {
                onSuccess.accept(result);
            } else {
                onFailure.accept(ex);
            }
        });
    }
}
```

### 消费者

```java
@Service
@Slf4j
public class OrderKafkaConsumer {

    @Autowired
    private OrderService orderService;

    @Autowired
    private IdempotentService idempotentService;

    /**
     * 单条消费
     */
    @KafkaListener(
        topics = "order-topic",
        groupId = "order-consumer-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consume(ConsumerRecord<String, OrderEvent> record, Acknowledgment ack) {
        String messageId = record.key();
        OrderEvent event = record.value();

        log.info("Received message: topic={}, partition={}, offset={}, key={}",
            record.topic(), record.partition(), record.offset(), messageId);

        try {
            // 幂等检查
            if (!idempotentService.checkAndMark(messageId)) {
                log.warn("Duplicate message: {}", messageId);
                ack.acknowledge();
                return;
            }

            // 处理业务
            orderService.processOrder(event);

            // 手动提交
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Failed to process message: {}", messageId, e);
            idempotentService.removeMark(messageId);
            // 不提交，会重试
            throw e;
        }
    }

    /**
     * 批量消费
     */
    @KafkaListener(
        topics = "batch-topic",
        groupId = "batch-consumer-group",
        containerFactory = "batchKafkaListenerContainerFactory"
    )
    public void consumeBatch(List<ConsumerRecord<String, Object>> records, Acknowledgment ack) {
        log.info("Received batch: size={}", records.size());

        try {
            List<Object> messages = records.stream()
                .map(ConsumerRecord::value)
                .toList();

            // 批量处理
            processBatch(messages);

            ack.acknowledge();
        } catch (Exception e) {
            log.error("Batch processing failed", e);
            throw e;
        }
    }

    /**
     * 并发消费 - 多分区
     */
    @KafkaListener(
        topics = "concurrent-topic",
        groupId = "concurrent-group",
        concurrency = "3"  // 3个消费者线程
    )
    public void consumeConcurrent(ConsumerRecord<String, Object> record, Acknowledgment ack) {
        log.info("Thread={}, partition={}", Thread.currentThread().getName(), record.partition());
        // 处理逻辑
        ack.acknowledge();
    }
}
```

### 消费者配置

```java
@Configuration
@EnableKafka
public class KafkaConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        // 错误处理
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate()),
            new FixedBackOff(1000L, 3) // 重试3次，间隔1秒
        ));

        return factory;
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> batchKafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(true);  // 批量消费
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}
```

### 自定义分区器

```java
public class OrderPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();

        if (key == null) {
            // 无 key 时轮询
            return ThreadLocalRandom.current().nextInt(numPartitions);
        }

        // 按用户ID分区，保证同一用户的消息顺序
        String userId = key.toString();
        return Math.abs(userId.hashCode()) % numPartitions;
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}
```

### 事务消息

```java
@Service
@RequiredArgsConstructor
public class TransactionalKafkaService {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    /**
     * 事务发送
     */
    @Transactional
    public void sendInTransaction(List<OrderEvent> events) {
        kafkaTemplate.executeInTransaction(operations -> {
            for (OrderEvent event : events) {
                operations.send("order-topic", event.getOrderId(), event);
            }
            return null;
        });
    }

    /**
     * 配合数据库事务
     */
    @Transactional
    public void createOrderWithKafka(Order order) {
        // 1. 保存订单到数据库
        orderRepository.save(order);

        // 2. 发送消息到 Kafka (同一事务)
        kafkaTemplate.send("order-topic", order.getId().toString(), new OrderCreatedEvent(order));
    }
}

// 事务配置
@Bean
public KafkaTransactionManager<String, Object> kafkaTransactionManager(
        ProducerFactory<String, Object> producerFactory) {
    return new KafkaTransactionManager<>(producerFactory);
}
```

## Topic 设计规范

| 业务 | Topic 命名 | 分区数 | 副本数 |
|------|------------|--------|--------|
| 订单 | order-events | 12 | 3 |
| 用户 | user-events | 6 | 3 |
| 日志 | app-logs | 24 | 2 |
| 指标 | metrics | 12 | 2 |

## Kafka vs RocketMQ

| 特性 | Kafka | RocketMQ |
|------|-------|----------|
| 吞吐量 | 更高 | 高 |
| 延迟消息 | 不支持 | 支持 |
| 事务消息 | 支持 | 更完善 |
| 消息回溯 | 按偏移量 | 按时间 |
| 运维复杂度 | 较高 | 较低 |

## 最佳实践清单

- [ ] 合理设置分区数 (建议为消费者数的整数倍)
- [ ] 启用幂等生产者 (enable.idempotence=true)
- [ ] 消费者手动提交 Offset
- [ ] 实现消费幂等
- [ ] 配置死信队列
- [ ] 监控消费者 Lag
- [ ] 按业务 Key 分区保证顺序
- [ ] 批量消费提高吞吐
