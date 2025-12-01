# SkyWalking 链路追踪最佳实践

## 角色设定

你是一位精通 Apache SkyWalking 的 APM 专家，擅长分布式链路追踪、性能监控和故障诊断。

## 提示词模板

### SkyWalking 部署配置

```
请帮我配置 SkyWalking：
- 部署模式：[单机/集群]
- 存储后端：[Elasticsearch/MySQL/H2]
- 采集方式：[Java Agent/SkyWalking SDK]
- 监控范围：[服务列表]
- 采样策略：[全量/采样率]

请提供部署配置和接入指南。
```

## 核心配置示例

### Agent 配置

```properties
# agent/config/agent.config
# 服务名称
agent.service_name=${SW_AGENT_NAME:order-service}

# OAP 服务器地址
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:localhost:11800}

# 采样率 (0-10000, 10000=100%)
agent.sample_n_per_3_secs=${SW_AGENT_SAMPLE:10000}

# 忽略的请求路径
agent.ignore_suffix=${SW_AGENT_IGNORE_SUFFIX:.jpg,.jpeg,.png,.css,.js}

# 实例名称
agent.instance_name=${SW_AGENT_INSTANCE_NAME:}

# 命名空间 (用于区分环境)
agent.namespace=${SW_AGENT_NAMESPACE:}

# 日志配置
logging.level=${SW_LOGGING_LEVEL:INFO}
logging.file_name=${SW_LOGGING_FILE_NAME:skywalking-api.log}
logging.max_file_size=${SW_LOGGING_MAX_FILE_SIZE:300 * 1024 * 1024}

# 插件配置
plugin.toolkit.log.transmit_formatted=${SW_PLUGIN_TOOLKIT_LOG_TRANSMIT_FORMATTED:true}
plugin.jdbc.trace_sql_parameters=${SW_JDBC_TRACE_SQL_PARAMETERS:true}
plugin.jdbc.sql_parameters_max_length=${SW_JDBC_SQL_PARAMETERS_MAX_LENGTH:512}

# Spring Cloud Gateway 插件
plugin.springcloud.gateway.collect_request_body=${SW_PLUGIN_SPRINGCLOUD_GATEWAY_COLLECT_REQUEST_BODY:true}

# Kafka 插件
plugin.kafka.bootstrap_servers=${SW_PLUGIN_KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
plugin.kafka.producer_config=${SW_PLUGIN_KAFKA_PRODUCER_CONFIG:}
```

### Docker Compose 部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: elasticsearch:7.17.10
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  skywalking-oap:
    image: apache/skywalking-oap-server:9.6.0
    container_name: skywalking-oap
    depends_on:
      elasticsearch:
        condition: service_healthy
    environment:
      SW_STORAGE: elasticsearch
      SW_STORAGE_ES_CLUSTER_NODES: elasticsearch:9200
      SW_HEALTH_CHECKER: default
      SW_TELEMETRY: prometheus
      JAVA_OPTS: "-Xms512m -Xmx512m"
    ports:
      - "11800:11800"  # gRPC
      - "12800:12800"  # HTTP
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:12800/internal/l7check || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  skywalking-ui:
    image: apache/skywalking-ui:9.6.0
    container_name: skywalking-ui
    depends_on:
      skywalking-oap:
        condition: service_healthy
    environment:
      SW_OAP_ADDRESS: http://skywalking-oap:12800
    ports:
      - "8080:8080"

volumes:
  es-data:
```

### Kubernetes 部署

```yaml
# skywalking-oap.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: skywalking-oap
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: skywalking-oap
  template:
    metadata:
      labels:
        app: skywalking-oap
    spec:
      containers:
        - name: oap
          image: apache/skywalking-oap-server:9.6.0
          ports:
            - containerPort: 11800
              name: grpc
            - containerPort: 12800
              name: http
          env:
            - name: SW_STORAGE
              value: elasticsearch
            - name: SW_STORAGE_ES_CLUSTER_NODES
              value: elasticsearch:9200
            - name: SW_CLUSTER
              value: kubernetes
            - name: SW_CLUSTER_K8S_NAMESPACE
              value: monitoring
            - name: SW_CLUSTER_K8S_LABEL
              value: app=skywalking-oap
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /internal/l7check
              port: 12800
            initialDelaySeconds: 60
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /internal/l7check
              port: 12800
            initialDelaySeconds: 30
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: skywalking-oap
  namespace: monitoring
