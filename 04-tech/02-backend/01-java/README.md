# Java 核心开发最佳实践

## 角色设定

你是一位精通 Java 17+ 的后端开发专家，擅长面向对象设计、并发编程、JVM 调优和设计模式应用。

## 提示词模板

### 代码实现

```
请用 Java 实现以下功能：
- 功能描述：[描述功能]
- Java 版本：[8/11/17/21]
- 是否使用新特性：[Record/Sealed/Pattern Matching]
- 并发需求：[单线程/多线程/响应式]

要求：
1. 遵循 SOLID 原则
2. 添加 JavaDoc 注释
3. 包含单元测试
4. 考虑异常处理
```

### 性能优化

```
请帮我优化以下 Java 代码的性能：
[粘贴代码]

当前问题：[内存占用高/响应慢/GC 频繁]
JVM 版本：[版本]
请分析并提供优化方案。
```

## 核心代码示例

### Java 17+ 新特性

```java
// Record 类 - 不可变数据载体
public record User(Long id, String name, String email) {
    // 紧凑构造器
    public User {
        Objects.requireNonNull(name, "name cannot be null");
        Objects.requireNonNull(email, "email cannot be null");
    }

    // 可以添加静态方法
    public static User of(String name, String email) {
        return new User(null, name, email);
    }
}

// Sealed 类 - 限制继承
public sealed interface Shape permits Circle, Rectangle, Triangle {
    double area();
}

public record Circle(double radius) implements Shape {
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

public record Rectangle(double width, double height) implements Shape {
    @Override
    public double area() {
        return width * height;
    }
}

// Pattern Matching
public String describe(Object obj) {
    return switch (obj) {
        case Integer i -> "Integer: " + i;
        case String s -> "String: " + s;
        case List<?> list when list.isEmpty() -> "Empty list";
        case List<?> list -> "List with " + list.size() + " elements";
        case null -> "null value";
        default -> "Unknown type";
    };
}

// instanceof 模式匹配
public double calculateArea(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t -> 0.5 * t.base() * t.height();
    };
}

// Text Blocks
String json = """
    {
        "name": "%s",
        "email": "%s"
    }
    """.formatted(name, email);
```

### 并发编程

```java
// CompletableFuture 异步编程
public CompletableFuture<OrderResult> processOrder(Order order) {
    return CompletableFuture
        .supplyAsync(() -> validateOrder(order), executor)
        .thenCompose(validated ->
            CompletableFuture.allOf(
                checkInventory(validated),
                calculatePrice(validated),
                verifyPayment(validated)
            ).thenApply(v -> validated)
        )
        .thenApplyAsync(this::createOrder, executor)
        .exceptionally(ex -> {
            log.error("Order processing failed", ex);
            return OrderResult.failed(ex.getMessage());
        });
}

// 并行流处理
public List<UserDTO> processUsers(List<User> users) {
    return users.parallelStream()
        .filter(User::isActive)
        .map(this::enrichUser)
        .sorted(Comparator.comparing(UserDTO::getName))
        .collect(Collectors.toList());
}

// Virtual Threads (Java 21+)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> futures = urls.stream()
        .map(url -> executor.submit(() -> fetchUrl(url)))
        .toList();

    for (Future<String> future : futures) {
        String result = future.get();
        process(result);
    }
}

// 结构化并发 (Java 21+)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<User> userFuture = scope.fork(() -> fetchUser(userId));
    Future<List<Order>> ordersFuture = scope.fork(() -> fetchOrders(userId));

    scope.join();
    scope.throwIfFailed();

    return new UserWithOrders(userFuture.resultNow(), ordersFuture.resultNow());
}
```

### 函数式编程

