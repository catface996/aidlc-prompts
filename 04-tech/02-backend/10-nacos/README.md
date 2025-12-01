# Nacos 注册配置中心最佳实践

## 角色设定

你是一位精通 Nacos 的微服务基础设施专家，擅长服务注册发现、配置管理和集群部署。

## 提示词模板

### Nacos 配置

```
请帮我配置 Nacos：
- 使用场景：[注册中心/配置中心/两者]
- 部署模式：[单机/集群]
- 存储方式：[内置/MySQL]
- 命名空间规划：[描述环境划分]
```

## 核心代码示例

### Spring Cloud 集成配置

```yaml
# bootstrap.yml
spring:
  application:
    name: order-service
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
  cloud:
    nacos:
      # 服务发现
      discovery:
        server-addr: ${NACOS_SERVER:localhost:8848}
        namespace: ${NACOS_NAMESPACE:dev}
        group: DEFAULT_GROUP
        username: ${NACOS_USERNAME:nacos}
        password: ${NACOS_PASSWORD:nacos}
        metadata:
          version: v1
          env: ${SPRING_PROFILES_ACTIVE:dev}
      # 配置中心
      config:
        server-addr: ${NACOS_SERVER:localhost:8848}
        namespace: ${NACOS_NAMESPACE:dev}
        group: DEFAULT_GROUP
        file-extension: yaml
        # 共享配置
        shared-configs:
          - data-id: common.yaml
            group: DEFAULT_GROUP
            refresh: true
          - data-id: redis.yaml
            group: DEFAULT_GROUP
            refresh: true
        # 扩展配置
        extension-configs:
          - data-id: ${spring.application.name}-${spring.profiles.active}.yaml
            group: DEFAULT_GROUP
            refresh: true
```

### 动态配置刷新

```java
@RestController
@RefreshScope  // 支持配置动态刷新
@RequiredArgsConstructor
public class ConfigController {

    @Value("${app.feature.enabled:false}")
    private boolean featureEnabled;

    @Value("${app.rate-limit:100}")
    private int rateLimit;

    @GetMapping("/config")
    public Map<String, Object> getConfig() {
        return Map.of(
            "featureEnabled", featureEnabled,
            "rateLimit", rateLimit
        );
    }
}

// 配置属性类
@Data
@Component
@ConfigurationProperties(prefix = "app")
@RefreshScope
public class AppProperties {

    private Feature feature = new Feature();
    private RateLimit rateLimit = new RateLimit();

    @Data
    public static class Feature {
        private boolean enabled = false;
        private List<String> whitelist = new ArrayList<>();
    }

    @Data
    public static class RateLimit {
        private int qps = 100;
        private int burst = 200;
    }
}
```

### 配置监听

```java
@Component
@Slf4j
public class NacosConfigListener {

    @NacosConfigListener(dataId = "order-service.yaml", groupId = "DEFAULT_GROUP")
    public void onConfigChange(String config) {
        log.info("Config changed: {}", config);
        // 处理配置变更
    }
}

// 或使用 Spring Cloud 方式
@Component
@Slf4j
public class ConfigChangeListener implements ApplicationListener<RefreshScopeRefreshedEvent> {

    @Override
    public void onApplicationEvent(RefreshScopeRefreshedEvent event) {
        log.info("Configuration refreshed");
        // 配置刷新后的处理逻辑
    }
}

// 监听环境变化
@Component
@Slf4j
public class EnvironmentChangeListener implements ApplicationListener<EnvironmentChangeEvent> {

    @Override
    public void onApplicationEvent(EnvironmentChangeEvent event) {
        log.info("Changed keys: {}", event.getKeys());
        for (String key : event.getKeys()) {
            // 处理特定配置变更
        }
    }
}
```

