# Spring Cloud Gateway 网关最佳实践

## 角色设定

你是一位精通 Spring Cloud Gateway 的微服务网关专家，擅长路由配置、过滤器开发、限流熔断和安全认证。

## 提示词模板

### Gateway 配置

```
请帮我配置 Spring Cloud Gateway：
- 服务发现：[Nacos/Eureka/Consul]
- 路由规则：[描述路由需求]
- 过滤器：[限流/认证/日志/...]
- 负载均衡：[策略]
- 熔断配置：[是否需要]

请提供配置和代码示例。
```

## 核心代码示例

### 依赖配置

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Gateway -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>

    <!-- 服务发现 - Nacos -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!-- 负载均衡 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>

    <!-- Sentinel 限流 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
    </dependency>

    <!-- Redis 限流 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    </dependency>

    <!-- JWT -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>
</dependencies>
```

### 基础配置

```yaml
# application.yml
spring:
  application:
    name: api-gateway
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_SERVER:localhost:8848}
        namespace: ${NACOS_NAMESPACE:dev}
    gateway:
      # 服务发现路由
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      # 默认过滤器
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin Access-Control-Allow-Credentials, RETAIN_UNIQUE
        - AddResponseHeader=X-Response-Time, %{response_time}ms
      # 全局 CORS
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOriginPatterns: "*"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowedHeaders: "*"
            allowCredentials: true
            maxAge: 3600
      # 路由配置
      routes:
        # 用户服务
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@ipKeyResolver}"

        # 订单服务
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST,PUT,DELETE
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: orderCircuitBreaker
                fallbackUri: forward:/fallback/order

        # 支付服务
        - id: payment-service
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          filters:
            - StripPrefix=1
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
                methods: GET,POST
                backoff:
                  firstBackoff: 100ms
                  maxBackoff: 500ms
                  factor: 2

        # WebSocket
        - id: websocket-service
          uri: lb:ws://websocket-service
          predicates:
            - Path=/ws/**
          filters:
            - StripPrefix=1

server:
  port: 8080

# 熔断配置
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10000
        permittedNumberOfCallsInHalfOpenState: 3
    instances:
      orderCircuitBreaker:
        baseConfig: default
  timelimiter:
    configs:
      default:
        timeoutDuration: 10s
```

### 全局过滤器

```java
/**
 * 请求日志过滤器
 */
@Component
@Slf4j
public class RequestLogFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();
        String requestId = UUID.randomUUID().toString();

        ServerHttpRequest request = exchange.getRequest();
        log.info("Request: id={}, method={}, path={}, ip={}",
            requestId,
            request.getMethod(),
            request.getPath(),
            getClientIp(request));

        // 添加请求ID到Header
        ServerHttpRequest mutatedRequest = request.mutate()
            .header("X-Request-ID", requestId)
            .build();

        return chain.filter(exchange.mutate().request(mutatedRequest).build())
            .doFinally(signalType -> {
                long duration = System.currentTimeMillis() - startTime;
                log.info("Response: id={}, status={}, duration={}ms",
                    requestId,
                    exchange.getResponse().getStatusCode(),
                    duration);
            });
    }

    private String getClientIp(ServerHttpRequest request) {
        String ip = request.getHeaders().getFirst("X-Forwarded-For");
        if (ip == null || ip.isEmpty()) {
            ip = request.getHeaders().getFirst("X-Real-IP");
        }
        if (ip == null || ip.isEmpty()) {
            ip = request.getRemoteAddress() != null ?
                request.getRemoteAddress().getAddress().getHostAddress() : "unknown";
        }
        return ip.split(",")[0].trim();
    }

    @Override
    public int getOrder() {
        return -100; // 最先执行
    }
}
```

### JWT 认证过滤器

```java
/**
 * JWT 认证过滤器
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter implements GlobalFilter, Ordered {

    private final JwtUtils jwtUtils;

    // 白名单路径
    private static final List<String> WHITE_LIST = List.of(
        "/api/auth/login",
        "/api/auth/register",
        "/api/auth/refresh",
        "/api/public/**",
        "/actuator/**"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getPath().value();

        // 白名单放行
        if (isWhiteListed(path)) {
            return chain.filter(exchange);
        }

        // 获取 Token
        String token = extractToken(request);
        if (token == null) {
            return unauthorized(exchange, "Missing token");
        }

        // 验证 Token
        try {
            Claims claims = jwtUtils.parseToken(token);

            // 检查是否过期
            if (claims.getExpiration().before(new Date())) {
                return unauthorized(exchange, "Token expired");
            }

            // 传递用户信息到下游服务
            ServerHttpRequest mutatedRequest = request.mutate()
                .header("X-User-ID", claims.getSubject())
                .header("X-User-Name", claims.get("username", String.class))
                .header("X-User-Roles", claims.get("roles", String.class))
                .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());
        } catch (Exception e) {
            log.error("Token validation failed", e);
            return unauthorized(exchange, "Invalid token");
        }
    }

    private boolean isWhiteListed(String path) {
        return WHITE_LIST.stream().anyMatch(pattern -> {
            if (pattern.endsWith("/**")) {
                return path.startsWith(pattern.replace("/**", ""));
            }
            return pattern.equals(path);
        });
    }

    private String extractToken(ServerHttpRequest request) {
        String bearerToken = request.getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);

        String body = String.format("{\"code\":401,\"message\":\"%s\"}", message);
        DataBuffer buffer = response.bufferFactory().wrap(body.getBytes(StandardCharsets.UTF_8));

        return response.writeWith(Mono.just(buffer));
    }

    @Override
    public int getOrder() {
        return -50; // 在日志过滤器之后
    }
}
```

### 限流配置

```java
@Configuration
public class RateLimiterConfig {

    /**
     * IP 限流 Key 解析器
     */
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> {
            String ip = exchange.getRequest().getHeaders().getFirst("X-Forwarded-For");
            if (ip == null) {
                ip = exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
            }
            return Mono.just(ip.split(",")[0].trim());
        };
    }

    /**
     * 用户限流 Key 解析器
     */
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-ID");
            return Mono.just(userId != null ? userId : "anonymous");
        };
    }

    /**
     * API Path 限流 Key 解析器
     */
    @Bean
    public KeyResolver pathKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest().getPath().value());
    }
}
```

### 自定义过滤器工厂

```java
/**
 * 请求体日志过滤器
 */
