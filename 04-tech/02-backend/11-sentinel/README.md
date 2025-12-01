# Sentinel 熔断限流最佳实践

## 角色设定

你是一位精通 Alibaba Sentinel 的微服务稳定性专家，擅长流量控制、熔断降级和系统保护。

## 提示词模板

### Sentinel 规则配置

```
请帮我配置 Sentinel 规则：
- 保护场景：[API限流/服务熔断/热点防护/系统保护]
- 流量模式：[直接/关联/链路]
- 阈值类型：[QPS/线程数]
- 降级策略：[慢调用/异常比例/异常数]
- 持久化方式：[Nacos/Apollo/文件]

请提供规则配置和代码示例。
```

## 核心代码示例

### 依赖配置

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Sentinel 核心 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>

    <!-- Sentinel 数据源 - Nacos -->
    <dependency>
        <groupId>com.alibaba.csp</groupId>
        <artifactId>sentinel-datasource-nacos</artifactId>
    </dependency>

    <!-- Sentinel 网关适配 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
    </dependency>
</dependencies>
```

### 基础配置

```yaml
# application.yml
spring:
  cloud:
    sentinel:
      enabled: true
      eager: true  # 立即初始化
      transport:
        dashboard: ${SENTINEL_DASHBOARD:localhost:8080}
        port: 8719  # 与 Dashboard 通信的端口
      # 规则持久化 - Nacos
      datasource:
        # 流控规则
        flow:
          nacos:
            server-addr: ${NACOS_SERVER:localhost:8848}
            namespace: ${NACOS_NAMESPACE:dev}
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: flow
        # 熔断规则
        degrade:
          nacos:
            server-addr: ${NACOS_SERVER:localhost:8848}
            namespace: ${NACOS_NAMESPACE:dev}
            data-id: ${spring.application.name}-degrade-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: degrade
        # 系统规则
        system:
          nacos:
            server-addr: ${NACOS_SERVER:localhost:8848}
            namespace: ${NACOS_NAMESPACE:dev}
            data-id: ${spring.application.name}-system-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: system
```

### 流控规则定义

```java
@Configuration
public class SentinelConfig {

    @PostConstruct
    public void initFlowRules() {
        List<FlowRule> rules = new ArrayList<>();

        // QPS 限流
        FlowRule qpsRule = new FlowRule();
        qpsRule.setResource("orderCreate");
        qpsRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        qpsRule.setCount(100);  // QPS 阈值
        qpsRule.setLimitApp("default");
        qpsRule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP);
        qpsRule.setWarmUpPeriodSec(10);  // 预热时间
        rules.add(qpsRule);

        // 线程数限流
        FlowRule threadRule = new FlowRule();
        threadRule.setResource("orderQuery");
        threadRule.setGrade(RuleConstant.FLOW_GRADE_THREAD);
        threadRule.setCount(50);  // 并发线程数
        rules.add(threadRule);

        // 关联限流
        FlowRule relateRule = new FlowRule();
        relateRule.setResource("orderCreate");
        relateRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        relateRule.setCount(100);
        relateRule.setStrategy(RuleConstant.STRATEGY_RELATE);
        relateRule.setRefResource("orderQuery");  // 关联资源
        rules.add(relateRule);

        // 匀速排队
        FlowRule rateLimiterRule = new FlowRule();
        rateLimiterRule.setResource("orderPay");
        rateLimiterRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rateLimiterRule.setCount(10);
        rateLimiterRule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER);
        rateLimiterRule.setMaxQueueingTimeMs(5000);  // 最大排队时间
        rules.add(rateLimiterRule);

        FlowRuleManager.loadRules(rules);
    }
}
```

### 熔断规则定义

```java
@Configuration
public class DegradeRuleConfig {

    @PostConstruct
    public void initDegradeRules() {
        List<DegradeRule> rules = new ArrayList<>();

        // 慢调用比例熔断
        DegradeRule slowCallRule = new DegradeRule();
        slowCallRule.setResource("userService");
        slowCallRule.setGrade(CircuitBreakerStrategy.SLOW_REQUEST_RATIO.getType());
        slowCallRule.setCount(0.5);  // 慢调用比例阈值
        slowCallRule.setSlowRatioThreshold(0.5);
        slowCallRule.setTimeWindow(30);  // 熔断时长(秒)
        slowCallRule.setMinRequestAmount(10);  // 最小请求数
        slowCallRule.setStatIntervalMs(10000);  // 统计时长
        rules.add(slowCallRule);

        // 异常比例熔断
        DegradeRule errorRatioRule = new DegradeRule();
        errorRatioRule.setResource("paymentService");
        errorRatioRule.setGrade(CircuitBreakerStrategy.ERROR_RATIO.getType());
        errorRatioRule.setCount(0.3);  // 异常比例阈值
        errorRatioRule.setTimeWindow(60);
        errorRatioRule.setMinRequestAmount(10);
        rules.add(errorRatioRule);

        // 异常数熔断
        DegradeRule errorCountRule = new DegradeRule();
        errorCountRule.setResource("inventoryService");
        errorCountRule.setGrade(CircuitBreakerStrategy.ERROR_COUNT.getType());
        errorCountRule.setCount(10);  // 异常数阈值
        errorCountRule.setTimeWindow(60);
        rules.add(errorCountRule);

        DegradeRuleManager.loadRules(rules);
    }
}
```

### 注解方式使用

```java
@Service
@Slf4j
public class OrderService {