### 服务发现使用

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ServiceDiscoveryService {

    private final DiscoveryClient discoveryClient;
    private final NacosServiceManager nacosServiceManager;

    /**
     * 获取服务实例列表
     */
    public List<ServiceInstance> getInstances(String serviceId) {
        return discoveryClient.getInstances(serviceId);
    }

    /**
     * 获取健康实例
     */
    public List<Instance> getHealthyInstances(String serviceName) throws NacosException {
        NamingService namingService = nacosServiceManager.getNamingService();
        return namingService.selectInstances(serviceName, true);
    }

    /**
     * 监听服务变化
     */
    public void subscribeService(String serviceName) throws NacosException {
        NamingService namingService = nacosServiceManager.getNamingService();
        namingService.subscribe(serviceName, event -> {
            if (event instanceof NamingEvent namingEvent) {
                log.info("Service {} instances changed: {}",
                    serviceName, namingEvent.getInstances().size());
            }
        });
    }

    /**
     * 按权重选择实例
     */
    public Instance selectOneHealthyInstance(String serviceName) throws NacosException {
        NamingService namingService = nacosServiceManager.getNamingService();
        return namingService.selectOneHealthyInstance(serviceName);
    }
}
```

### 手动注册/注销服务

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ServiceRegistryService {

    private final NacosServiceManager nacosServiceManager;

    @Value("${spring.application.name}")
    private String serviceName;

    @Value("${server.port}")
    private int port;

    /**
     * 注册服务
     */
    public void register() throws NacosException {
        NamingService namingService = nacosServiceManager.getNamingService();

        Instance instance = new Instance();
        instance.setIp(getLocalIp());
        instance.setPort(port);
        instance.setWeight(1.0);
        instance.setHealthy(true);
        instance.setMetadata(Map.of(
            "version", "v1",
            "env", "prod"
        ));

        namingService.registerInstance(serviceName, instance);
        log.info("Service registered: {}:{}", instance.getIp(), instance.getPort());
    }

    /**
     * 注销服务
     */
    public void deregister() throws NacosException {
        NamingService namingService = nacosServiceManager.getNamingService();
        namingService.deregisterInstance(serviceName, getLocalIp(), port);
        log.info("Service deregistered");
    }

    /**
     * 更新实例元数据
     */
    public void updateMetadata(Map<String, String> metadata) throws NacosException {
        NamingService namingService = nacosServiceManager.getNamingService();
        Instance instance = new Instance();
        instance.setIp(getLocalIp());
        instance.setPort(port);
        instance.setMetadata(metadata);
        namingService.registerInstance(serviceName, instance);
    }
}
```

### Nacos 配置管理 API

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class NacosConfigService {

    private final NacosConfigManager nacosConfigManager;

    /**
     * 获取配置
     */
    public String getConfig(String dataId, String group) throws NacosException {
        ConfigService configService = nacosConfigManager.getConfigService();
        return configService.getConfig(dataId, group, 5000);
    }

    /**
     * 发布配置
     */
    public boolean publishConfig(String dataId, String group, String content) throws NacosException {
        ConfigService configService = nacosConfigManager.getConfigService();
        return configService.publishConfig(dataId, group, content, "yaml");
    }

    /**
     * 删除配置
     */
    public boolean removeConfig(String dataId, String group) throws NacosException {
        ConfigService configService = nacosConfigManager.getConfigService();
        return configService.removeConfig(dataId, group);
    }

    /**
     * 监听配置变化
     */
    public void addListener(String dataId, String group) throws NacosException {
        ConfigService configService = nacosConfigManager.getConfigService();
        configService.addListener(dataId, group, new Listener() {
            @Override
            public Executor getExecutor() {
                return null;
            }

            @Override
            public void receiveConfigInfo(String configInfo) {
                log.info("Config changed - dataId={}, group={}, content={}",
                    dataId, group, configInfo);
            }
        });
    }
}
```

### 命名空间规划

```
Nacos 命名空间规划:
├── dev (开发环境)
│   ├── common.yaml          # 公共配置
│   ├── redis.yaml           # Redis 配置
│   ├── order-service.yaml   # 订单服务配置
│   └── user-service.yaml    # 用户服务配置
├── test (测试环境)
│   └── ...
├── staging (预发环境)
│   └── ...
└── prod (生产环境)
    └── ...
```

### 配置文件示例

```yaml
# common.yaml (公共配置)
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai

logging:
  level:
    root: INFO

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics

---
# order-service.yaml
app:
  feature:
    enabled: true
    whitelist:
      - user001
      - user002
  rate-limit:
    qps: 1000
    burst: 2000

order:
  expire-minutes: 30
  max-items: 100
```

## 最佳实践清单

- [ ] 按环境划分命名空间
- [ ] 公共配置抽取到 shared-configs
- [ ] 敏感配置使用加密
- [ ] 配置类使用 @RefreshScope
- [ ] 监听配置变化做相应处理
- [ ] 服务实例设置合理的元数据
- [ ] 集群部署保证高可用
- [ ] 配置变更要有审计记录
