# Logging 日志最佳实践

## 角色设定

你是一位精通企业级日志系统的专家，擅长日志框架配置、ELK 日志收集和日志分析。

## 提示词模板

### 日志配置

```
请帮我配置日志系统：
- 日志框架：[Logback/Log4j2]
- 输出格式：[文本/JSON]
- 日志级别：[描述不同包的级别]
- 收集方案：[ELK/Loki/云服务]
- 存储需求：[保留时长/大小限制]

请提供配置和最佳实践。
```

## 核心配置示例

### Logback 配置

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">

    <!-- 属性定义 -->
    <springProperty scope="context" name="APP_NAME" source="spring.application.name" defaultValue="app"/>
    <property name="LOG_PATH" value="${LOG_PATH:-./logs}"/>
    <property name="LOG_MAX_SIZE" value="100MB"/>
    <property name="LOG_MAX_HISTORY" value="30"/>
    <property name="LOG_TOTAL_SIZE" value="10GB"/>

    <!-- 日志格式 -->
    <property name="CONSOLE_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%-5level) [%thread] %cyan(%logger{36}) - %msg%n"/>
    <property name="FILE_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] [%X{traceId}] %logger{36} - %msg%n"/>

    <!-- JSON 格式 -->
    <property name="JSON_PATTERN">
        {"timestamp":"%d{yyyy-MM-dd'T'HH:mm:ss.SSSXXX}","level":"%level","logger":"%logger","thread":"%thread","traceId":"%X{traceId}","spanId":"%X{spanId}","message":"%msg","exception":"%ex{full}"}
    </property>

    <!-- 控制台输出 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONSOLE_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 滚动文件输出 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}.log</file>
        <encoder>
            <pattern>${FILE_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>${LOG_MAX_SIZE}</maxFileSize>
            <maxHistory>${LOG_MAX_HISTORY}</maxHistory>
            <totalSizeCap>${LOG_TOTAL_SIZE}</totalSizeCap>
        </rollingPolicy>
    </appender>

    <!-- JSON 格式输出 (用于 ELK) -->
    <appender name="JSON_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}-json.log</file>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <customFields>{"app":"${APP_NAME}","env":"${SPRING_PROFILES_ACTIVE:-dev}"}</customFields>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-json.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>${LOG_MAX_SIZE}</maxFileSize>
            <maxHistory>${LOG_MAX_HISTORY}</maxHistory>
            <totalSizeCap>${LOG_TOTAL_SIZE}</totalSizeCap>
        </rollingPolicy>
    </appender>

    <!-- 错误日志单独文件 -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}-error.log</file>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
        <encoder>
            <pattern>${FILE_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-error.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>${LOG_MAX_SIZE}</maxFileSize>
            <maxHistory>${LOG_MAX_HISTORY}</maxHistory>
        </rollingPolicy>
    </appender>

    <!-- 异步输出 -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <discardingThreshold>0</discardingThreshold>
        <queueSize>512</queueSize>
        <includeCallerData>true</includeCallerData>
        <appender-ref ref="FILE"/>
    </appender>

    <appender name="ASYNC_JSON" class="ch.qos.logback.classic.AsyncAppender">
        <discardingThreshold>0</discardingThreshold>
        <queueSize>512</queueSize>
        <appender-ref ref="JSON_FILE"/>
    </appender>

    <!-- 日志级别配置 -->
    <logger name="com.example" level="DEBUG"/>
    <logger name="org.springframework" level="INFO"/>
    <logger name="org.springframework.web" level="DEBUG"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
    <logger name="org.hibernate.type.descriptor.sql" level="TRACE"/>
    <logger name="com.zaxxer.hikari" level="INFO"/>
    <logger name="io.lettuce" level="INFO"/>
    <logger name="org.apache.kafka" level="WARN"/>

    <!-- 开发环境 -->
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="ASYNC_FILE"/>
        </root>
    </springProfile>

    <!-- 生产环境 -->
    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="ASYNC_FILE"/>
            <appender-ref ref="ASYNC_JSON"/>
            <appender-ref ref="ERROR_FILE"/>
        </root>
    </springProfile>

</configuration>
```

### Log4j2 配置

```xml
<!-- log4j2-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" monitorInterval="30">
    <Properties>
        <Property name="APP_NAME">${spring:spring.application.name:-app}</Property>
        <Property name="LOG_PATH">${sys:LOG_PATH:-./logs}</Property>
        <Property name="PATTERN">%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] [%X{traceId}] %logger{36} - %msg%n</Property>
    </Properties>

    <Appenders>
        <!-- 控制台 -->
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="${PATTERN}"/>
        </Console>

        <!-- 滚动文件 -->
        <RollingFile name="RollingFile" fileName="${LOG_PATH}/${APP_NAME}.log"
                     filePattern="${LOG_PATH}/${APP_NAME}.%d{yyyy-MM-dd}.%i.log.gz">
            <PatternLayout pattern="${PATTERN}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1"/>
                <SizeBasedTriggeringPolicy size="100MB"/>
            </Policies>
            <DefaultRolloverStrategy max="30">
                <Delete basePath="${LOG_PATH}" maxDepth="1">
                    <IfFileName glob="${APP_NAME}.*.log.gz"/>
                    <IfLastModified age="30d"/>
                </Delete>
            </DefaultRolloverStrategy>
        </RollingFile>

        <!-- JSON 格式 -->
        <RollingFile name="JsonFile" fileName="${LOG_PATH}/${APP_NAME}-json.log"
                     filePattern="${LOG_PATH}/${APP_NAME}-json.%d{yyyy-MM-dd}.%i.log.gz">
            <JsonLayout compact="true" eventEol="true" stacktraceAsString="true">
                <KeyValuePair key="app" value="${APP_NAME}"/>
            </JsonLayout>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1"/>
                <SizeBasedTriggeringPolicy size="100MB"/>
            </Policies>
        </RollingFile>

        <!-- 异步 -->
        <Async name="AsyncFile">
            <AppenderRef ref="RollingFile"/>
        </Async>
    </Appenders>

    <Loggers>
        <Logger name="com.example" level="debug"/>
        <Logger name="org.springframework" level="info"/>
        <Logger name="org.hibernate.SQL" level="debug"/>

        <Root level="info">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="AsyncFile"/>
        </Root>
    </Loggers>