@Component
@Slf4j
public class RequestBodyLogGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RequestBodyLogGatewayFilterFactory.Config> {

    public RequestBodyLogGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            if (!config.isEnabled()) {
                return chain.filter(exchange);
            }

            return DataBufferUtils.join(exchange.getRequest().getBody())
                .flatMap(dataBuffer -> {
                    byte[] bytes = new byte[dataBuffer.readableByteCount()];
                    dataBuffer.read(bytes);
                    DataBufferUtils.release(dataBuffer);

                    String body = new String(bytes, StandardCharsets.UTF_8);
                    log.info("Request body: {}", body);

                    ServerHttpRequest mutatedRequest = new ServerHttpRequestDecorator(exchange.getRequest()) {
                        @Override
                        public Flux<DataBuffer> getBody() {
                            return Flux.just(exchange.getResponse().bufferFactory().wrap(bytes));
                        }
                    };

                    return chain.filter(exchange.mutate().request(mutatedRequest).build());
                })
                .switchIfEmpty(chain.filter(exchange));
        };
    }

    @Data
    public static class Config {
        private boolean enabled = true;
    }
}
```

### 响应修改过滤器

```java
/**
 * 响应包装过滤器
 */
@Component
@Slf4j
public class ResponseWrapperFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpResponse originalResponse = exchange.getResponse();

        DataBufferFactory bufferFactory = originalResponse.bufferFactory();
        ServerHttpResponseDecorator decoratedResponse = new ServerHttpResponseDecorator(originalResponse) {
            @Override
            public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                if (body instanceof Flux) {
                    Flux<? extends DataBuffer> fluxBody = (Flux<? extends DataBuffer>) body;
                    return super.writeWith(fluxBody.buffer().map(dataBuffers -> {
                        // 合并响应体
                        DataBuffer join = bufferFactory.join(dataBuffers);
                        byte[] content = new byte[join.readableByteCount()];
                        join.read(content);
                        DataBufferUtils.release(join);

                        // 包装响应
                        String responseBody = new String(content, StandardCharsets.UTF_8);
                        String wrappedBody = wrapResponse(responseBody, originalResponse.getStatusCode());

                        byte[] newContent = wrappedBody.getBytes(StandardCharsets.UTF_8);
                        originalResponse.getHeaders().setContentLength(newContent.length);
                        return bufferFactory.wrap(newContent);
                    }));
                }
                return super.writeWith(body);
            }
        };

        return chain.filter(exchange.mutate().response(decoratedResponse).build());
    }

    private String wrapResponse(String body, HttpStatusCode status) {
        // 只处理 JSON 响应
        if (body.startsWith("{") || body.startsWith("[")) {
            return String.format("{\"code\":%d,\"data\":%s,\"timestamp\":%d}",
                status.value(), body, System.currentTimeMillis());
        }
        return body;
    }

    @Override
    public int getOrder() {
        return -10;
    }
}
```

### 降级处理

```java
@RestController
@RequestMapping("/fallback")
@Slf4j
public class FallbackController {

