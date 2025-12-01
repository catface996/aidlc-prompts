# Monitoring 监控最佳实践

## 角色设定

你是一位精通微服务监控的 SRE 专家，擅长 Prometheus + Grafana 监控体系、指标设计和告警配置。

## 提示词模板

### 监控方案设计

```
请帮我设计监控方案：
- 监控类型：[应用/基础设施/业务]
- 指标需求：[列出关键指标]
- 告警策略：[描述告警规则]
- 可视化需求：[Dashboard 需求]
- 存储需求：[数据保留时长]

请提供配置和最佳实践。
```

## 核心配置示例

### Spring Boot Actuator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers,env
      base-path: /actuator
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true
    prometheus:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      env: ${SPRING_PROFILES_ACTIVE:dev}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.9, 0.95, 0.99
      sla:
        http.server.requests: 100ms, 500ms, 1s
  health:
    redis:
      enabled: true
    db:
      enabled: true
    diskspace:
      enabled: true
```

### 依赖配置

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- Micrometer Tracing -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
</dependencies>
```

### 自定义指标

```java
@Component
@RequiredArgsConstructor
public class CustomMetrics {

    private final MeterRegistry meterRegistry;

    // 计数器
    private Counter orderCreatedCounter;
    private Counter orderFailedCounter;

    // 计量器
    private AtomicInteger activeUsers;

    // 计时器
    private Timer orderProcessTimer;

    // 分布摘要
    private DistributionSummary orderAmountSummary;

    @PostConstruct
    public void init() {
        // 订单创建计数器
        orderCreatedCounter = Counter.builder("order.created.total")
            .description("Total number of orders created")
            .tags("type", "created")
            .register(meterRegistry);

        orderFailedCounter = Counter.builder("order.failed.total")
            .description("Total number of failed orders")
            .tags("type", "failed")
            .register(meterRegistry);

        // 活跃用户数
        activeUsers = meterRegistry.gauge("users.active",
            new AtomicInteger(0));

        // 订单处理时间
        orderProcessTimer = Timer.builder("order.process.duration")
            .description("Order processing duration")
            .publishPercentiles(0.5, 0.9, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(meterRegistry);

        // 订单金额分布
        orderAmountSummary = DistributionSummary.builder("order.amount")
            .description("Order amount distribution")
            .baseUnit("yuan")
            .publishPercentiles(0.5, 0.9, 0.95, 0.99)
            .register(meterRegistry);
    }

    public void incrementOrderCreated() {
        orderCreatedCounter.increment();
    }

    public void incrementOrderFailed(String reason) {
        Counter.builder("order.failed.total")
            .tag("reason", reason)
            .register(meterRegistry)
            .increment();
    }

    public void recordOrderProcessTime(long milliseconds) {
        orderProcessTimer.record(milliseconds, TimeUnit.MILLISECONDS);
    }

    public void recordOrderAmount(double amount) {
        orderAmountSummary.record(amount);
    }

    public void setActiveUsers(int count) {
        activeUsers.set(count);
    }
}
```

### 业务指标切面

```java
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class MetricsAspect {

    private final MeterRegistry meterRegistry;

    @Around("@annotation(timed)")
    public Object recordTime(ProceedingJoinPoint joinPoint, Timed timed) throws Throwable {
        String metricName = timed.value().isEmpty() ?
            joinPoint.getSignature().toShortString() : timed.value();

        Timer.Sample sample = Timer.start(meterRegistry);
        String status = "success";

        try {
            return joinPoint.proceed();
        } catch (Exception e) {
            status = "error";
            throw e;
        } finally {
            sample.stop(Timer.builder(metricName)
                .tag("class", joinPoint.getTarget().getClass().getSimpleName())
                .tag("method", joinPoint.getSignature().getName())
                .tag("status", status)
                .register(meterRegistry));
        }
    }

    @Around("@annotation(counted)")
    public Object recordCount(ProceedingJoinPoint joinPoint, Counted counted) throws Throwable {
        String metricName = counted.value().isEmpty() ?
            joinPoint.getSignature().toShortString() : counted.value();

        try {
            Object result = joinPoint.proceed();
            Counter.builder(metricName)
                .tag("status", "success")
                .register(meterRegistry)
                .increment();
            return result;
        } catch (Exception e) {
            Counter.builder(metricName)
                .tag("status", "error")
                .tag("exception", e.getClass().getSimpleName())
                .register(meterRegistry)
                .increment();
            throw e;
        }
    }
}
```

### 健康检查

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {

    private final RedisTemplate<String, Object> redisTemplate;
    private final DataSource dataSource;

    public CustomHealthIndicator(RedisTemplate<String, Object> redisTemplate,
                                 DataSource dataSource) {
        this.redisTemplate = redisTemplate;
        this.dataSource = dataSource;
    }

    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();

        // 检查 Redis
        try {
            redisTemplate.getConnectionFactory().getConnection().ping();
            details.put("redis", "UP");
        } catch (Exception e) {
            details.put("redis", "DOWN: " + e.getMessage());
            return Health.down().withDetails(details).build();
        }

        // 检查数据库
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(5)) {
                details.put("database", "UP");
            } else {
                details.put("database", "DOWN");
                return Health.down().withDetails(details).build();
            }
        } catch (SQLException e) {
            details.put("database", "DOWN: " + e.getMessage());
            return Health.down().withDetails(details).build();
        }

        return Health.up().withDetails(details).build();
    }
}

