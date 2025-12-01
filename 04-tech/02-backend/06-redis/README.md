# Redis 缓存最佳实践

## 角色设定

你是一位精通 Redis 7.x 的缓存专家，擅长数据结构选型、缓存策略设计、分布式锁和高可用架构。

## 提示词模板

### 缓存方案设计

```
请帮我设计 Redis 缓存方案：
- 业务场景：[描述业务]
- 数据特点：[热点数据/冷数据/实时性要求]
- 读写比例：[读多写少/读写均衡]
- 数据量：[预估数据量和内存]
- 一致性要求：[强一致/最终一致]

请提供：
1. Key 设计
2. 数据结构选型
3. 过期策略
4. 缓存更新策略
```

### 问题排查

```
请帮我排查 Redis 问题：
- 问题描述：[缓存穿透/击穿/雪崩/热点Key/大Key]
- 当前配置：[描述配置]
- 监控数据：[描述监控指标]

请分析原因并提供解决方案。
```

## 核心代码示例

### Spring Boot Redis 配置

```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // Key 序列化
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());

        // Value 序列化 - JSON
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.activateDefaultTyping(
            mapper.getPolymorphicTypeValidator(),
            ObjectMapper.DefaultTyping.NON_FINAL
        );
        serializer.setObjectMapper(mapper);

        template.setValueSerializer(serializer);
        template.setHashValueSerializer(serializer);

        template.afterPropertiesSet();
        return template;
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("user", config.entryTtl(Duration.ofHours(1)))
            .withCacheConfiguration("product", config.entryTtl(Duration.ofMinutes(10)))
            .build();
    }
}
```

### 缓存服务封装

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class CacheService {

    private final RedisTemplate<String, Object> redisTemplate;

    // 设置缓存
    public void set(String key, Object value, long timeout, TimeUnit unit) {
        redisTemplate.opsForValue().set(key, value, timeout, unit);
    }

    // 获取缓存
    public <T> T get(String key, Class<T> clazz) {
        Object value = redisTemplate.opsForValue().get(key);
        return clazz.cast(value);
    }

    // 删除缓存
    public Boolean delete(String key) {
        return redisTemplate.delete(key);
    }

    // 批量删除
    public Long deleteByPattern(String pattern) {
        Set<String> keys = redisTemplate.keys(pattern);
        if (keys != null && !keys.isEmpty()) {
            return redisTemplate.delete(keys);
        }
        return 0L;
    }

    // 设置过期时间
    public Boolean expire(String key, long timeout, TimeUnit unit) {
        return redisTemplate.expire(key, timeout, unit);
    }

    // 判断是否存在
    public Boolean hasKey(String key) {
        return redisTemplate.hasKey(key);
    }

    // 自增
    public Long increment(String key, long delta) {
        return redisTemplate.opsForValue().increment(key, delta);
    }

    // Hash 操作
    public void hSet(String key, String field, Object value) {
        redisTemplate.opsForHash().put(key, field, value);
    }

    public Object hGet(String key, String field) {
        return redisTemplate.opsForHash().get(key, field);
    }

    public Map<Object, Object> hGetAll(String key) {
        return redisTemplate.opsForHash().entries(key);
    }

    // List 操作
    public Long lPush(String key, Object value) {
        return redisTemplate.opsForList().leftPush(key, value);
    }

    public Object rPop(String key) {
        return redisTemplate.opsForList().rightPop(key);
    }

    // Set 操作
    public Long sAdd(String key, Object... values) {
        return redisTemplate.opsForSet().add(key, values);
    }

    public Set<Object> sMembers(String key) {
        return redisTemplate.opsForSet().members(key);
    }

    // ZSet 操作
    public Boolean zAdd(String key, Object value, double score) {
        return redisTemplate.opsForZSet().add(key, value, score);
    }

    public Set<Object> zRange(String key, long start, long end) {
        return redisTemplate.opsForZSet().range(key, start, end);
    }
}
```

### 分布式锁

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class RedisLockService {

    private final StringRedisTemplate redisTemplate;
    private static final String LOCK_PREFIX = "lock:";

    /**
     * 尝试获取分布式锁
     */
    public boolean tryLock(String lockKey, String requestId, long expireTime, TimeUnit unit) {
        String key = LOCK_PREFIX + lockKey;
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(key, requestId, expireTime, unit);
        return Boolean.TRUE.equals(result);
    }

    /**
     * 释放分布式锁 (Lua 脚本保证原子性)
     */
    public boolean unlock(String lockKey, String requestId) {
        String key = LOCK_PREFIX + lockKey;
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;

        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(key),
            requestId
        );
        return Long.valueOf(1L).equals(result);
    }

    /**
     * 使用锁执行业务
     */
    public <T> T executeWithLock(String lockKey, long waitTime, long leaseTime,
                                  TimeUnit unit, Supplier<T> supplier) {
        String requestId = UUID.randomUUID().toString();
        long startTime = System.currentTimeMillis();
        long waitTimeMillis = unit.toMillis(waitTime);

        try {
            // 自旋等待获取锁
            while (System.currentTimeMillis() - startTime < waitTimeMillis) {
                if (tryLock(lockKey, requestId, leaseTime, unit)) {
                    try {
                        return supplier.get();
                    } finally {
                        unlock(lockKey, requestId);
                    }
                }
                Thread.sleep(50);
            }
            throw new BusinessException(ErrorCode.LOCK_ACQUIRE_FAILED);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new BusinessException(ErrorCode.LOCK_ACQUIRE_FAILED);
        }
    }
}

// 使用示例
@Service
@RequiredArgsConstructor
public class OrderService {

    private final RedisLockService lockService;

    public void createOrder(CreateOrderRequest request) {
        String lockKey = "order:create:" + request.getUserId();

        lockService.executeWithLock(lockKey, 5, 30, TimeUnit.SECONDS, () -> {
            // 创建订单的业务逻辑
            doCreateOrder(request);
            return null;
        });
    }
}
```

