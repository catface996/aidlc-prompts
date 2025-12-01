# API 测试最佳实践

## 角色设定

你是一位精通 API 测试的质量工程师，擅长 REST Assured、Postman、接口契约测试和自动化 API 验证。

## 提示词模板

### API 测试编写

```
请帮我编写 API 测试：
- 接口信息：[HTTP 方法和路径]
- 请求参数：[Headers/Body/Query]
- 响应验证：[状态码/响应体/Headers]
- 测试场景：[正常/异常/边界]

请提供完整的测试代码。
```

## 核心代码示例

### REST Assured 测试

```java
// OrderApiTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderApiTest {

    @LocalServerPort
    private int port;

    @Autowired
    private JwtTokenProvider tokenProvider;

    private String baseUrl;
    private String authToken;

    @BeforeEach
    void setUp() {
        baseUrl = "http://localhost:" + port;
        authToken = tokenProvider.generateToken("testuser");

        RestAssured.baseURI = baseUrl;
        RestAssured.enableLoggingOfRequestAndResponseIfValidationFails();
    }

    @Test
    @DisplayName("创建订单 - 成功")
    void shouldCreateOrderSuccessfully() {
        String requestBody = """
            {
                "userId": 1,
                "productId": 100,
                "quantity": 2,
                "shippingAddress": {
                    "name": "张三",
                    "phone": "13800138000",
                    "address": "北京市朝阳区"
                }
            }
            """;

        given()
            .header("Authorization", "Bearer " + authToken)
            .contentType(ContentType.JSON)
            .body(requestBody)
        .when()
            .post("/api/orders")
        .then()
            .statusCode(201)
            .contentType(ContentType.JSON)
            .body("id", notNullValue())
            .body("status", equalTo("CREATED"))
            .body("userId", equalTo(1))
            .body("totalAmount", greaterThan(0f))
            .body("createdAt", notNullValue())
            .header("Location", matchesPattern("/api/orders/\\d+"));
    }

    @Test
    @DisplayName("创建订单 - 未授权")
    void shouldReturn401WhenUnauthorized() {
        given()
            .contentType(ContentType.JSON)
            .body("{}")
        .when()
            .post("/api/orders")
        .then()
            .statusCode(401)
            .body("code", equalTo("UNAUTHORIZED"));
    }

    @Test
    @DisplayName("创建订单 - 参数校验失败")
    void shouldReturn400WhenInvalidRequest() {
        String invalidRequest = """
            {
                "userId": null,
                "productId": 100,
                "quantity": -1
            }
            """;

        given()
            .header("Authorization", "Bearer " + authToken)
            .contentType(ContentType.JSON)
            .body(invalidRequest)
        .when()
            .post("/api/orders")
        .then()
            .statusCode(400)
            .body("code", equalTo("VALIDATION_ERROR"))
            .body("errors", hasSize(greaterThan(0)))
            .body("errors.field", hasItems("userId", "quantity"));
    }

    @Test
    @DisplayName("获取订单列表 - 分页")
    void shouldReturnPaginatedOrders() {
        given()
            .header("Authorization", "Bearer " + authToken)
            .queryParam("page", 0)
            .queryParam("size", 10)
            .queryParam("sort", "createdAt,desc")
        .when()
            .get("/api/orders")
        .then()
            .statusCode(200)
            .body("content", notNullValue())
            .body("content.size()", lessThanOrEqualTo(10))
            .body("totalPages", greaterThanOrEqualTo(0))
            .body("totalElements", greaterThanOrEqualTo(0))
            .body("pageable.pageNumber", equalTo(0))
            .body("pageable.pageSize", equalTo(10));
    }

    @Test
    @DisplayName("获取订单详情")
    void shouldReturnOrderDetail() {
        // 先创建订单
        int orderId = createTestOrder();

        given()
            .header("Authorization", "Bearer " + authToken)
            .pathParam("id", orderId)
        .when()
            .get("/api/orders/{id}")
        .then()
            .statusCode(200)
            .body("id", equalTo(orderId))
            .body("items", notNullValue())
            .body("shippingAddress", notNullValue())
            .body("paymentInfo", notNullValue());
    }

    @Test
    @DisplayName("获取订单详情 - 不存在")
    void shouldReturn404WhenOrderNotFound() {
        given()
            .header("Authorization", "Bearer " + authToken)
            .pathParam("id", 99999)
        .when()
            .get("/api/orders/{id}")
        .then()
            .statusCode(404)
            .body("code", equalTo("ORDER_NOT_FOUND"));
    }

    @Test
    @DisplayName("更新订单状态")
    void shouldUpdateOrderStatus() {
        int orderId = createTestOrder();

        given()
            .header("Authorization", "Bearer " + authToken)
            .contentType(ContentType.JSON)
            .pathParam("id", orderId)
            .body("{\"status\": \"PAID\"}")
        .when()
            .patch("/api/orders/{id}/status")
        .then()
            .statusCode(200)
            .body("status", equalTo("PAID"));
    }

    @Test
    @DisplayName("删除订单 - 只能删除待支付订单")
    void shouldDeletePendingOrder() {
        int orderId = createTestOrder();

        given()
            .header("Authorization", "Bearer " + authToken)
            .pathParam("id", orderId)
        .when()
            .delete("/api/orders/{id}")
        .then()
            .statusCode(204);

        // 验证已删除
        given()
            .header("Authorization", "Bearer " + authToken)
            .pathParam("id", orderId)
        .when()
            .get("/api/orders/{id}")
        .then()
            .statusCode(404);
    }

    // 响应时间测试
    @Test
    @DisplayName("API 响应时间")
    void shouldRespondWithinTimeout() {
        given()
            .header("Authorization", "Bearer " + authToken)
        .when()
            .get("/api/orders")
        .then()
            .time(lessThan(2000L));  // 2秒内响应
    }

    // 响应 Schema 验证
    @Test
    @DisplayName("响应 Schema 验证")
    void shouldMatchResponseSchema() {
        given()
            .header("Authorization", "Bearer " + authToken)
        .when()
            .get("/api/orders/1")
        .then()
            .body(matchesJsonSchemaInClasspath("schemas/order-response.json"));
    }

    private int createTestOrder() {
        return given()
            .header("Authorization", "Bearer " + authToken)
            .contentType(ContentType.JSON)
            .body("""
                {
                    "userId": 1,
                    "productId": 100,
                    "quantity": 1
                }
                """)
        .when()
            .post("/api/orders")
        .then()
            .extract()
            .path("id");
    }
}
```