// Liveness Probe
@Component
public class LivenessHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        // 应用存活检查
        return Health.up().build();
    }
}

// Readiness Probe
@Component
public class ReadinessHealthIndicator implements HealthIndicator {

    private final AtomicBoolean ready = new AtomicBoolean(false);

    @Override
    public Health health() {
        if (ready.get()) {
            return Health.up().build();
        }
        return Health.down().withDetail("reason", "Application not ready").build();
    }

    public void setReady(boolean ready) {
        this.ready.set(ready);
    }
}
```

### Prometheus 配置

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:
  # Prometheus 自身
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Spring Boot 应用
  - job_name: 'spring-boot'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s
    static_configs:
      - targets:
          - 'order-service:8080'
          - 'user-service:8080'
          - 'payment-service:8080'
        labels:
          env: 'prod'

  # Kubernetes 服务发现
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod

  # Node Exporter
  - job_name: 'node'
    static_configs:
      - targets:
          - 'node-exporter:9100'

  # MySQL
  - job_name: 'mysql'
    static_configs:
      - targets:
          - 'mysqld-exporter:9104'

  # Redis
  - job_name: 'redis'
    static_configs:
      - targets:
          - 'redis-exporter:9121'
```

### 告警规则

```yaml
# alert-rules.yml
groups:
  - name: application
    rules:
      # 服务宕机
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute."

      # 高错误率
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (application)
          /
          sum(rate(http_server_requests_seconds_count[5m])) by (application)
          > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate on {{ $labels.application }}"
          description: "Error rate is {{ $value | humanizePercentage }} (> 5%)"

      # 响应时间过长
      - alert: HighResponseTime
        expr: |
          histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application))
          > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time on {{ $labels.application }}"
          description: "P95 response time is {{ $value | humanizeDuration }}"

      # 内存使用过高
      - alert: HighMemoryUsage
        expr: |
          jvm_memory_used_bytes{area="heap"}
          /
          jvm_memory_max_bytes{area="heap"}
          > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High heap memory usage on {{ $labels.application }}"
          description: "Heap memory usage is {{ $value | humanizePercentage }}"

      # CPU 使用过高
      - alert: HighCpuUsage
        expr: |
          system_cpu_usage > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.application }}"
          description: "CPU usage is {{ $value | humanizePercentage }}"

  - name: infrastructure
    rules:
      # 磁盘空间
      - alert: DiskSpaceLow
        expr: |
          (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk space is {{ $value | humanizePercentage }} available"

      # 数据库连接池
      - alert: DatabaseConnectionPoolExhausted
        expr: |
          hikaricp_connections_active / hikaricp_connections_max > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool nearly exhausted"
          description: "Connection pool usage is {{ $value | humanizePercentage }}"
```

### Alertmanager 配置

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager'
  smtp_auth_password: 'password'

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'critical-alerts'
      continue: true
    - match:
        severity: warning
      receiver: 'warning-alerts'

receivers:
  - name: 'default'
    email_configs:
      - to: 'ops@example.com'

  - name: 'critical-alerts'
    email_configs:
      - to: 'oncall@example.com'
    webhook_configs:
      - url: 'http://alert-service:8080/api/alerts/critical'

  - name: 'warning-alerts'
    email_configs:
      - to: 'dev@example.com'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

### Grafana Dashboard JSON

```json
{
  "title": "Spring Boot Application",
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_server_requests_seconds_count[5m])) by (application)",
          "legendFormat": "{{application}}"
        }
      ]
    },
    {
      "title": "Response Time P95",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application))",
          "legendFormat": "{{application}}"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_server_requests_seconds_count{status=~'5..'}[5m])) by (application) / sum(rate(http_server_requests_seconds_count[5m])) by (application)",
          "legendFormat": "{{application}}"
        }
      ]
    },
    {
      "title": "JVM Heap Memory",
      "type": "gauge",
      "targets": [
        {
          "expr": "jvm_memory_used_bytes{area='heap'} / jvm_memory_max_bytes{area='heap'}"
        }
      ]
    }
  ]
}
```

### Docker Compose 部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.47.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'

  grafana:
    image: grafana/grafana:10.1.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources

  alertmanager:
    image: prom/alertmanager:v0.26.0
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml

  node-exporter:
    image: prom/node-exporter:v1.6.1
    ports:
      - "9100:9100"

volumes:
  prometheus-data:
  grafana-data:
```

## 核心监控指标

| 类型 | 指标 | 说明 |
|------|------|------|
| RED | Rate | 请求速率 |
| RED | Errors | 错误率 |
| RED | Duration | 响应时间 |
| USE | Utilization | 资源使用率 |
| USE | Saturation | 饱和度 |
| USE | Errors | 错误数 |

## 最佳实践清单

- [ ] 配置 Spring Boot Actuator
- [ ] 暴露 Prometheus 指标端点
- [ ] 定义核心业务指标
- [ ] 配置健康检查探针
- [ ] 设置合理的告警规则
- [ ] 部署 Grafana Dashboard
- [ ] 配置告警通知渠道
- [ ] 监控数据保留策略
