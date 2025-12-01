# Spring Boot 开发最佳实践

## 角色设定

你是一位精通 Spring Boot 3.x 的后端开发专家，擅长自动配置、Web 开发、数据访问和微服务架构。

## 提示词模板

### 项目搭建

```
请帮我搭建 Spring Boot 项目：
- Spring Boot 版本：[2.7/3.x]
- Java 版本：[17/21]
- 构建工具：[Maven/Gradle]
- 需要的功能：
  - [ ] Web API
  - [ ] 数据库访问
  - [ ] 缓存
  - [ ] 安全认证
  - [ ] 消息队列
```

### 功能开发

```
请帮我实现以下 Spring Boot 功能：
- 功能描述：[描述功能]
- 涉及层次：[Controller/Service/Repository]
- 数据库：[MySQL/PostgreSQL/MongoDB]
- 是否需要事务：[是/否]
```

## 核心代码示例

### 项目结构

```
src/main/java/com/example/
├── Application.java
├── config/              # 配置类
│   ├── SecurityConfig.java
│   ├── RedisConfig.java
│   └── SwaggerConfig.java
├── controller/          # 控制器
│   └── UserController.java
├── service/             # 业务服务
│   ├── UserService.java
│   └── impl/
│       └── UserServiceImpl.java
├── repository/          # 数据访问
│   └── UserRepository.java
├── entity/              # 实体类
│   └── User.java
├── dto/                 # 数据传输对象
│   ├── request/
│   └── response/
├── exception/           # 异常处理
│   ├── BusinessException.java
│   └── GlobalExceptionHandler.java
├── common/              # 公共组件
│   ├── Result.java
│   └── PageResult.java
└── util/                # 工具类
```

### 统一响应封装

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Result<T> {
    private int code;
    private String message;
    private T data;
    private long timestamp;

    public static <T> Result<T> success(T data) {
        return new Result<>(200, "success", data, System.currentTimeMillis());
    }

    public static <T> Result<T> success() {
        return success(null);
    }

    public static <T> Result<T> error(int code, String message) {
        return new Result<>(code, message, null, System.currentTimeMillis());
    }

    public static <T> Result<T> error(ErrorCode errorCode) {
        return new Result<>(errorCode.getCode(), errorCode.getMessage(), null, System.currentTimeMillis());
    }
}

// 分页响应
@Data
public class PageResult<T> {
    private List<T> list;
    private long total;
    private int pageNum;
    private int pageSize;
    private int pages;

    public static <T> PageResult<T> of(Page<T> page) {
        PageResult<T> result = new PageResult<>();
        result.setList(page.getContent());
        result.setTotal(page.getTotalElements());
        result.setPageNum(page.getNumber() + 1);
        result.setPageSize(page.getSize());
        result.setPages(page.getTotalPages());
        return result;
    }
}
```

### Controller 层

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Tag(name = "用户管理", description = "用户相关接口")
public class UserController {

    private final UserService userService;

    @GetMapping
    @Operation(summary = "分页查询用户")
    public Result<PageResult<UserVO>> list(
            @RequestParam(defaultValue = "1") int pageNum,
            @RequestParam(defaultValue = "10") int pageSize,
            @RequestParam(required = false) String keyword) {
        PageResult<UserVO> result = userService.findByPage(pageNum, pageSize, keyword);
        return Result.success(result);
    }

    @GetMapping("/{id}")
    @Operation(summary = "根据ID查询用户")
    public Result<UserVO> getById(@PathVariable Long id) {
        UserVO user = userService.findById(id);
        return Result.success(user);
    }

    @PostMapping
    @Operation(summary = "创建用户")
    public Result<UserVO> create(@RequestBody @Valid CreateUserRequest request) {
        UserVO user = userService.create(request);
        return Result.success(user);
    }

    @PutMapping("/{id}")
    @Operation(summary = "更新用户")
    public Result<UserVO> update(
            @PathVariable Long id,
            @RequestBody @Valid UpdateUserRequest request) {
        UserVO user = userService.update(id, request);
        return Result.success(user);
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "删除用户")
    public Result<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return Result.success();
    }
}
```

### Service 层

