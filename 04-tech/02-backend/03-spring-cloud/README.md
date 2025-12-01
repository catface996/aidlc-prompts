# Spring Cloud 微服务最佳实践

## 角色设定

你是一位精通 Spring Cloud 的微服务架构专家，擅长服务注册发现、配置中心、服务网关、熔断限流和分布式事务。

## 提示词模板

### 微服务架构设计

```
请帮我设计微服务架构：
- 业务场景：[描述业务]
- 服务拆分：[列出服务]
- 技术选型：
  - 注册中心：[Nacos/Eureka/Consul]
  - 配置中心：[Nacos/Apollo/Config]
  - 网关：[Gateway/Zuul]
  - 熔断：[Sentinel/Hystrix]
  - 链路追踪：[SkyWalking/Zipkin]
```

### 服务通信

```
请帮我实现微服务间通信：
- 通信方式：[HTTP/gRPC/消息队列]
- 调用模式：[同步/异步]
- 容错策略：[重试/熔断/降级]
```

## 核心配置示例

### 项目依赖 (pom.xml)

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2023.0.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Nacos 服务发现 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!-- Nacos 配置中心 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>

    <!-- OpenFeign -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>

    <!-- LoadBalancer -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>

    <!-- Sentinel -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>

    <!-- Seata 分布式事务 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
    </dependency>
</dependencies>
```

### Nacos 配置

```yaml
# bootstrap.yml
spring:
  application:
    name: order-service
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_SERVER:localhost:8848}
        namespace: ${NACOS_NAMESPACE:dev}
        group: DEFAULT_GROUP
      config:
        server-addr: ${NACOS_SERVER:localhost:8848}
        namespace: ${NACOS_NAMESPACE:dev}
        file-extension: yaml
        shared-configs:
          - data-id: common.yaml
            group: DEFAULT_GROUP
            refresh: true
```

### OpenFeign 服务调用

```java
// Feign 客户端定义
@FeignClient(
    name = "user-service",
    fallbackFactory = UserClientFallbackFactory.class,
    configuration = FeignConfig.class
)
public interface UserClient {

    @GetMapping("/api/v1/users/{id}")
    Result<UserDTO> getById(@PathVariable("id") Long id);

    @GetMapping("/api/v1/users")
    Result<List<UserDTO>> getByIds(@RequestParam("ids") List<Long> ids);

    @PostMapping("/api/v1/users/batch")
    Result<Map<Long, UserDTO>> getBatch(@RequestBody List<Long> ids);
}

// Fallback 工厂
@Component
@Slf4j
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        log.error("UserClient fallback, cause: {}", cause.getMessage());

        return new UserClient() {
            @Override
            public Result<UserDTO> getById(Long id) {
                return Result.error(ErrorCode.SERVICE_UNAVAILABLE, "用户服务暂不可用");
            }

            @Override
            public Result<List<UserDTO>> getByIds(List<Long> ids) {
                return Result.error(ErrorCode.SERVICE_UNAVAILABLE, "用户服务暂不可用");
            }

            @Override
            public Result<Map<Long, UserDTO>> getBatch(List<Long> ids) {
                return Result.success(Collections.emptyMap());
            }
        };
    }
}

// Feign 配置
@Configuration
public class FeignConfig {

    @Bean
    public RequestInterceptor requestInterceptor() {
        return template -> {
            // 传递请求头
            ServletRequestAttributes attributes = (ServletRequestAttributes)
                RequestContextHolder.getRequestAttributes();
            if (attributes != null) {
                HttpServletRequest request = attributes.getRequest();
                template.header("Authorization", request.getHeader("Authorization"));
                template.header("X-Request-Id", request.getHeader("X-Request-Id"));
            }
        };
    }

    @Bean
    public Retryer retryer() {
        return new Retryer.Default(100, 1000, 3);
    }
}
```

### Spring Cloud Gateway

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=2
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
          filters:
            - StripPrefix=2

      default-filters:
        - name: Retry
          args:
            retries: 3
            statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
            methods: GET
            backoff:
              firstBackoff: 100ms
              maxBackoff: 500ms
              factor: 2

      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods: "*"
            allowedHeaders: "*"
```

```java
// 自定义全局过滤器
@Component
@Slf4j
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getPath().value();

        // 白名单路径
        if (isWhitelist(path)) {
            return chain.filter(exchange);
        }

        // 验证 Token
        String token = request.getHeaders().getFirst("Authorization");
        if (StringUtils.isEmpty(token)) {
            return unauthorized(exchange, "Missing token");
        }

        try {
            Claims claims = JwtUtil.parseToken(token);
            // 将用户信息传递给下游服务
            ServerHttpRequest newRequest = request.mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Name", claims.get("username", String.class))
                .build();
            return chain.filter(exchange.mutate().request(newRequest).build());
        } catch (Exception e) {
            return unauthorized(exchange, "Invalid token");
        }
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);

        String body = JSON.toJSONString(Result.error(401, message));
        DataBuffer buffer = response.bufferFactory().wrap(body.getBytes(StandardCharsets.UTF_8));
        return response.writeWith(Mono.just(buffer));
    }

    @Override
    public int getOrder() {
        return -100;
    }
}
```

