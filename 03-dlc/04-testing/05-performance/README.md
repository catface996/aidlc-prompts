# 性能测试最佳实践

## 角色设定

你是一位精通性能测试的专家，擅长 JMeter、Gatling、k6、压力测试和性能分析。

## 提示词模板

### 性能测试设计

```
请帮我设计性能测试方案：
- 测试目标：[接口/页面/系统]
- 性能指标：[QPS/响应时间/并发数]
- 测试类型：[负载/压力/容量/浸泡]
- 测试工具：[JMeter/Gatling/k6]
- 测试环境：[配置信息]

请提供测试脚本和执行方案。
```

## 核心代码示例

### JMeter 测试计划

```xml
<!-- order-performance-test.jmx -->
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="订单系统性能测试">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments">
        <collectionProp name="Arguments.arguments">
          <elementProp name="BASE_URL" elementType="Argument">
            <stringProp name="Argument.name">BASE_URL</stringProp>
            <stringProp name="Argument.value">http://localhost:8080</stringProp>
          </elementProp>
        </collectionProp>
      </elementProp>
    </TestPlan>
    <hashTree>
      <!-- 线程组 -->
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="订单创建测试">
        <stringProp name="ThreadGroup.num_threads">100</stringProp>
        <stringProp name="ThreadGroup.ramp_time">60</stringProp>
        <stringProp name="ThreadGroup.duration">300</stringProp>
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
      </ThreadGroup>
      <hashTree>
        <!-- HTTP 请求 -->
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="创建订单">
          <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
            <collectionProp name="Arguments.arguments">
              <elementProp name="" elementType="HTTPArgument">
                <stringProp name="Argument.value">{"userId":${userId},"productId":${productId},"quantity":1}</stringProp>
              </elementProp>
            </collectionProp>
          </elementProp>
          <stringProp name="HTTPSampler.path">/api/orders</stringProp>
          <stringProp name="HTTPSampler.method">POST</stringProp>
        </HTTPSamplerProxy>
        <hashTree>
          <!-- 响应断言 -->
          <ResponseAssertion guiclass="AssertionGui" testclass="ResponseAssertion" testname="验证状态码">
            <collectionProp name="Asserion.test_strings">
              <stringProp>201</stringProp>
            </collectionProp>
            <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
          </ResponseAssertion>
        </hashTree>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

### Gatling 测试脚本

```scala
// OrderSimulation.scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class OrderSimulation extends Simulation {

  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .acceptEncodingHeader("gzip, deflate")

  // 用户登录获取 Token
  val login = exec(
    http("Login")
      .post("/api/auth/login")
      .body(StringBody("""{"username":"testuser","password":"password123"}"""))
      .check(jsonPath("$.token").saveAs("authToken"))
  )

  // 创建订单场景
  val createOrder = exec(
    http("Create Order")
      .post("/api/orders")
      .header("Authorization", "Bearer ${authToken}")
      .body(StringBody("""{"userId":1,"productId":${productId},"quantity":1}"""))
      .check(status.is(201))
      .check(jsonPath("$.id").saveAs("orderId"))
  )

  // 查询订单场景
  val getOrder = exec(
    http("Get Order")
      .get("/api/orders/${orderId}")
      .header("Authorization", "Bearer ${authToken}")
      .check(status.is(200))
  )

  // 查询订单列表
  val listOrders = exec(
    http("List Orders")
      .get("/api/orders")
      .header("Authorization", "Bearer ${authToken}")
      .queryParam("page", "0")
      .queryParam("size", "10")
      .check(status.is(200))
  )

  // 用户场景
  val orderScenario = scenario("Order Flow")
    .exec(login)
    .pause(1.second)
    .feed(csv("products.csv").circular)
    .repeat(10) {
      exec(createOrder)
        .pause(500.milliseconds, 1.second)
        .exec(getOrder)
        .pause(300.milliseconds)
    }
    .exec(listOrders)

  // 负载测试
  setUp(
    orderScenario.inject(
      // 预热阶段
      rampUsers(10).during(30.seconds),
      // 稳定负载
      constantUsersPerSec(50).during(5.minutes),
      // 峰值测试
      rampUsersPerSec(50).to(100).during(2.minutes),
      constantUsersPerSec(100).during(3.minutes),
      // 降压
      rampUsersPerSec(100).to(10).during(1.minute)
    )
  ).protocols(httpProtocol)
    .assertions(
      global.responseTime.percentile(95).lt(500),    // P95 < 500ms
      global.responseTime.percentile(99).lt(1000),   // P99 < 1s
      global.successfulRequests.percent.gt(99),       // 成功率 > 99%
      global.requestsPerSec.gt(100)                   // QPS > 100
    )
}

// 浸泡测试
class SoakTestSimulation extends Simulation {

  val httpProtocol = http.baseUrl("http://localhost:8080")

  val scenario = scenario("Soak Test")
    .exec(/* 测试逻辑 */)

  setUp(
    scenario.inject(
      constantUsersPerSec(50).during(4.hours)  // 持续4小时
    )
  ).protocols(httpProtocol)
}

// 压力测试
class StressTestSimulation extends Simulation {

  val httpProtocol = http.baseUrl("http://localhost:8080")

  val scenario = scenario("Stress Test")
    .exec(/* 测试逻辑 */)

  setUp(
    scenario.inject(
      incrementUsersPerSec(10)
        .times(10)
        .eachLevelLasting(2.minutes)
        .separatedByRampsLasting(30.seconds)
        .startingFrom(10)
    )
  ).protocols(httpProtocol)
}
```

### k6 测试脚本

```javascript
// order-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// 自定义指标
const errorRate = new Rate('errors');
const orderCreationTime = new Trend('order_creation_time');