    /**
     * 使用注解定义资源
     */
    @SentinelResource(
        value = "createOrder",
        blockHandler = "createOrderBlockHandler",
        fallback = "createOrderFallback",
        exceptionsToIgnore = {IllegalArgumentException.class}
    )
    public Order createOrder(OrderRequest request) {
        // 业务逻辑
        return orderRepository.save(buildOrder(request));
    }

    /**
     * 限流/熔断处理 - 必须是 public，参数相同，最后加 BlockException
     */
    public Order createOrderBlockHandler(OrderRequest request, BlockException ex) {
        log.warn("Order creation blocked: {}", ex.getClass().getSimpleName());
        if (ex instanceof FlowException) {
            throw new BusinessException(ErrorCode.RATE_LIMIT_EXCEEDED);
        } else if (ex instanceof DegradeException) {
            throw new BusinessException(ErrorCode.SERVICE_DEGRADED);
        }
        throw new BusinessException(ErrorCode.SYSTEM_BUSY);
    }

    /**
     * 业务异常降级处理
     */
    public Order createOrderFallback(OrderRequest request, Throwable ex) {
        log.error("Order creation failed, fallback triggered", ex);
        // 返回降级结果或抛出自定义异常
        throw new BusinessException(ErrorCode.SERVICE_UNAVAILABLE);
    }
}
```

### 全局异常处理

```java
@RestControllerAdvice
@Slf4j
public class SentinelExceptionHandler {