### 契约测试 (Spring Cloud Contract)

```java
// 消费者端契约
// contracts/order/createOrder.groovy
Contract.make {
    description "创建订单成功"

    request {
        method POST()
        url '/api/orders'
        headers {
            contentType applicationJson()
            header 'Authorization': $(consumer(containing('Bearer')))
        }
        body([
            userId: $(consumer(regex('[0-9]+')), producer(1)),
            productId: $(consumer(regex('[0-9]+')), producer(100)),
            quantity: $(consumer(regex('[1-9][0-9]*')), producer(2))
        ])
    }

    response {
        status CREATED()
        headers {
            contentType applicationJson()
        }
        body([
            id: $(producer(regex('[0-9]+')), consumer(1)),
            userId: fromRequest().body('$.userId'),
            status: 'CREATED',
            totalAmount: $(producer(regex('[0-9]+\\.?[0-9]*')), consumer(199.98))
        ])
    }
}

// 消费者测试
@SpringBootTest
@AutoConfigureStubRunner(
    ids = "com.example:order-service:+:stubs:8080",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class OrderClientContractTest {

    @Autowired
    private OrderClient orderClient;

    @Test
    void shouldCreateOrder() {
        OrderRequest request = new OrderRequest(1L, 100L, 2);

        OrderResponse response = orderClient.createOrder(request);

        assertThat(response.getId()).isNotNull();
        assertThat(response.getStatus()).isEqualTo("CREATED");
    }
}
```

### Postman/Newman 测试