// 测试配置
export const options = {
  stages: [
    { duration: '1m', target: 50 },   // 预热
    { duration: '3m', target: 100 },  // 稳定负载
    { duration: '2m', target: 200 },  // 峰值
    { duration: '1m', target: 0 },    // 降压
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    errors: ['rate<0.01'],
    order_creation_time: ['p(95)<300'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

// 登录获取 Token
export function setup() {
  const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    username: 'testuser',
    password: 'password123',
  }), {
    headers: { 'Content-Type': 'application/json' },
  });

  return { token: loginRes.json('token') };
}

// 主测试函数
export default function (data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };

  // 创建订单
  const orderStart = Date.now();
  const orderRes = http.post(`${BASE_URL}/api/orders`, JSON.stringify({
    userId: 1,
    productId: Math.floor(Math.random() * 100) + 1,
    quantity: Math.floor(Math.random() * 5) + 1,
  }), { headers });

  orderCreationTime.add(Date.now() - orderStart);

  const orderSuccess = check(orderRes, {
    'order created': (r) => r.status === 201,
    'order has id': (r) => r.json('id') !== undefined,
  });

  errorRate.add(!orderSuccess);

  if (orderSuccess) {
    const orderId = orderRes.json('id');

    sleep(0.5);

    // 查询订单
    const getRes = http.get(`${BASE_URL}/api/orders/${orderId}`, { headers });
    check(getRes, {
      'get order success': (r) => r.status === 200,
    });
  }

  sleep(Math.random() * 2 + 1);
}

// 测试结束后的清理
export function teardown(data) {
  console.log('Test completed');
}
```

### 前端性能测试 (Lighthouse)

```javascript
// lighthouse-test.js
const lighthouse = require('lighthouse');
const chromeLauncher = require('chrome-launcher');

async function runLighthouse(url) {
  const chrome = await chromeLauncher.launch({ chromeFlags: ['--headless'] });

  const options = {
    logLevel: 'info',
    output: 'json',
    onlyCategories: ['performance'],
    port: chrome.port,
  };

  const runnerResult = await lighthouse(url, options);
  const report = runnerResult.lhr;

  console.log('Performance Score:', report.categories.performance.score * 100);
  console.log('First Contentful Paint:', report.audits['first-contentful-paint'].displayValue);
  console.log('Largest Contentful Paint:', report.audits['largest-contentful-paint'].displayValue);
  console.log('Time to Interactive:', report.audits['interactive'].displayValue);
  console.log('Total Blocking Time:', report.audits['total-blocking-time'].displayValue);
  console.log('Cumulative Layout Shift:', report.audits['cumulative-layout-shift'].displayValue);

  await chrome.kill();

  // 性能阈值验证
  const assertions = {
    'performance score >= 90': report.categories.performance.score >= 0.9,
    'FCP < 2s': report.audits['first-contentful-paint'].numericValue < 2000,
    'LCP < 2.5s': report.audits['largest-contentful-paint'].numericValue < 2500,
    'TTI < 5s': report.audits['interactive'].numericValue < 5000,
  };

  Object.entries(assertions).forEach(([name, passed]) => {
    console.log(`${passed ? '✓' : '✗'} ${name}`);
  });

  return report;
}

runLighthouse('http://localhost:3000');
```

### 数据库性能测试

```java
// DatabasePerformanceTest.java
@SpringBootTest
class DatabasePerformanceTest {

    @Autowired
    private OrderRepository orderRepository;

    @Test
    @DisplayName("批量插入性能测试")
    void shouldMeasureBatchInsertPerformance() {
        int batchSize = 1000;
        List<Order> orders = generateOrders(batchSize);

        long startTime = System.currentTimeMillis();
        orderRepository.saveAll(orders);
        long duration = System.currentTimeMillis() - startTime;

        System.out.println("Batch insert " + batchSize + " orders took: " + duration + "ms");
        assertThat(duration).isLessThan(5000);  // 5秒内完成
    }

    @Test
    @DisplayName("查询性能测试")
    void shouldMeasureQueryPerformance() {
        // 预热
        orderRepository.findAll(PageRequest.of(0, 100));

        // 测试
        int iterations = 100;
        long totalTime = 0;

        for (int i = 0; i < iterations; i++) {
            long start = System.nanoTime();
            orderRepository.findByUserIdAndStatus(1L, OrderStatus.CREATED,
                PageRequest.of(0, 20));
            totalTime += System.nanoTime() - start;
        }

        double avgTime = totalTime / iterations / 1_000_000.0;
        System.out.println("Average query time: " + avgTime + "ms");
        assertThat(avgTime).isLessThan(50);  // 平均 50ms 以内
    }
}
```

## 性能指标

| 指标 | 说明 | 参考值 |
|------|------|--------|
| 响应时间 P95 | 95% 请求的响应时间 | < 500ms |
| 响应时间 P99 | 99% 请求的响应时间 | < 1s |
| QPS | 每秒请求数 | 根据业务需求 |
| 错误率 | 失败请求比例 | < 1% |
| 并发数 | 同时处理的请求数 | 根据资源配置 |
| 吞吐量 | 单位时间处理的数据量 | 根据业务需求 |

## 最佳实践清单

- [ ] 明确性能目标和指标
- [ ] 准备接近生产的测试环境
- [ ] 准备足够的测试数据
- [ ] 逐步增加负载
- [ ] 监控系统资源
- [ ] 分析性能瓶颈
- [ ] 记录和对比测试结果
- [ ] 持续集成性能测试