### Sentinel 熔断限流

```java
// 资源定义
@Service
public class OrderService {

    @SentinelResource(
        value = "createOrder",
        blockHandler = "createOrderBlockHandler",
        fallback = "createOrderFallback"
    )
    public Order createOrder(CreateOrderRequest request) {
        // 创建订单逻辑
        return order;
    }

    // 限流/熔断时的处理
    public Order createOrderBlockHandler(CreateOrderRequest request, BlockException ex) {
        throw new BusinessException(ErrorCode.SERVICE_BUSY, "服务繁忙，请稍后重试");
    }

    // 异常时的降级处理
    public Order createOrderFallback(CreateOrderRequest request, Throwable ex) {
        log.error("Create order failed", ex);
        throw new BusinessException(ErrorCode.SERVICE_ERROR, "订单创建失败");
    }
}

// Sentinel 规则配置
@Configuration
public class SentinelConfig {

    @PostConstruct
    public void init() {
        // 流控规则
        FlowRule flowRule = new FlowRule();
        flowRule.setResource("createOrder");
        flowRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        flowRule.setCount(100);
        flowRule.setLimitApp("default");
        FlowRuleManager.loadRules(Collections.singletonList(flowRule));

        // 熔断规则
        DegradeRule degradeRule = new DegradeRule();
        degradeRule.setResource("createOrder");
        degradeRule.setGrade(CircuitBreakerStrategy.SLOW_REQUEST_RATIO.getType());
        degradeRule.setCount(0.5); // 慢调用比例阈值
        degradeRule.setTimeWindow(10); // 熔断时长
        degradeRule.setMinRequestAmount(10); // 最小请求数
        degradeRule.setSlowRatioThreshold(0.5);
        DegradeRuleManager.loadRules(Collections.singletonList(degradeRule));
    }
}
```

### Seata 分布式事务

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;
    private final AccountClient accountClient;

    @GlobalTransactional(name = "create-order", rollbackFor = Exception.class)
    public Order createOrder(CreateOrderRequest request) {
        // 1. 创建订单
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setProductId(request.getProductId());
        order.setQuantity(request.getQuantity());
        order.setStatus(OrderStatus.PENDING);
        order = orderRepository.save(order);

        // 2. 扣减库存
        Result<Void> inventoryResult = inventoryClient.deduct(
            request.getProductId(), request.getQuantity());
        if (!inventoryResult.isSuccess()) {
            throw new BusinessException(ErrorCode.INVENTORY_INSUFFICIENT);
        }

        // 3. 扣减账户余额
        Result<Void> accountResult = accountClient.deduct(
            request.getUserId(), order.getTotalAmount());
        if (!accountResult.isSuccess()) {
            throw new BusinessException(ErrorCode.BALANCE_INSUFFICIENT);
        }

        // 4. 更新订单状态
        order.setStatus(OrderStatus.PAID);
        return orderRepository.save(order);
    }
}
```

```yaml
# Seata 配置
seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: default_tx_group
  registry:
    type: nacos
    nacos:
      server-addr: ${NACOS_SERVER:localhost:8848}
      namespace: ${NACOS_NAMESPACE:dev}
  config:
    type: nacos
    nacos:
      server-addr: ${NACOS_SERVER:localhost:8848}
      namespace: ${NACOS_NAMESPACE:dev}
```

## 微服务架构图

```
                    ┌─────────────────┐
                    │   API Gateway   │
                    │  (Spring Cloud  │
                    │    Gateway)     │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  User Service │   │ Order Service │   │Product Service│
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
        ┌───────────────┐     ┌───────────────┐
        │     Nacos     │     │    Sentinel   │
        │ (注册/配置)   │     │  (熔断限流)   │
        └───────────────┘     └───────────────┘
```

## 最佳实践清单

- [ ] 使用 Nacos 作为注册和配置中心
- [ ] 使用 OpenFeign + LoadBalancer 服务调用
- [ ] 使用 Sentinel 实现熔断限流
- [ ] 使用 Seata 处理分布式事务
- [ ] 配置统一的网关鉴权
- [ ] 实现服务间的链路追踪
- [ ] 配置合理的超时和重试策略
- [ ] 设计降级和容错方案
