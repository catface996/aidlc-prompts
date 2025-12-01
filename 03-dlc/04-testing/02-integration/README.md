# 集成测试最佳实践

## 角色设定

你是一位精通集成测试的软件质量专家，擅长 Spring Boot Test、Testcontainers、数据库集成和服务间测试。

## 提示词模板

### 集成测试编写

```
请帮我编写集成测试：
- 测试场景：[描述集成场景]
- 涉及组件：[数据库/缓存/消息队列/外部服务]
- 测试框架：[Spring Boot Test/Testcontainers]
- 数据准备：[描述测试数据需求]

请提供完整的测试代码。
```

## 核心代码示例

### Spring Boot 集成测试

```java
// OrderIntegrationTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@Transactional
@ActiveProfiles("test")
class OrderIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private ProductRepository productRepository;

    private User testUser;
    private Product testProduct;

    @BeforeEach
    void setUp() {
        testUser = userRepository.save(User.builder()
            .username("testuser")
            .email("test@example.com")
            .build());

        testProduct = productRepository.save(Product.builder()
            .name("Test Product")
            .price(new BigDecimal("99.99"))
            .stock(100)
            .build());
    }

    @Test
    @DisplayName("完整订单创建流程")
    void shouldCreateOrderSuccessfully() throws Exception {
        // Given
        OrderRequest request = OrderRequest.builder()
            .userId(testUser.getId())
            .productId(testProduct.getId())
            .quantity(2)
            .build();

        // When & Then
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists())
            .andExpect(jsonPath("$.userId").value(testUser.getId()))
            .andExpect(jsonPath("$.status").value("CREATED"))
            .andExpect(jsonPath("$.totalAmount").value(199.98));

        // 验证数据库状态
        List<Order> orders = orderRepository.findByUserId(testUser.getId());
        assertThat(orders).hasSize(1);

        // 验证库存扣减
        Product updatedProduct = productRepository.findById(testProduct.getId()).orElseThrow();
        assertThat(updatedProduct.getStock()).isEqualTo(98);
    }

    @Test
    @DisplayName("创建订单 - 库存不足")
    void shouldReturnErrorWhenInsufficientStock() throws Exception {
        OrderRequest request = OrderRequest.builder()
            .userId(testUser.getId())
            .productId(testProduct.getId())
            .quantity(1000)
            .build();

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.code").value("INSUFFICIENT_STOCK"));

        // 验证订单未创建
        assertThat(orderRepository.findByUserId(testUser.getId())).isEmpty();
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    @DisplayName("管理员查询所有订单")
    void shouldReturnAllOrdersForAdmin() throws Exception {
        // 准备数据
        orderRepository.save(Order.builder()
            .userId(testUser.getId())
            .productId(testProduct.getId())
            .quantity(1)
            .status(OrderStatus.CREATED)
            .build());

        mockMvc.perform(get("/api/admin/orders"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.content").isArray())
            .andExpect(jsonPath("$.content.length()").value(1));
    }
}
```

### Testcontainers 集成

```java
// DatabaseIntegrationTest.java
@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
class DatabaseIntegrationTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Test
    @DisplayName("数据库 CRUD 操作")
    void shouldPerformDatabaseOperations() {
        // Create
        User user = userRepository.save(User.builder()
            .username("testuser")
            .email("test@example.com")
            .build());
        assertThat(user.getId()).isNotNull();

        // Read
        User found = userRepository.findById(user.getId()).orElseThrow();
        assertThat(found.getUsername()).isEqualTo("testuser");

        // Update
        found.setEmail("updated@example.com");
        userRepository.save(found);
        User updated = userRepository.findById(user.getId()).orElseThrow();
        assertThat(updated.getEmail()).isEqualTo("updated@example.com");

        // Delete
        userRepository.delete(updated);
        assertThat(userRepository.findById(user.getId())).isEmpty();
    }

    @Test
    @DisplayName("Redis 缓存操作")
    void shouldCacheDataInRedis() {
        String key = "user:1";
        User user = User.builder()
            .id(1L)
            .username("cached_user")
            .build();

        redisTemplate.opsForValue().set(key, user, Duration.ofMinutes(5));

        User cached = (User) redisTemplate.opsForValue().get(key);
        assertThat(cached).isNotNull();
        assertThat(cached.getUsername()).isEqualTo("cached_user");
    }
}
```