```java
public interface UserService {
    PageResult<UserVO> findByPage(int pageNum, int pageSize, String keyword);
    UserVO findById(Long id);
    UserVO create(CreateUserRequest request);
    UserVO update(Long id, UpdateUserRequest request);
    void delete(Long id);
}

@Service
@RequiredArgsConstructor
@Slf4j
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final UserMapper userMapper;

    @Override
    @Transactional(readOnly = true)
    public PageResult<UserVO> findByPage(int pageNum, int pageSize, String keyword) {
        Pageable pageable = PageRequest.of(pageNum - 1, pageSize, Sort.by("createdAt").descending());

        Page<User> page;
        if (StringUtils.hasText(keyword)) {
            page = userRepository.findByNameContainingOrEmailContaining(keyword, keyword, pageable);
        } else {
            page = userRepository.findAll(pageable);
        }

        return PageResult.of(page.map(userMapper::toVO));
    }

    @Override
    @Transactional(readOnly = true)
    public UserVO findById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND, id));
        return userMapper.toVO(user);
    }

    @Override
    @Transactional
    public UserVO create(CreateUserRequest request) {
        // 检查邮箱是否已存在
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new BusinessException(ErrorCode.EMAIL_ALREADY_EXISTS, request.getEmail());
        }

        User user = userMapper.toEntity(request);
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        user.setStatus(UserStatus.ACTIVE);

        user = userRepository.save(user);
        log.info("Created user: {}", user.getId());

        return userMapper.toVO(user);
    }

    @Override
    @Transactional
    public UserVO update(Long id, UpdateUserRequest request) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND, id));

        userMapper.updateEntity(request, user);
        user = userRepository.save(user);

        return userMapper.toVO(user);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        if (!userRepository.existsById(id)) {
            throw new BusinessException(ErrorCode.USER_NOT_FOUND, id);
        }
        userRepository.deleteById(id);
        log.info("Deleted user: {}", id);
    }
}
```

### Repository 层

```java
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

    boolean existsByEmail(String email);

    Page<User> findByNameContainingOrEmailContaining(
        String name, String email, Pageable pageable);

    @Query("SELECT u FROM User u WHERE u.status = :status AND u.createdAt > :date")
    List<User> findActiveUsersAfter(
        @Param("status") UserStatus status,
        @Param("date") LocalDateTime date);

    @Modifying
    @Query("UPDATE User u SET u.status = :status WHERE u.id IN :ids")
    int updateStatusByIds(
        @Param("status") UserStatus status,
        @Param("ids") List<Long> ids);
}
```

### 配置文件

```yaml
# application.yml
spring:
  application:
    name: user-service

  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

  datasource:
    url: jdbc:mysql://${DB_HOST:localhost}:3306/${DB_NAME:user_db}?useSSL=false&serverTimezone=Asia/Shanghai
    username: ${DB_USER:root}
    password: ${DB_PASSWORD:root}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 20000

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.MySQL8Dialect

  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
    password: ${REDIS_PASSWORD:}
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0

  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
    serialization:
      write-dates-as-timestamps: false

server:
  port: 8080
  servlet:
    context-path: /api

logging:
  level:
    root: INFO
    com.example: DEBUG
    org.springframework.web: INFO
    org.hibernate.SQL: DEBUG

# 自定义配置
app:
  jwt:
    secret: ${JWT_SECRET:your-secret-key}
    expiration: 86400000
  upload:
    path: /data/uploads
    max-size: 10MB
```

### 全局异常处理

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<Result<?>> handleBusinessException(BusinessException e) {
        log.warn("Business exception: {}", e.getMessage());
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(Result.error(e.getErrorCode().getCode(), e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Result<?>> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(Result.error(400, message));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Result<?>> handleException(Exception e) {
        log.error("Unexpected exception", e);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(Result.error(500, "服务器内部错误"));
    }
}
```

### 自定义配置属性

```java
@ConfigurationProperties(prefix = "app")
@Validated
public record AppProperties(
    @NotNull Jwt jwt,
    @NotNull Upload upload
) {
    public record Jwt(
        @NotBlank String secret,
        @Positive long expiration
    ) {}

    public record Upload(
        @NotBlank String path,
        @NotBlank String maxSize
    ) {}
}

// 启用配置
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 最佳实践清单

- [ ] 使用分层架构组织代码
- [ ] 统一响应格式封装
- [ ] 全局异常处理
- [ ] 参数校验 (@Valid)
- [ ] 使用配置属性类
- [ ] 合理使用事务注解
- [ ] 编写单元测试和集成测试
- [ ] 配置健康检查和监控
