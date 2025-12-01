# RocketMQ 消息队列最佳实践

## 角色设定

你是一位精通 RocketMQ 5.x 的消息中间件专家，擅长消息可靠性、顺序消息、事务消息和高可用架构。

## 提示词模板

### 消息方案设计

```
请帮我设计 RocketMQ 消息方案：
- 业务场景：[描述业务]
- 消息类型：[普通/顺序/延迟/事务]
- 可靠性要求：[至少一次/至多一次/精确一次]
- 吞吐量：[预估 QPS]
- 延迟要求：[实时/准实时/可延迟]

请提供：
1. Topic 设计
2. 生产者配置
3. 消费者配置
4. 异常处理
```

## 核心代码示例

### Spring Boot 配置

```yaml
# application.yml
rocketmq:
  name-server: ${ROCKETMQ_NAMESRV:localhost:9876}
  producer:
    group: ${spring.application.name}-producer
    send-message-timeout: 3000
    retry-times-when-send-failed: 2
    retry-times-when-send-async-failed: 2
  consumer:
    group: ${spring.application.name}-consumer
    pull-batch-size: 32
```

### 生产者

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderMessageProducer {

    private final RocketMQTemplate rocketMQTemplate;

    /**
     * 发送普通消息
     */
    public void sendOrderCreatedMessage(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getUserId(),
            order.getTotalAmount(),
            LocalDateTime.now()
        );

        rocketMQTemplate.convertAndSend("order-topic:created", event);
        log.info("Sent order created message: {}", order.getId());
    }

    /**
     * 发送同步消息 (等待 Broker 确认)
     */
    public SendResult sendSync(String topic, Object payload) {
        Message<Object> message = MessageBuilder
            .withPayload(payload)
            .setHeader(MessageConst.PROPERTY_KEYS, UUID.randomUUID().toString())
            .build();

        SendResult result = rocketMQTemplate.syncSend(topic, message);
        log.info("Sync send result: {}", result.getSendStatus());
        return result;
    }

    /**
     * 发送异步消息
     */
    public void sendAsync(String topic, Object payload) {
        Message<Object> message = MessageBuilder.withPayload(payload).build();

        rocketMQTemplate.asyncSend(topic, message, new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {
                log.info("Async send success: {}", sendResult.getMsgId());
            }

            @Override
            public void onException(Throwable e) {
                log.error("Async send failed", e);
                // 重试或补偿逻辑
            }
        });
    }

    /**
     * 发送延迟消息
     */
    public void sendDelayedMessage(String topic, Object payload, int delayLevel) {
        // delayLevel: 1=1s, 2=5s, 3=10s, 4=30s, 5=1m, 6=2m, 7=3m...
        Message<Object> message = MessageBuilder.withPayload(payload).build();
        rocketMQTemplate.syncSend(topic, message, 3000, delayLevel);
    }

    /**
     * 发送顺序消息
     */
    public void sendOrderlyMessage(String topic, Object payload, String orderId) {
        // 相同 orderId 的消息会发送到同一个队列，保证顺序
        rocketMQTemplate.syncSendOrderly(topic, payload, orderId);
    }

    /**
     * 发送事务消息
     */
    public void sendTransactionMessage(Order order) {
        Message<OrderCreatedEvent> message = MessageBuilder
            .withPayload(new OrderCreatedEvent(order))
            .setHeader("orderId", order.getId())
            .build();

        rocketMQTemplate.sendMessageInTransaction(
            "order-topic:transaction",
            message,
            order // 传递给本地事务执行器
        );
    }
}
```

### 消费者

```java
@Service
@RocketMQMessageListener(
    topic = "order-topic",
    selectorExpression = "created",
    consumerGroup = "order-consumer-group",
    consumeMode = ConsumeMode.CONCURRENTLY,
    messageModel = MessageModel.CLUSTERING
)
@Slf4j
public class OrderCreatedConsumer implements RocketMQListener<OrderCreatedEvent> {

    @Autowired
    private OrderService orderService;

    @Override
    public void onMessage(OrderCreatedEvent event) {
        log.info("Received order created event: {}", event.getOrderId());

        try {
            // 幂等性检查
            if (orderService.isProcessed(event.getOrderId())) {
                log.warn("Order already processed: {}", event.getOrderId());
                return;
            }

            // 处理业务逻辑
            orderService.processOrderCreated(event);

            log.info("Order processed successfully: {}", event.getOrderId());
        } catch (Exception e) {
            log.error("Failed to process order: {}", event.getOrderId(), e);
            // 抛出异常会触发重试
            throw new RuntimeException("Process order failed", e);
        }
    }
}

/**
 * 顺序消费者
 */
@Service
@RocketMQMessageListener(
    topic = "order-topic",
    selectorExpression = "status-change",
    consumerGroup = "order-status-consumer",
    consumeMode = ConsumeMode.ORDERLY // 顺序消费
)
@Slf4j
public class OrderStatusConsumer implements RocketMQListener<OrderStatusEvent> {

    @Override
    public void onMessage(OrderStatusEvent event) {
        log.info("Received order status event: {} -> {}",
            event.getOrderId(), event.getNewStatus());
        // 顺序处理订单状态变更
    }
}

/**
 * 批量消费者
 */
