# Seata 分布式事务最佳实践

## 角色设定

你是一位精通 Seata 的分布式事务专家，擅长 AT、TCC、Saga 模式的应用和全局事务管理。

## 提示词模板

### Seata 方案设计

```
请帮我设计 Seata 分布式事务方案：
- 业务场景：[描述跨服务事务场景]
- 事务模式：[AT/TCC/Saga/XA]
- 数据库类型：[MySQL/PostgreSQL/Oracle]
- 涉及服务：[列出相关服务]
- 一致性要求：[强一致/最终一致]

请提供配置和代码示例。
```

## 核心代码示例

### 依赖配置

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Seata Starter -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
    </dependency>

    <!-- Seata 序列化 -->
    <dependency>
        <groupId>io.seata</groupId>
        <artifactId>seata-serializer-kryo</artifactId>
    </dependency>
</dependencies>
```

### 基础配置

```yaml
# application.yml
spring:
  cloud:
    alibaba:
      seata:
        tx-service-group: ${spring.application.name}-tx-group

seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: ${spring.application.name}-tx-group
  # 注册中心配置
  registry:
    type: nacos
    nacos:
      server-addr: ${NACOS_SERVER:localhost:8848}
      namespace: ${NACOS_NAMESPACE:dev}
      group: SEATA_GROUP
      application: seata-server
  # 配置中心
  config:
    type: nacos
    nacos:
      server-addr: ${NACOS_SERVER:localhost:8848}
      namespace: ${NACOS_NAMESPACE:dev}
      group: SEATA_GROUP
      data-id: seata-server.properties
  # 客户端配置
  client:
    rm:
      async-commit-buffer-limit: 10000
      report-retry-count: 5
      table-meta-check-enable: false
      report-success-enable: false
      saga-branch-register-enable: false
    tm:
      commit-retry-count: 5
      rollback-retry-count: 5
      default-global-transaction-timeout: 60000
      degrade-check: false
    undo:
      data-validation: true
      log-serialization: kryo
      log-table: undo_log
      only-care-update-columns: true
  # 服务端配置
  service:
    vgroup-mapping:
      order-service-tx-group: default
      user-service-tx-group: default
      inventory-service-tx-group: default