</Configuration>
```

### 日志工具类

```java
@Slf4j
public class LogUtils {

    /**
     * 设置 MDC 上下文
     */
    public static void setContext(String traceId, String userId) {
        MDC.put("traceId", traceId);
        MDC.put("userId", userId);
    }

    /**
     * 清除 MDC 上下文
     */
    public static void clearContext() {
        MDC.clear();
    }

    /**
     * 记录方法执行时间
     */
    public static <T> T logExecutionTime(String methodName, Supplier<T> supplier) {
        long startTime = System.currentTimeMillis();
        try {
            T result = supplier.get();
            log.info("{} executed in {}ms", methodName, System.currentTimeMillis() - startTime);
            return result;
        } catch (Exception e) {
            log.error("{} failed after {}ms", methodName, System.currentTimeMillis() - startTime, e);
            throw e;
        }
    }

    /**
     * 脱敏日志
     */
    public static String maskSensitive(String value, int prefixLength, int suffixLength) {
        if (value == null || value.length() <= prefixLength + suffixLength) {
            return "***";
        }
        String prefix = value.substring(0, prefixLength);
        String suffix = value.substring(value.length() - suffixLength);
        return prefix + "***" + suffix;
    }
}
```

### 请求日志过滤器

```java
@Component
@Slf4j
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        String traceId = request.getHeader("X-Trace-ID");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString().replace("-", "");
        }
        MDC.put("traceId", traceId);

        long startTime = System.currentTimeMillis();

        // 包装请求和响应以便读取内容
        ContentCachingRequestWrapper requestWrapper = new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper(response);

        try {
            // 记录请求
            logRequest(requestWrapper);

            filterChain.doFilter(requestWrapper, responseWrapper);

            // 记录响应
            logResponse(responseWrapper, System.currentTimeMillis() - startTime);

        } finally {
            responseWrapper.copyBodyToResponse();
            MDC.clear();
        }
    }

    private void logRequest(ContentCachingRequestWrapper request) {
        log.info(">>> Request: {} {} from {}",
            request.getMethod(),
            request.getRequestURI(),
            getClientIP(request));

        if (log.isDebugEnabled()) {
            String body = getRequestBody(request);
            if (!body.isEmpty()) {
                log.debug(">>> Request Body: {}", body);
            }
        }
    }

    private void logResponse(ContentCachingResponseWrapper response, long duration) {
        log.info("<<< Response: {} in {}ms", response.getStatus(), duration);

        if (log.isDebugEnabled() && response.getContentSize() > 0) {
            String body = new String(response.getContentAsByteArray(), StandardCharsets.UTF_8);
            log.debug("<<< Response Body: {}", truncate(body, 1000));
        }
    }

    private String getClientIP(HttpServletRequest request) {
        String ip = request.getHeader("X-Forwarded-For");
        if (ip == null || ip.isEmpty()) {
            ip = request.getRemoteAddr();
        }
        return ip.split(",")[0].trim();
    }

    private String getRequestBody(ContentCachingRequestWrapper request) {
        byte[] content = request.getContentAsByteArray();
        return content.length > 0 ? new String(content, StandardCharsets.UTF_8) : "";
    }

    private String truncate(String str, int maxLength) {
        return str.length() > maxLength ? str.substring(0, maxLength) + "..." : str;
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getRequestURI();
        return path.startsWith("/actuator") || path.startsWith("/swagger");
    }
}
```

### AOP 方法日志

```java
@Aspect
@Component
@Slf4j
public class LoggingAspect {