```java
// Optional 正确使用
public Optional<User> findUser(Long id) {
    return Optional.ofNullable(userRepository.findById(id))
        .filter(User::isActive)
        .map(this::enrichUser);
}

// 避免 Optional 滥用
// ❌ 错误
public Optional<String> getName() {
    return Optional.ofNullable(this.name);
}

// ✅ 正确 - 只在可能缺失时使用
public String getName() {
    return this.name;
}

// Stream 链式操作
public Map<String, List<Order>> groupOrdersByStatus(List<Order> orders) {
    return orders.stream()
        .filter(order -> order.getAmount().compareTo(BigDecimal.ZERO) > 0)
        .collect(Collectors.groupingBy(
            order -> order.getStatus().name(),
            Collectors.toList()
        ));
}

// 自定义 Collector
public static <T> Collector<T, ?, List<List<T>>> partitionBySize(int size) {
    return Collector.of(
        ArrayList::new,
        (list, item) -> {
            if (list.isEmpty() || list.get(list.size() - 1).size() >= size) {
                list.add(new ArrayList<>());
            }
            list.get(list.size() - 1).add(item);
        },
        (left, right) -> {
            left.addAll(right);
            return left;
        }
    );
}
```

### 异常处理

```java
// 自定义业务异常
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;
    private final Object[] args;

    public BusinessException(ErrorCode errorCode, Object... args) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
        this.args = args;
    }

    public String getFormattedMessage() {
        return String.format(errorCode.getMessage(), args);
    }
}

// 错误码枚举
public enum ErrorCode {
    USER_NOT_FOUND("用户不存在: %s"),
    INSUFFICIENT_BALANCE("余额不足，当前余额: %s，需要: %s"),
    ORDER_EXPIRED("订单已过期: %s");

    private final String message;

    ErrorCode(String message) {
        this.message = message;
    }

    public String getMessage() {
        return message;
    }
}

// 统一异常处理
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiResponse<?>> handleBusinessException(BusinessException e) {
        return ResponseEntity
            .badRequest()
            .body(ApiResponse.error(e.getErrorCode(), e.getFormattedMessage()));
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ApiResponse<?>> handleValidation(ConstraintViolationException e) {
        String message = e.getConstraintViolations().stream()
            .map(ConstraintViolation::getMessage)
            .collect(Collectors.joining(", "));
        return ResponseEntity
            .badRequest()
            .body(ApiResponse.error(ErrorCode.VALIDATION_ERROR, message));
    }
}
```

### 设计模式

```java
// Builder 模式
@Builder
public class HttpRequest {
    private final String url;
    private final HttpMethod method;
    @Builder.Default
    private final Map<String, String> headers = new HashMap<>();
    private final String body;
    @Builder.Default
    private final Duration timeout = Duration.ofSeconds(30);
}

// 使用
HttpRequest request = HttpRequest.builder()
    .url("https://api.example.com/users")
    .method(HttpMethod.POST)
    .body(jsonBody)
    .timeout(Duration.ofSeconds(10))
    .build();

// 策略模式
public interface PaymentStrategy {
    PaymentResult pay(Order order);
}

@Component("alipay")
public class AlipayStrategy implements PaymentStrategy {
    @Override
    public PaymentResult pay(Order order) {
        // 支付宝支付逻辑
    }
}

@Component("wechat")
public class WechatPayStrategy implements PaymentStrategy {
    @Override
    public PaymentResult pay(Order order) {
        // 微信支付逻辑
    }
}

@Service
public class PaymentService {
    private final Map<String, PaymentStrategy> strategies;

    public PaymentService(Map<String, PaymentStrategy> strategies) {
        this.strategies = strategies;
    }

    public PaymentResult pay(String paymentType, Order order) {
        PaymentStrategy strategy = strategies.get(paymentType);
        if (strategy == null) {
            throw new BusinessException(ErrorCode.UNSUPPORTED_PAYMENT_TYPE, paymentType);
        }
        return strategy.pay(order);
    }
}
```

### JVM 调优参数

```bash
# JVM 基础参数
java -Xms4g -Xmx4g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/logs/heapdump.hprof \
  -Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=10M \
  -jar app.jar

# ZGC (低延迟)
java -Xms4g -Xmx4g \
  -XX:+UseZGC \
  -XX:+ZGenerational \
  -jar app.jar
```

## 最佳实践清单

- [ ] 使用 Java 17+ 新特性简化代码
- [ ] 优先使用不可变对象 (Record)
- [ ] 正确使用 Optional 避免 NPE
- [ ] 使用 CompletableFuture 处理异步
- [ ] 定义清晰的异常层次结构
- [ ] 遵循 SOLID 设计原则
- [ ] 编写单元测试覆盖核心逻辑
- [ ] 合理配置 JVM 参数