### Kafka 集成测试

```java
// KafkaIntegrationTest.java
@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    brokerProperties = {"listeners=PLAINTEXT://localhost:9092", "port=9092"}
)
class KafkaIntegrationTest {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Autowired
    private OrderEventConsumer orderEventConsumer;

    @Test
    @DisplayName("发送和接收订单事件")
    void shouldSendAndReceiveOrderEvent() throws Exception {
        // Given
        OrderEvent event = OrderEvent.builder()
            .orderId(1L)
            .userId(100L)
            .eventType("ORDER_CREATED")
            .timestamp(Instant.now())
            .build();

        // When
        kafkaTemplate.send("order-events", event.getOrderId().toString(), event).get();

        // Then
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                assertThat(orderEventConsumer.getReceivedEvents())
                    .anyMatch(e -> e.getOrderId().equals(1L));
            });
    }
}
```

### REST 客户端集成测试

```java
// ExternalServiceIntegrationTest.java
@SpringBootTest
@AutoConfigureWireMock(port = 0)
class ExternalServiceIntegrationTest {

    @Autowired
    private PaymentClient paymentClient;

    @Value("${wiremock.server.port}")
    private int wireMockPort;

    @BeforeEach
    void setUp() {
        // 配置 WireMock 基础 URL
    }

    @Test
    @DisplayName("调用支付服务成功")
    void shouldCallPaymentServiceSuccessfully() {
        // 配置 Mock 响应
        stubFor(post(urlEqualTo("/api/payments"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "paymentId": "PAY123",
                        "status": "SUCCESS",
                        "amount": 100.00
                    }
                    """)));

        // 执行调用
        PaymentResponse response = paymentClient.createPayment(
            PaymentRequest.builder()
                .orderId(1L)
                .amount(new BigDecimal("100.00"))
                .build()
        );

        // 验证
        assertThat(response.getPaymentId()).isEqualTo("PAY123");
        assertThat(response.getStatus()).isEqualTo("SUCCESS");

        // 验证请求
        verify(postRequestedFor(urlEqualTo("/api/payments"))
            .withRequestBody(matchingJsonPath("$.orderId", equalTo("1"))));
    }

    @Test
    @DisplayName("支付服务超时处理")
    void shouldHandlePaymentServiceTimeout() {
        stubFor(post(urlEqualTo("/api/payments"))
            .willReturn(aResponse()
                .withStatus(200)
                .withFixedDelay(5000)));  // 5秒延迟

        assertThatThrownBy(() -> paymentClient.createPayment(
                PaymentRequest.builder()
                    .orderId(1L)
                    .amount(new BigDecimal("100.00"))
                    .build()))
            .isInstanceOf(TimeoutException.class);
    }
}
```

### 测试数据管理

```java
// TestDataBuilder.java
@Component
@Profile("test")
public class TestDataBuilder {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private OrderRepository orderRepository;

    public User createUser(String username) {
        return userRepository.save(User.builder()
            .username(username)
            .email(username + "@test.com")
            .build());
    }

    public Product createProduct(String name, BigDecimal price, int stock) {
        return productRepository.save(Product.builder()
            .name(name)
            .price(price)
            .stock(stock)
            .build());
    }

    public Order createOrder(User user, Product product, int quantity) {
        return orderRepository.save(Order.builder()
            .userId(user.getId())
            .productId(product.getId())
            .quantity(quantity)
            .totalAmount(product.getPrice().multiply(BigDecimal.valueOf(quantity)))
            .status(OrderStatus.CREATED)
            .build());
    }

    @Transactional
    public void cleanAll() {
        orderRepository.deleteAll();
        productRepository.deleteAll();
        userRepository.deleteAll();
    }
}
```

### 测试配置

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=MySQL
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
  redis:
    host: localhost
    port: 6379
  kafka:
    bootstrap-servers: ${spring.embedded.kafka.brokers}

logging:
  level:
    org.springframework.test: DEBUG
    org.testcontainers: INFO
```

## 最佳实践清单

- [ ] 使用真实数据库 (Testcontainers)
- [ ] 测试数据隔离 (@Transactional)
- [ ] Mock 外部服务 (WireMock)
- [ ] 验证数据库状态变更
- [ ] 测试异常和边界情况
- [ ] 使用测试专用配置文件
- [ ] 清理测试数据
- [ ] 控制测试执行时间