    @Around("@annotation(com.example.annotation.Loggable)")
    public Object logMethod(ProceedingJoinPoint joinPoint) throws Throwable {
        String className = joinPoint.getTarget().getClass().getSimpleName();
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();

        log.info(">>> {}.{}() called with: {}", className, methodName, Arrays.toString(args));

        long startTime = System.currentTimeMillis();
        try {
            Object result = joinPoint.proceed();
            long duration = System.currentTimeMillis() - startTime;

            log.info("<<< {}.{}() returned: {} in {}ms",
                className, methodName, result, duration);
            return result;
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            log.error("!!! {}.{}() threw {} after {}ms",
                className, methodName, e.getClass().getSimpleName(), duration);
            throw e;
        }
    }

    @Around("execution(* com.example.service..*(..))")
    public Object logService(ProceedingJoinPoint joinPoint) throws Throwable {
        // 服务层日志
        return logMethod(joinPoint);
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {
}
```

### ELK 集成 - Filebeat 配置

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app/*-json.log
    json.keys_under_root: true
    json.add_error_key: true
    json.message_key: message

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "app-logs-%{+yyyy.MM.dd}"

setup.template.name: "app-logs"
setup.template.pattern: "app-logs-*"
setup.ilm.enabled: false
```

### Docker 日志驱动

```yaml
# docker-compose.yml
services:
  app:
    image: app:latest
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "3"
        labels: "app,env"
        tag: "{{.Name}}/{{.ID}}"

  # 或使用 fluentd
  app-fluentd:
    image: app:latest
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: app.{{.Name}}
        fluentd-async: "true"
```

### 日志级别动态调整

```java
@RestController
@RequestMapping("/api/admin/logging")
@PreAuthorize("hasRole('ADMIN')")
public class LoggingController {

    @PostMapping("/level")
    public ResponseEntity<String> setLogLevel(
            @RequestParam String loggerName,
            @RequestParam String level) {

        LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();
        ch.qos.logback.classic.Logger logger = loggerContext.getLogger(loggerName);

        if (logger != null) {
            logger.setLevel(Level.toLevel(level));
            return ResponseEntity.ok("Log level updated: " + loggerName + " -> " + level);
        }
        return ResponseEntity.notFound().build();
    }

    @GetMapping("/level")
    public ResponseEntity<Map<String, String>> getLogLevels() {
        LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();

        Map<String, String> levels = new HashMap<>();
        for (ch.qos.logback.classic.Logger logger : loggerContext.getLoggerList()) {
            if (logger.getLevel() != null) {
                levels.put(logger.getName(), logger.getLevel().toString());
            }
        }
        return ResponseEntity.ok(levels);
    }
}
```

## 日志级别使用规范

| 级别 | 使用场景 |
|------|----------|
| ERROR | 系统错误、异常、需要立即处理 |
| WARN | 潜在问题、可恢复的错误 |
| INFO | 重要业务流程、状态变更 |
| DEBUG | 开发调试、详细执行过程 |
| TRACE | 最详细日志、数据追踪 |

## 最佳实践清单

- [ ] 使用 SLF4J + Logback/Log4j2
- [ ] 配置异步日志输出
- [ ] JSON 格式便于收集分析
- [ ] 设置合理的日志滚动策略
- [ ] MDC 传递链路追踪信息
- [ ] 敏感信息脱敏处理
- [ ] 区分环境日志级别
- [ ] 接入 ELK/Loki 日志系统