spec:
  selector:
    app: skywalking-oap
  ports:
    - name: grpc
      port: 11800
      targetPort: 11800
    - name: http
      port: 12800
      targetPort: 12800
```

### Java 应用接入

```dockerfile
# Dockerfile
FROM eclipse-temurin:17-jre-alpine

# 下载 SkyWalking Agent
RUN wget -O /tmp/skywalking-agent.tar.gz \
    https://archive.apache.org/dist/skywalking/java-agent/9.0.0/apache-skywalking-java-agent-9.0.0.tgz && \
    tar -xzf /tmp/skywalking-agent.tar.gz -C /opt && \
    rm /tmp/skywalking-agent.tar.gz

WORKDIR /app
COPY target/*.jar app.jar

ENV SW_AGENT_NAME=order-service
ENV SW_AGENT_COLLECTOR_BACKEND_SERVICES=skywalking-oap:11800

ENTRYPOINT ["java", \
    "-javaagent:/opt/skywalking-agent/skywalking-agent.jar", \
    "-jar", "app.jar"]
```

### 自定义 Span

```java
@Service
@Slf4j
public class OrderService {

    /**
     * 使用 @Trace 注解创建 Span
     */
    @Trace(operationName = "createOrder")
    @Tags({
        @Tag(key = "userId", value = "arg[0]"),
        @Tag(key = "orderId", value = "returnedObj.id")
    })
    public Order createOrder(Long userId, OrderRequest request) {
        // 业务逻辑
        return order;
    }

    /**
     * 手动创建 Span
     */
    public void processOrder(Order order) {
        AbstractSpan span = ContextManager.createLocalSpan("processOrder");
        span.tag(new StringTag("orderId"), order.getId().toString());

        try {
            // 处理逻辑
            validateOrder(order);
            saveOrder(order);

            span.log(System.currentTimeMillis(), "Order processed successfully");
        } catch (Exception e) {
            span.errorOccurred();
            span.log(System.currentTimeMillis(), e.getMessage());
            throw e;
        } finally {
            ContextManager.stopSpan();
        }
    }

    /**
     * 异步任务追踪
     */
    @Trace(operationName = "asyncTask")
    public void asyncProcess(Order order) {
        ContextSnapshot snapshot = ContextManager.capture();

        CompletableFuture.runAsync(() -> {
            ContextManager.continued(snapshot);
            try {
                // 异步处理逻辑
            } finally {
                ContextManager.stopSpan();
            }
        });
    }
}
```

### 日志关联

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%tid] [%thread] %-5level %logger{36} - %msg%n</pattern>
            </layout>
        </encoder>
    </appender>

    <!-- 日志上报到 SkyWalking -->
    <appender name="SKYWALKING" class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.log.GRPCLogClientAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%tid] [%thread] %-5level %logger{36} - %msg%n</pattern>
            </layout>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="SKYWALKING"/>
    </root>
</configuration>
```

### Log4j2 集成

```xml
<!-- log4j2.xml -->
<Configuration>
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} [%traceId] [%thread] %-5level %logger{36} - %msg%n"/>
        </Console>

        <GRPCLogClientAppender name="grpc-log">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} [%traceId] [%thread] %-5level %logger{36} - %msg%n"/>
        </GRPCLogClientAppender>
    </Appenders>

    <Loggers>
        <Root level="INFO">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="grpc-log"/>
        </Root>
    </Loggers>
</Configuration>
```

### 自定义指标