    @GetMapping("/order")
    public Mono<ResponseEntity<Map<String, Object>>> orderFallback() {
        log.warn("Order service fallback triggered");
        return Mono.just(ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "code", 503,
                "message", "Order service is temporarily unavailable",
                "timestamp", System.currentTimeMillis()
            )));
    }

    @GetMapping("/user")
    public Mono<ResponseEntity<Map<String, Object>>> userFallback() {
        log.warn("User service fallback triggered");
        return Mono.just(ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "code", 503,
                "message", "User service is temporarily unavailable",
                "timestamp", System.currentTimeMillis()
            )));
    }

    @GetMapping("/default")
    public Mono<ResponseEntity<Map<String, Object>>> defaultFallback() {
        return Mono.just(ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "code", 503,
                "message", "Service temporarily unavailable",
                "timestamp", System.currentTimeMillis()
            )));
    }
}
```

### 动态路由

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class DynamicRouteService implements ApplicationEventPublisherAware {

    private final RouteDefinitionWriter routeDefinitionWriter;
    private ApplicationEventPublisher publisher;

    /**
     * 添加路由
     */
    public void addRoute(RouteDefinition route) {
        routeDefinitionWriter.save(Mono.just(route)).subscribe();
        publisher.publishEvent(new RefreshRoutesEvent(this));
        log.info("Route added: {}", route.getId());
    }

    /**
     * 删除路由
     */
    public void deleteRoute(String routeId) {
        routeDefinitionWriter.delete(Mono.just(routeId)).subscribe();
        publisher.publishEvent(new RefreshRoutesEvent(this));
        log.info("Route deleted: {}", routeId);
    }

    /**
     * 从 Nacos 加载路由
     */
    public void loadRoutesFromNacos(String config) {
        try {
            List<RouteDefinition> routes = JSON.parseArray(config, RouteDefinition.class);
            routes.forEach(this::addRoute);
            log.info("Loaded {} routes from Nacos", routes.size());
        } catch (Exception e) {
            log.error("Failed to load routes from Nacos", e);
        }
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }
}
```

### 全局异常处理

```java
@Configuration
@Slf4j
public class GlobalExceptionConfig {

    @Bean
    @Order(-1)
    public ErrorWebExceptionHandler errorWebExceptionHandler() {
        return (exchange, ex) -> {
            ServerHttpResponse response = exchange.getResponse();
            response.getHeaders().setContentType(MediaType.APPLICATION_JSON);

            int code;
            String message;

            if (ex instanceof ResponseStatusException rse) {
                response.setStatusCode(rse.getStatusCode());
                code = rse.getStatusCode().value();
                message = rse.getReason();
            } else if (ex instanceof CircuitBreakerOpenException) {
                response.setStatusCode(HttpStatus.SERVICE_UNAVAILABLE);
                code = 503;
                message = "Service circuit breaker is open";
            } else {
                response.setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
                code = 500;
                message = "Internal server error";
                log.error("Gateway error", ex);
            }

            String body = String.format("{\"code\":%d,\"message\":\"%s\",\"timestamp\":%d}",
                code, message, System.currentTimeMillis());

            DataBuffer buffer = response.bufferFactory()
                .wrap(body.getBytes(StandardCharsets.UTF_8));

            return response.writeWith(Mono.just(buffer));
        };
    }
}
```

## 过滤器执行顺序

| Order 值 | 过滤器 | 说明 |
|---------|--------|------|
| -100 | RequestLogFilter | 请求日志 |
| -50 | JwtAuthenticationFilter | JWT 认证 |
| -10 | ResponseWrapperFilter | 响应包装 |
| 0 | 默认过滤器 | Spring Cloud Gateway 内置 |

## 最佳实践清单

- [ ] 配置全局 CORS
- [ ] 实现 JWT 认证过滤器
- [ ] 配置限流规则
- [ ] 实现熔断降级
- [ ] 添加请求日志
- [ ] 配置路由动态刷新
- [ ] 设置合理的超时时间
- [ ] 实现全局异常处理