    @ExceptionHandler(FlowException.class)
    public ResponseEntity<ApiResponse<?>> handleFlowException(FlowException e) {
        log.warn("Flow control triggered: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
            .body(ApiResponse.error(ErrorCode.RATE_LIMIT_EXCEEDED));
    }

    @ExceptionHandler(DegradeException.class)
    public ResponseEntity<ApiResponse<?>> handleDegradeException(DegradeException e) {
        log.warn("Circuit breaker triggered: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(ApiResponse.error(ErrorCode.SERVICE_DEGRADED));
    }

    @ExceptionHandler(ParamFlowException.class)
    public ResponseEntity<ApiResponse<?>> handleParamFlowException(ParamFlowException e) {
        log.warn("Hotspot param flow control: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
            .body(ApiResponse.error(ErrorCode.HOTSPOT_PARAM_LIMIT));
    }

    @ExceptionHandler(SystemBlockException.class)
    public ResponseEntity<ApiResponse<?>> handleSystemBlockException(SystemBlockException e) {
        log.warn("System protection triggered: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(ApiResponse.error(ErrorCode.SYSTEM_PROTECTION));
    }
}
```

### 热点参数限流

```java
@Service
public class ProductService {

    @SentinelResource(
        value = "getProduct",
        blockHandler = "getProductBlockHandler"
    )
    public Product getProduct(@SentinelParam Long productId) {
        return productRepository.findById(productId)
            .orElseThrow(() -> new NotFoundException("Product not found"));
    }

    public Product getProductBlockHandler(Long productId, BlockException ex) {
        log.warn("Product {} query blocked", productId);
        throw new BusinessException(ErrorCode.RATE_LIMIT_EXCEEDED);
    }
}

// 热点规则配置
@Configuration
public class HotspotRuleConfig {

    @PostConstruct
    public void initParamFlowRules() {
        ParamFlowRule rule = new ParamFlowRule();
        rule.setResource("getProduct");
        rule.setParamIdx(0);  // 第一个参数
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setCount(100);

        // 针对特定参数值的限流
        ParamFlowItem item = new ParamFlowItem();
        item.setObject("1001");  // 热门商品ID
        item.setClassType(Long.class.getName());
        item.setCount(10);  // 更严格的限流
        rule.setParamFlowItemList(Collections.singletonList(item));

        ParamFlowRuleManager.loadRules(Collections.singletonList(rule));
    }
}
```

### Feign 集成

```java
// Feign 客户端
@FeignClient(
    name = "user-service",
    fallbackFactory = UserClientFallbackFactory.class
)
public interface UserClient {

    @GetMapping("/users/{id}")
    User getUser(@PathVariable("id") Long id);

    @PostMapping("/users")
    User createUser(@RequestBody UserRequest request);
}

// Fallback Factory
@Component
@Slf4j
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        log.error("UserClient fallback triggered", cause);

        return new UserClient() {
            @Override
            public User getUser(Long id) {
                // 返回默认用户或缓存数据
                return User.defaultUser();
            }

            @Override
            public User createUser(UserRequest request) {
                throw new BusinessException(ErrorCode.SERVICE_UNAVAILABLE);
            }
        };
    }
}
```

### 系统保护规则

```java
@Configuration
public class SystemRuleConfig {

    @PostConstruct
    public void initSystemRules() {
        List<SystemRule> rules = new ArrayList<>();

        // CPU 使用率保护
        SystemRule cpuRule = new SystemRule();
        cpuRule.setHighestCpuUsage(0.8);  // CPU 使用率超过 80% 触发
        rules.add(cpuRule);

        // 系统 Load 保护
        SystemRule loadRule = new SystemRule();
        loadRule.setHighestSystemLoad(10);  // 系统 load 超过 10 触发
        rules.add(loadRule);

        // 入口 QPS 保护
        SystemRule qpsRule = new SystemRule();
        qpsRule.setQps(1000);  // 入口 QPS 限制
        rules.add(qpsRule);

        // 并发线程数保护
        SystemRule threadRule = new SystemRule();
        threadRule.setMaxThread(500);  // 最大并发线程数
        rules.add(threadRule);

        // RT 保护
        SystemRule rtRule = new SystemRule();
        rtRule.setAvgRt(100);  // 平均 RT 超过 100ms 触发
        rules.add(rtRule);

        SystemRuleManager.loadRules(rules);
    }
}
```

### 规则持久化 JSON 示例

```json
// flow-rules.json
[
  {
    "resource": "orderCreate",
    "limitApp": "default",
    "grade": 1,
    "count": 100,
    "strategy": 0,
    "controlBehavior": 1,
    "warmUpPeriodSec": 10
  },
  {
    "resource": "orderQuery",
    "limitApp": "default",
    "grade": 0,
    "count": 50
  }
]

// degrade-rules.json
[
  {
    "resource": "userService",
    "grade": 0,
    "count": 500,
    "slowRatioThreshold": 0.5,
    "timeWindow": 30,
    "minRequestAmount": 10,
    "statIntervalMs": 10000
  }
]
```

### 自定义 Slot Chain

```java
@Component
public class CustomSlotChainBuilder implements SlotChainBuilder {

    @Override
    public ProcessorSlotChain build() {
        ProcessorSlotChain chain = new DefaultProcessorSlotChain();

        // 添加自定义 Slot
        chain.addLast(new CustomLogSlot());

        // 默认 Slot
        chain.addLast(new NodeSelectorSlot());
        chain.addLast(new ClusterBuilderSlot());
        chain.addLast(new StatisticSlot());
        chain.addLast(new FlowSlot());
        chain.addLast(new DegradeSlot());

        return chain;
    }
}

// 自定义日志 Slot
public class CustomLogSlot extends AbstractLinkedProcessorSlot<Object> {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper,
            Object param, int count, boolean prioritized, Object... args) throws Throwable {
        // 记录入口日志
        fireEntry(context, resourceWrapper, param, count, prioritized, args);
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper,
            int count, Object... args) {
        // 记录出口日志
        fireExit(context, resourceWrapper, count, args);
    }
}
```

## Sentinel vs Hystrix

| 特性 | Sentinel | Hystrix |
|------|----------|---------|
| 隔离策略 | 信号量 | 线程池/信号量 |
| 熔断策略 | 慢调用/异常比例/异常数 | 异常比例 |
| 实时统计 | 滑动窗口 | 滑动窗口 |
| 规则配置 | 动态规则 | 静态配置 |
| 控制台 | 完善的 Dashboard | 简单 Dashboard |
| 流量整形 | 支持 | 不支持 |
| 系统保护 | 支持 | 不支持 |
| 维护状态 | 活跃 | 停止维护 |

## 最佳实践清单

- [ ] 核心接口配置限流规则
- [ ] 依赖服务配置熔断规则
- [ ] 热点数据配置参数限流
- [ ] 启用系统保护规则
- [ ] 规则持久化到 Nacos
- [ ] 配置全局异常处理
- [ ] Feign 集成 Fallback
- [ ] 监控 Dashboard 部署