### 缓存模式

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class UserCacheService {

    private final UserRepository userRepository;
    private final CacheService cacheService;

    private static final String USER_KEY_PREFIX = "user:";
    private static final long USER_CACHE_TTL = 30; // 分钟

    /**
     * Cache-Aside 模式 (旁路缓存)
     */
    public User findById(Long id) {
        String key = USER_KEY_PREFIX + id;

        // 1. 先查缓存
        User user = cacheService.get(key, User.class);
        if (user != null) {
            return user;
        }

        // 2. 缓存未命中，查数据库
        user = userRepository.findById(id).orElse(null);

        // 3. 写入缓存 (空值也缓存，防止穿透)
        if (user != null) {
            cacheService.set(key, user, USER_CACHE_TTL, TimeUnit.MINUTES);
        } else {
            // 缓存空值，较短过期时间
            cacheService.set(key, new NullValue(), 5, TimeUnit.MINUTES);
        }

        return user;
    }

    /**
     * 更新时删除缓存 (推荐)
     */
    @Transactional
    public void update(User user) {
        // 1. 更新数据库
        userRepository.save(user);

        // 2. 删除缓存
        String key = USER_KEY_PREFIX + user.getId();
        cacheService.delete(key);
    }

    /**
     * 缓存预热
     */
    @PostConstruct
    public void warmUp() {
        List<User> hotUsers = userRepository.findHotUsers(100);
        for (User user : hotUsers) {
            String key = USER_KEY_PREFIX + user.getId();
            cacheService.set(key, user, USER_CACHE_TTL, TimeUnit.MINUTES);
        }
        log.info("Cache warm-up completed, loaded {} users", hotUsers.size());
    }
}
```

### 缓存问题解决方案

```java
@Service
@RequiredArgsConstructor
public class CacheSolutionService {

    private final CacheService cacheService;
    private final BloomFilter<String> bloomFilter;

    /**
     * 防止缓存穿透 - 布隆过滤器
     */
    public Product findProductWithBloom(Long productId) {
        String key = "product:" + productId;

        // 布隆过滤器判断是否存在
        if (!bloomFilter.mightContain(key)) {
            return null; // 一定不存在
        }

        // 查缓存
        Product product = cacheService.get(key, Product.class);
        if (product != null) {
            return product;
        }

        // 查数据库
        product = productRepository.findById(productId).orElse(null);
        if (product != null) {
            cacheService.set(key, product, 30, TimeUnit.MINUTES);
        }
        return product;
    }

    /**
     * 防止缓存击穿 - 互斥锁
     */
    public Product findProductWithMutex(Long productId) {
        String key = "product:" + productId;
        String lockKey = "lock:product:" + productId;

        Product product = cacheService.get(key, Product.class);
        if (product != null) {
            return product;
        }

        // 获取分布式锁
        String requestId = UUID.randomUUID().toString();
        if (lockService.tryLock(lockKey, requestId, 10, TimeUnit.SECONDS)) {
            try {
                // 双重检查
                product = cacheService.get(key, Product.class);
                if (product != null) {
                    return product;
                }

                // 查数据库
                product = productRepository.findById(productId).orElse(null);
                if (product != null) {
                    cacheService.set(key, product, 30, TimeUnit.MINUTES);
                }
                return product;
            } finally {
                lockService.unlock(lockKey, requestId);
            }
        } else {
            // 等待后重试
            Thread.sleep(100);
            return findProductWithMutex(productId);
        }
    }

    /**
     * 防止缓存雪崩 - 随机过期时间
     */
    public void setWithRandomExpire(String key, Object value, long baseTtl) {
        // 随机增加 0-10 分钟
        long randomTtl = baseTtl + ThreadLocalRandom.current().nextLong(10);
        cacheService.set(key, value, randomTtl, TimeUnit.MINUTES);
    }
}
```

### Key 设计规范

| 场景 | Key 格式 | 示例 |
|------|----------|------|
| 用户信息 | user:{userId} | user:12345 |
| 订单详情 | order:{orderId} | order:20240301001 |
| 用户订单列表 | user:{userId}:orders | user:12345:orders |
| 商品库存 | product:{productId}:stock | product:100:stock |
| 分布式锁 | lock:{业务}:{标识} | lock:order:12345 |
| 计数器 | counter:{业务}:{日期} | counter:login:20240301 |

## 最佳实践清单

- [ ] Key 命名规范，使用冒号分隔
- [ ] 设置合理的过期时间
- [ ] 避免大 Key (String < 10KB, 集合 < 5000)
- [ ] 使用 Pipeline 批量操作
- [ ] 缓存空值防止穿透
- [ ] 使用分布式锁防止击穿
- [ ] 随机过期时间防止雪崩
- [ ] 热点 Key 使用本地缓存