@Service
@RocketMQMessageListener(
    topic = "batch-topic",
    consumerGroup = "batch-consumer-group",
    consumeMode = ConsumeMode.CONCURRENTLY
)
@Slf4j
public class BatchConsumer implements RocketMQListener<List<MessageExt>> {

    @Override
    public void onMessage(List<MessageExt> messages) {
        log.info("Received batch messages, size: {}", messages.size());
        for (MessageExt message : messages) {
            // 处理每条消息
            processMessage(message);
        }
    }
}
```

### 事务消息

```java
@RocketMQTransactionListener
@Slf4j
public class OrderTransactionListener implements RocketMQLocalTransactionListener {

    @Autowired
    private OrderService orderService;

    /**
     * 执行本地事务
     */
    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        Order order = (Order) arg;
        log.info("Execute local transaction for order: {}", order.getId());

        try {
            // 执行本地事务 (创建订单)
            orderService.createOrderInTransaction(order);
            return RocketMQLocalTransactionState.COMMIT;
        } catch (Exception e) {
            log.error("Local transaction failed", e);
            return RocketMQLocalTransactionState.ROLLBACK;
        }
    }

    /**
     * 事务状态回查
     */
    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        String orderId = msg.getHeaders().get("orderId", String.class);
        log.info("Check local transaction for order: {}", orderId);

        // 查询订单是否创建成功
        Order order = orderService.findById(Long.valueOf(orderId));
        if (order != null && order.getStatus() == OrderStatus.CREATED) {
            return RocketMQLocalTransactionState.COMMIT;
        } else if (order == null) {
            return RocketMQLocalTransactionState.ROLLBACK;
        } else {
            return RocketMQLocalTransactionState.UNKNOWN;
        }
    }
}
```

### 消息幂等处理

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class IdempotentService {

    private final StringRedisTemplate redisTemplate;
    private static final String IDEMPOTENT_KEY_PREFIX = "mq:idempotent:";
    private static final long IDEMPOTENT_TTL = 24; // 小时

    /**
     * 检查并标记消息已处理
     */
    public boolean checkAndMark(String messageId) {
        String key = IDEMPOTENT_KEY_PREFIX + messageId;
        Boolean success = redisTemplate.opsForValue()
            .setIfAbsent(key, "1", IDEMPOTENT_TTL, TimeUnit.HOURS);
        return Boolean.TRUE.equals(success);
    }

    /**
     * 移除标记 (处理失败时调用)
     */
    public void removeMark(String messageId) {
        String key = IDEMPOTENT_KEY_PREFIX + messageId;
        redisTemplate.delete(key);
    }
}

// 消费者中使用
@Override
public void onMessage(MessageExt message) {
    String messageId = message.getMsgId();

    // 幂等检查
    if (!idempotentService.checkAndMark(messageId)) {
        log.warn("Duplicate message: {}", messageId);
        return;
    }

    try {
        // 处理业务
        processMessage(message);
    } catch (Exception e) {
        // 处理失败，移除标记，允许重试
        idempotentService.removeMark(messageId);
        throw e;
    }
}
```

### 死信队列处理

```java
@Service
@RocketMQMessageListener(
    topic = "%DLQ%order-consumer-group", // 死信队列 Topic
    consumerGroup = "dlq-consumer-group"
)
@Slf4j
public class DeadLetterConsumer implements RocketMQListener<MessageExt> {

    @Autowired
    private AlertService alertService;

    @Autowired
    private DeadLetterRepository deadLetterRepository;

    @Override
    public void onMessage(MessageExt message) {
        log.error("Dead letter message received: {}", message.getMsgId());

        // 保存到数据库
        DeadLetterMessage dlm = new DeadLetterMessage();
        dlm.setMessageId(message.getMsgId());
        dlm.setTopic(message.getTopic());
        dlm.setTags(message.getTags());
        dlm.setBody(new String(message.getBody()));
        dlm.setRetryTimes(message.getReconsumeTimes());
        dlm.setCreateTime(LocalDateTime.now());
        deadLetterRepository.save(dlm);

        // 发送告警
        alertService.sendAlert(
            "死信消息告警",
            "MessageId: " + message.getMsgId() + ", Topic: " + message.getTopic()
        );
    }
}
```

## Topic 设计规范

| 场景 | Topic 命名 | Tag 示例 |
|------|------------|----------|
| 订单 | order-topic | created / paid / shipped / completed |
| 用户 | user-topic | registered / updated / deleted |
| 支付 | payment-topic | success / failed / refund |
| 库存 | inventory-topic | deduct / rollback / sync |
| 通知 | notification-topic | sms / email / push |

## 消息可靠性保证

```
生产端:
1. 同步发送 + 重试
2. 本地消息表 + 定时扫描

Broker 端:
1. 同步刷盘
2. 主从同步复制

消费端:
1. 手动 ACK
2. 幂等消费
3. 死信队列兜底
```

## 最佳实践清单

- [ ] Topic 和 ConsumerGroup 命名规范
- [ ] 重要消息使用同步发送
- [ ] 消费者实现幂等处理
- [ ] 设置合理的重试次数
- [ ] 配置死信队列
- [ ] 监控消息堆积
- [ ] 顺序消息使用顺序消费模式
- [ ] 事务消息实现回查逻辑