```

### Undo Log 表

```sql
-- 每个业务数据库都需要创建
CREATE TABLE IF NOT EXISTS `undo_log` (
  `id` BIGINT(20) NOT NULL AUTO_INCREMENT COMMENT 'increment id',
  `branch_id` BIGINT(20) NOT NULL COMMENT 'branch transaction id',
  `xid` VARCHAR(100) NOT NULL COMMENT 'global transaction id',
  `context` VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
  `rollback_info` LONGBLOB NOT NULL COMMENT 'rollback info',
  `log_status` INT(11) NOT NULL COMMENT '0:normal status,1:defense status',
  `log_created` DATETIME NOT NULL COMMENT 'create datetime',
  `log_modified` DATETIME NOT NULL COMMENT 'modify datetime',
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COMMENT='AT transaction mode undo table';
```

### AT 模式 - 订单服务

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {

    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;
    private final AccountClient accountClient;

    /**
     * 创建订单 - 分布式事务入口
     * @GlobalTransactional 开启全局事务
     */
    @GlobalTransactional(name = "createOrder", rollbackFor = Exception.class)
    public Order createOrder(OrderRequest request) {
        log.info("Creating order, xid: {}", RootContext.getXID());

        // 1. 创建订单
        Order order = Order.builder()
            .orderNo(generateOrderNo())
            .userId(request.getUserId())
            .productId(request.getProductId())
            .quantity(request.getQuantity())
            .amount(request.getAmount())
            .status(OrderStatus.CREATED)
            .build();
        orderRepository.save(order);

        // 2. 扣减库存 (远程调用)
        inventoryClient.deduct(request.getProductId(), request.getQuantity());

        // 3. 扣减账户余额 (远程调用)
        accountClient.deduct(request.getUserId(), request.getAmount());

        // 4. 更新订单状态
        order.setStatus(OrderStatus.PAID);
        orderRepository.save(order);

        log.info("Order created successfully: {}", order.getOrderNo());
        return order;
    }

    /**
     * 带超时的全局事务
     */
    @GlobalTransactional(name = "createOrderWithTimeout",
                         rollbackFor = Exception.class,
                         timeoutMills = 30000)
    public Order createOrderWithTimeout(OrderRequest request) {
        // 业务逻辑
        return null;
    }
}
```

### AT 模式 - 库存服务

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class InventoryService {

    private final InventoryRepository inventoryRepository;

    /**
     * 扣减库存 - AT 模式自动管理
     */
    @Transactional(rollbackFor = Exception.class)
    public void deduct(Long productId, Integer quantity) {
        log.info("Deducting inventory, xid: {}", RootContext.getXID());

        Inventory inventory = inventoryRepository.findByProductId(productId)
            .orElseThrow(() -> new NotFoundException("Product not found"));

        if (inventory.getStock() < quantity) {
            throw new BusinessException(ErrorCode.INSUFFICIENT_STOCK);
        }

        inventory.setStock(inventory.getStock() - quantity);
        inventoryRepository.save(inventory);

        log.info("Inventory deducted: productId={}, quantity={}", productId, quantity);
    }
}
```

### TCC 模式

```java
/**
 * TCC 模式接口定义
 */
@LocalTCC
public interface TccOrderService {

    /**
     * Try - 资源预留
     */
    @TwoPhaseBusinessAction(
        name = "prepareOrder",
        commitMethod = "commit",
        rollbackMethod = "rollback"
    )
    boolean prepare(
        @BusinessActionContextParameter(paramName = "orderId") Long orderId,
        @BusinessActionContextParameter(paramName = "userId") Long userId,
        @BusinessActionContextParameter(paramName = "amount") BigDecimal amount
    );

    /**
     * Confirm - 确认提交
     */
    boolean commit(BusinessActionContext context);

    /**
     * Cancel - 回滚释放
     */
    boolean rollback(BusinessActionContext context);
}

/**
 * TCC 模式实现
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class TccOrderServiceImpl implements TccOrderService {

    private final OrderRepository orderRepository;
    private final TccRecordRepository tccRecordRepository;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean prepare(Long orderId, Long userId, BigDecimal amount) {
        log.info("TCC Prepare - orderId: {}, xid: {}", orderId, RootContext.getXID());

        // 幂等检查
        String xid = RootContext.getXID();
        if (tccRecordRepository.existsByXidAndBranchType(xid, "ORDER_PREPARE")) {
            log.warn("Duplicate prepare request: {}", xid);
            return true;
        }

        // 创建订单 - 冻结状态
        Order order = Order.builder()
            .id(orderId)
            .userId(userId)
            .amount(amount)
            .status(OrderStatus.FROZEN)  // 冻结状态
            .build();
        orderRepository.save(order);

        // 记录 TCC 状态
        TccRecord record = TccRecord.builder()
            .xid(xid)
            .branchType("ORDER_PREPARE")
            .businessKey(orderId.toString())
            .status(TccStatus.PREPARED)
            .build();
        tccRecordRepository.save(record);

        return true;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean commit(BusinessActionContext context) {
        String xid = context.getXid();
        Long orderId = context.getActionContext("orderId", Long.class);

        log.info("TCC Commit - orderId: {}, xid: {}", orderId, xid);

        // 幂等检查
        TccRecord record = tccRecordRepository.findByXidAndBranchType(xid, "ORDER_PREPARE");
        if (record == null || record.getStatus() == TccStatus.COMMITTED) {
            log.warn("Already committed or not found: {}", xid);
            return true;
        }

        // 更新订单状态
        Order order = orderRepository.findById(orderId).orElse(null);
        if (order != null) {
            order.setStatus(OrderStatus.PAID);
            orderRepository.save(order);
        }

        // 更新 TCC 记录
        record.setStatus(TccStatus.COMMITTED);
        tccRecordRepository.save(record);

        return true;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean rollback(BusinessActionContext context) {
        String xid = context.getXid();
        Long orderId = context.getActionContext("orderId", Long.class);

        log.info("TCC Rollback - orderId: {}, xid: {}", orderId, xid);

        // 幂等检查 - 空回滚
        TccRecord record = tccRecordRepository.findByXidAndBranchType(xid, "ORDER_PREPARE");
        if (record == null) {
            log.warn("Empty rollback - record not found: {}", xid);
            // 记录空回滚，防止悬挂
            TccRecord emptyRecord = TccRecord.builder()
                .xid(xid)
                .branchType("ORDER_PREPARE")
                .status(TccStatus.ROLLBACK_EMPTY)
                .build();
            tccRecordRepository.save(emptyRecord);
            return true;
        }

        if (record.getStatus() == TccStatus.ROLLED_BACK) {
            log.warn("Already rolled back: {}", xid);
            return true;
        }

        // 删除订单
        orderRepository.deleteById(orderId);

        // 更新 TCC 记录
        record.setStatus(TccStatus.ROLLED_BACK);
        tccRecordRepository.save(record);

        return true;
    }
}
```

### TCC 账户服务

```java
@LocalTCC
public interface TccAccountService {

    @TwoPhaseBusinessAction(
        name = "prepareDeduct",
        commitMethod = "commit",
        rollbackMethod = "rollback"
    )
    boolean prepare(
        @BusinessActionContextParameter(paramName = "userId") Long userId,
        @BusinessActionContextParameter(paramName = "amount") BigDecimal amount
    );

    boolean commit(BusinessActionContext context);

    boolean rollback(BusinessActionContext context);
}

@Service
@RequiredArgsConstructor
@Slf4j
public class TccAccountServiceImpl implements TccAccountService {

    private final AccountRepository accountRepository;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean prepare(Long userId, BigDecimal amount) {
        log.info("TCC Account Prepare - userId: {}, amount: {}", userId, amount);

        Account account = accountRepository.findByUserId(userId)
            .orElseThrow(() -> new NotFoundException("Account not found"));

        // 检查余额
        if (account.getBalance().compareTo(amount) < 0) {
            throw new BusinessException(ErrorCode.INSUFFICIENT_BALANCE);
        }

        // 冻结金额
        account.setBalance(account.getBalance().subtract(amount));
        account.setFrozenAmount(account.getFrozenAmount().add(amount));
        accountRepository.save(account);

        return true;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean commit(BusinessActionContext context) {
        Long userId = context.getActionContext("userId", Long.class);
        BigDecimal amount = context.getActionContext("amount", BigDecimal.class);

        log.info("TCC Account Commit - userId: {}", userId);

        // 扣减冻结金额
        Account account = accountRepository.findByUserId(userId).orElse(null);
        if (account != null) {
            account.setFrozenAmount(account.getFrozenAmount().subtract(amount));
            accountRepository.save(account);
        }

        return true;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean rollback(BusinessActionContext context) {
        Long userId = context.getActionContext("userId", Long.class);
        BigDecimal amount = context.getActionContext("amount", BigDecimal.class);

        log.info("TCC Account Rollback - userId: {}", userId);

        // 解冻金额
        Account account = accountRepository.findByUserId(userId).orElse(null);
        if (account != null && amount != null) {
            account.setBalance(account.getBalance().add(amount));
            account.setFrozenAmount(account.getFrozenAmount().subtract(amount));
            accountRepository.save(account);
        }

        return true;
    }
}
```

### Saga 模式

```java
/**
 * Saga 状态机配置
 */
@Configuration
public class SagaConfig {

    @Bean
    public StateMachineEngine stateMachineEngine(DataSource dataSource) {
        DbStateMachineConfig config = new DbStateMachineConfig();
        config.setDataSource(dataSource);
        config.setResources(new String[]{"classpath:statemachine/*.json"});
        config.setEnableAsync(true);
        config.setThreadPoolExecutor(sagaThreadPool());

        ProcessCtrlStateMachineEngine engine = new ProcessCtrlStateMachineEngine();
        engine.setStateMachineConfig(config);
        return engine;
    }

    @Bean
    public ThreadPoolExecutor sagaThreadPool() {
        return new ThreadPoolExecutor(
            5, 20, 60, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(100),
            new ThreadFactoryBuilder().setNameFormat("saga-%d").build()
        );
    }
}
```

```json
// resources/statemachine/order-saga.json
{
  "Name": "orderSaga",
  "Comment": "订单创建 Saga 流程",
  "StartState": "CreateOrder",
  "Version": "1.0.0",
  "States": {
    "CreateOrder": {
      "Type": "ServiceTask",
      "ServiceName": "orderService",
      "ServiceMethod": "createOrder",
      "CompensateState": "CompensateCreateOrder",
      "Next": "DeductInventory",
      "Input": ["$.[orderId]", "$.[userId]", "$.[productId]", "$.[quantity]"],
      "Output": {
        "orderResult": "$.#root"
      }
    },
    "CompensateCreateOrder": {
      "Type": "ServiceTask",
      "ServiceName": "orderService",
      "ServiceMethod": "cancelOrder"
    },
    "DeductInventory": {
      "Type": "ServiceTask",
      "ServiceName": "inventoryService",
      "ServiceMethod": "deduct",
      "CompensateState": "CompensateDeductInventory",
      "Next": "DeductAccount",
      "Input": ["$.[productId]", "$.[quantity]"]
    },
    "CompensateDeductInventory": {
      "Type": "ServiceTask",
      "ServiceName": "inventoryService",
      "ServiceMethod": "compensate"
    },
    "DeductAccount": {
      "Type": "ServiceTask",
      "ServiceName": "accountService",
      "ServiceMethod": "deduct",
      "CompensateState": "CompensateDeductAccount",
      "Next": "Succeed",
      "Input": ["$.[userId]", "$.[amount]"]
    },
    "CompensateDeductAccount": {
      "Type": "ServiceTask",
      "ServiceName": "accountService",
      "ServiceMethod": "compensate"
    },
    "Succeed": {
      "Type": "Succeed"
    }
  }
}
```

### XID 传递

```java
/**
 * Feign 拦截器 - 传递 XID
 */
@Component
public class SeataFeignInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        String xid = RootContext.getXID();
        if (StringUtils.hasText(xid)) {
            template.header(RootContext.KEY_XID, xid);
        }
    }
}

/**
 * 服务端接收 XID
 */
@Component
public class SeataHandlerInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
            Object handler) {
        String xid = request.getHeader(RootContext.KEY_XID);
        if (StringUtils.hasText(xid)) {
            RootContext.bind(xid);
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
            Object handler, Exception ex) {
        RootContext.unbind();
    }
}
```

## 事务模式对比

| 特性 | AT | TCC | Saga | XA |
|------|-----|-----|------|-----|
| 侵入性 | 低 | 高 | 中 | 低 |
| 性能 | 高 | 高 | 高 | 低 |
| 一致性 | 最终一致 | 最终一致 | 最终一致 | 强一致 |
| 适用场景 | 通用 | 高并发 | 长事务 | 传统数据库 |
| 开发成本 | 低 | 高 | 中 | 低 |
| 回滚方式 | 自动 | 手动补偿 | 逆向补偿 | 自动 |

## 最佳实践清单

- [ ] 选择合适的事务模式
- [ ] 业务数据库创建 undo_log 表
- [ ] 配置事务超时时间
- [ ] TCC 实现幂等和空回滚处理
- [ ] 正确传递 XID
- [ ] 配置合理的重试次数
- [ ] 监控全局事务状态
- [ ] 事务日志定期清理