```java
@Component
@Slf4j
public class CustomMetrics {

    private final MeterFactory meterFactory;
    private final Counter orderCounter;
    private final Histogram orderLatency;

    public CustomMetrics() {
        this.meterFactory = MeterFactory.INSTANCE;

        // 计数器
        this.orderCounter = meterFactory.createCounter(
            "order_total",
            "Total number of orders"
        );

        // 直方图
        this.orderLatency = meterFactory.createHistogram(
            "order_latency_seconds",
            "Order processing latency"
        );
    }

    public void recordOrder(String status) {
        orderCounter.increment(1, "status", status);
    }

    public void recordLatency(long milliseconds) {
        orderLatency.addValue(milliseconds / 1000.0);
    }
}
```

### 告警规则

```yaml
# alarm-settings.yml
rules:
  # 服务响应时间告警
  service_resp_time_rule:
    metrics-name: service_resp_time
    op: ">"
    threshold: 1000
    period: 10
    count: 3
    silence-period: 5
    message: Service {name} response time is more than 1000ms in last 10 minutes.

  # 服务成功率告警
  service_sla_rule:
    metrics-name: service_sla
    op: "<"
    threshold: 9800
    period: 10
    count: 2
    silence-period: 3
    message: Service {name} successful rate is less than 98% in last 10 minutes.

  # 端点响应时间告警
  endpoint_resp_time_rule:
    metrics-name: endpoint_resp_time
    threshold: 2000
    op: ">"
    period: 10
    count: 2
    silence-period: 5
    message: Endpoint {name} response time is more than 2000ms in last 10 minutes.

  # 数据库响应时间告警
  database_access_resp_time_rule:
    metrics-name: database_access_resp_time
    threshold: 500
    op: ">"
    period: 10
    count: 3
    silence-period: 5
    message: Database {name} response time is more than 500ms in last 10 minutes.

webhooks:
  - url: http://alert-service:8080/api/alerts
    timeout: 10000
```

### Spring Boot Actuator 集成

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
```

### 过滤和采样配置

```properties
# 忽略特定端点
agent.trace.ignore_path=${SW_AGENT_TRACE_IGNORE_PATH:/actuator/**,/health,/ready}

# 采样配置
agent.sample_n_per_3_secs=1000

# 忽略异常
statuscheck.ignored_exceptions=${SW_STATUSCHECK_IGNORED_EXCEPTIONS:java.io.IOException}

# 最大 Span 数量限制
agent.span_limit_per_segment=${SW_AGENT_SPAN_LIMIT:300}
```

## OAP 配置

```yaml
# application.yml (OAP Server)
storage:
  selector: elasticsearch
  elasticsearch:
    nameSpace: ${SW_NAMESPACE:"skywalking"}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200}
    protocol: ${SW_STORAGE_ES_HTTP_PROTOCOL:"http"}
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:1}
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:1}
    recordDataTTL: ${SW_STORAGE_ES_RECORD_DATA_TTL:7}
    metricsDataTTL: ${SW_STORAGE_ES_METRICS_DATA_TTL:90}

cluster:
  selector: ${SW_CLUSTER:standalone}
  kubernetes:
    namespace: ${SW_CLUSTER_K8S_NAMESPACE:default}
    labelSelector: ${SW_CLUSTER_K8S_LABEL:app=collector}
    uidEnvName: ${SW_CLUSTER_K8S_UID:SKYWALKING_COLLECTOR_UID}

receiver-trace:
  selector: ${SW_RECEIVER_TRACE:default}
  default:
    sampleRate: ${SW_TRACE_SAMPLE_RATE:10000}
```

## 最佳实践清单

- [ ] 合理设置采样率
- [ ] 配置日志与 TraceId 关联
- [ ] 设置告警规则
- [ ] 忽略健康检查等端点
- [ ] Elasticsearch 配置合理 TTL
- [ ] 生产环境集群部署 OAP
- [ ] 自定义业务 Span 和指标
- [ ] 异步任务正确传递上下文