```json
// postman/order-api-tests.json
{
  "info": {
    "name": "Order API Tests",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    { "key": "baseUrl", "value": "http://localhost:8080" },
    { "key": "authToken", "value": "" }
  ],
  "item": [
    {
      "name": "Login",
      "event": [
        {
          "listen": "test",
          "script": {
            "exec": [
              "pm.test('Status is 200', () => pm.response.to.have.status(200));",
              "const json = pm.response.json();",
              "pm.collectionVariables.set('authToken', json.token);"
            ]
          }
        }
      ],
      "request": {
        "method": "POST",
        "url": "{{baseUrl}}/api/auth/login",
        "body": {
          "mode": "raw",
          "raw": "{\"username\": \"testuser\", \"password\": \"password123\"}"
        }
      }
    },
    {
      "name": "Create Order",
      "event": [
        {
          "listen": "test",
          "script": {
            "exec": [
              "pm.test('Status is 201', () => pm.response.to.have.status(201));",
              "pm.test('Has order id', () => {",
              "    const json = pm.response.json();",
              "    pm.expect(json.id).to.be.a('number');",
              "    pm.collectionVariables.set('orderId', json.id);",
              "});",
              "pm.test('Status is CREATED', () => {",
              "    pm.expect(pm.response.json().status).to.equal('CREATED');",
              "});"
            ]
          }
        }
      ],
      "request": {
        "method": "POST",
        "url": "{{baseUrl}}/api/orders",
        "header": [
          { "key": "Authorization", "value": "Bearer {{authToken}}" }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\"userId\": 1, \"productId\": 100, \"quantity\": 2}"
        }
      }
    },
    {
      "name": "Get Order",
      "event": [
        {
          "listen": "test",
          "script": {
            "exec": [
              "pm.test('Status is 200', () => pm.response.to.have.status(200));",
              "pm.test('Order id matches', () => {",
              "    const orderId = pm.collectionVariables.get('orderId');",
              "    pm.expect(pm.response.json().id).to.equal(orderId);",
              "});"
            ]
          }
        }
      ],
      "request": {
        "method": "GET",
        "url": "{{baseUrl}}/api/orders/{{orderId}}",
        "header": [
          { "key": "Authorization", "value": "Bearer {{authToken}}" }
        ]
      }
    }
  ]
}
```

### OpenAPI 规范验证

```java
// OpenApiValidationTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OpenApiValidationTest {

    @LocalServerPort
    private int port;

    private OpenApiInteractionValidator validator;

    @BeforeEach
    void setUp() {
        validator = OpenApiInteractionValidator
            .createForSpecificationUrl("http://localhost:" + port + "/v3/api-docs")
            .build();
    }

    @Test
    void shouldValidateCreateOrderRequest() {
        SimpleRequest request = SimpleRequest.Builder
            .post("/api/orders")
            .withContentType("application/json")
            .withBody("{\"userId\": 1, \"productId\": 100, \"quantity\": 2}")
            .build();

        ValidationReport report = validator.validateRequest(request);
        assertThat(report.hasErrors()).isFalse();
    }

    @Test
    void shouldValidateCreateOrderResponse() {
        SimpleResponse response = SimpleResponse.Builder
            .status(201)
            .withContentType("application/json")
            .withBody("{\"id\": 1, \"status\": \"CREATED\", \"totalAmount\": 199.98}")
            .build();

        ValidationReport report = validator.validateResponse("/api/orders", Method.POST, response);
        assertThat(report.hasErrors()).isFalse();
    }
}
```

## HTTP 状态码测试矩阵

| 场景 | 期望状态码 |
|------|-----------|
| 创建成功 | 201 Created |
| 查询成功 | 200 OK |
| 更新成功 | 200 OK |
| 删除成功 | 204 No Content |
| 参数错误 | 400 Bad Request |
| 未授权 | 401 Unauthorized |
| 无权限 | 403 Forbidden |
| 资源不存在 | 404 Not Found |
| 业务冲突 | 409 Conflict |
| 服务器错误 | 500 Internal Server Error |

## 最佳实践清单

- [ ] 覆盖所有 HTTP 方法
- [ ] 验证请求参数校验
- [ ] 验证响应结构和数据
- [ ] 测试认证和授权
- [ ] 测试错误处理
- [ ] 响应时间验证
- [ ] 契约测试保证兼容性
- [ ] CI/CD 集成自动化
