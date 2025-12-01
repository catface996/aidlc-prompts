# MyBatis 持久层最佳实践

## 角色设定

你是一位精通 MyBatis 3.x 和 MyBatis-Plus 的持久层专家，擅长 SQL 映射、动态 SQL、性能优化和代码生成。

## 提示词模板

### SQL 映射

```
请帮我编写 MyBatis 映射：
- 表结构：[描述表结构]
- 查询需求：[描述查询条件]
- 结果映射：[单表/关联查询/嵌套结果]
- 是否分页：[是/否]

请提供 Mapper 接口和 XML 配置。
```

### 动态 SQL

```
请帮我编写动态 SQL：
- 查询条件：[列出可选条件]
- 排序要求：[描述排序]
- 特殊需求：[批量操作/条件更新/...]
```

## 核心代码示例

### MyBatis-Plus 配置

```java
@Configuration
@MapperScan("com.example.mapper")
public class MyBatisConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));

        // 乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());

        // 防全表更新删除插件
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());

        return interceptor;
    }

    @Bean
    public MetaObjectHandler metaObjectHandler() {
        return new MyMetaObjectHandler();
    }
}

// 自动填充处理器
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createdAt", LocalDateTime::now, LocalDateTime.class);
        this.strictInsertFill(metaObject, "updatedAt", LocalDateTime::now, LocalDateTime.class);
        this.strictInsertFill(metaObject, "deleted", () -> 0, Integer.class);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updatedAt", LocalDateTime::now, LocalDateTime.class);
    }
}
```

### 实体类

```java
@Data
@TableName("user")
public class User {

    @TableId(type = IdType.ASSIGN_ID)
    private Long id;

    private String username;

    private String email;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createdAt;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updatedAt;

    @Version
    private Integer version;

    @TableLogic
    @TableField(fill = FieldFill.INSERT)
    private Integer deleted;
}

@Data
@TableName("`order`")
public class Order {

    @TableId(type = IdType.ASSIGN_ID)
    private Long id;

    private String orderNo;

    private Long userId;

    private BigDecimal totalAmount;

    @TableField(typeHandler = JacksonTypeHandler.class)
    private OrderExtra extra; // JSON 字段

    private OrderStatus status;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createdAt;
}
```

### Mapper 接口

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {

    /**
     * 自定义查询 - 注解方式
     */
    @Select("SELECT * FROM user WHERE email = #{email} AND deleted = 0")
    User findByEmail(@Param("email") String email);

    /**
     * 关联查询 - XML 方式
     */
    UserVO findUserWithOrders(@Param("userId") Long userId);

    /**
     * 批量插入
     */
    int batchInsert(@Param("list") List<User> users);

    /**
     * 动态查询
     */
    List<User> selectByCondition(@Param("query") UserQuery query);
}
```

### XML 映射文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.UserMapper">

    <!-- 结果映射 -->
    <resultMap id="UserWithOrdersMap" type="com.example.vo.UserVO">
        <id property="id" column="id"/>
        <result property="username" column="username"/>
        <result property="email" column="email"/>
        <result property="createdAt" column="created_at"/>
        <collection property="orders" ofType="com.example.vo.OrderVO">
            <id property="id" column="order_id"/>
            <result property="orderNo" column="order_no"/>
            <result property="totalAmount" column="total_amount"/>
            <result property="status" column="status"/>
            <result property="createdAt" column="order_created_at"/>
        </collection>
    </resultMap>

    <!-- 关联查询 -->
    <select id="findUserWithOrders" resultMap="UserWithOrdersMap">
        SELECT
            u.id,
            u.username,
            u.email,
            u.created_at,
            o.id as order_id,
            o.order_no,
            o.total_amount,
            o.status,
            o.created_at as order_created_at
        FROM user u
        LEFT JOIN `order` o ON u.id = o.user_id AND o.deleted = 0
        WHERE u.id = #{userId} AND u.deleted = 0
        ORDER BY o.created_at DESC
    </select>

    <!-- 批量插入 -->
    <insert id="batchInsert" parameterType="list">
        INSERT INTO user (id, username, email, created_at, updated_at, deleted)
        VALUES
        <foreach collection="list" item="item" separator=",">
            (#{item.id}, #{item.username}, #{item.email},
             #{item.createdAt}, #{item.updatedAt}, #{item.deleted})
        </foreach>
    </insert>

    <!-- 动态查询 -->
    <select id="selectByCondition" resultType="com.example.entity.User">
        SELECT * FROM user
        <where>
            deleted = 0
            <if test="query.username != null and query.username != ''">
                AND username LIKE CONCAT('%', #{query.username}, '%')
            </if>
            <if test="query.email != null and query.email != ''">
                AND email = #{query.email}
            </if>
            <if test="query.status != null">
                AND status = #{query.status}
            </if>
            <if test="query.startTime != null">
                AND created_at >= #{query.startTime}
            </if>
            <if test="query.endTime != null">
                AND created_at &lt;= #{query.endTime}
            </if>
            <if test="query.ids != null and query.ids.size() > 0">
                AND id IN
                <foreach collection="query.ids" item="id" open="(" separator="," close=")">
                    #{id}
                </foreach>
            </if>
        </where>
        <choose>
            <when test="query.orderBy == 'created_at_desc'">
                ORDER BY created_at DESC
            </when>
            <when test="query.orderBy == 'created_at_asc'">
                ORDER BY created_at ASC
            </when>
            <otherwise>
                ORDER BY id DESC
            </otherwise>
        </choose>
    </select>

    <!-- 动态更新 -->
    <update id="updateSelective">
        UPDATE user
        <set>
            <if test="username != null">username = #{username},</if>
            <if test="email != null">email = #{email},</if>
            <if test="status != null">status = #{status},</if>
            updated_at = NOW()
        </set>
        WHERE id = #{id} AND deleted = 0
    </update>

</mapper>
```

### Service 层使用

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserMapper userMapper;

    /**
     * MyBatis-Plus Lambda 查询
     */
    public List<User> findByCondition(UserQuery query) {
        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<User>()
            .like(StringUtils.hasText(query.getUsername()), User::getUsername, query.getUsername())
            .eq(StringUtils.hasText(query.getEmail()), User::getEmail, query.getEmail())
            .eq(query.getStatus() != null, User::getStatus, query.getStatus())
            .ge(query.getStartTime() != null, User::getCreatedAt, query.getStartTime())
            .le(query.getEndTime() != null, User::getCreatedAt, query.getEndTime())
            .in(CollectionUtils.isNotEmpty(query.getIds()), User::getId, query.getIds())
            .orderByDesc(User::getCreatedAt);

        return userMapper.selectList(wrapper);
    }

    /**
     * 分页查询
     */
    public IPage<User> findByPage(int pageNum, int pageSize, UserQuery query) {
        Page<User> page = new Page<>(pageNum, pageSize);

        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<User>()
            .like(StringUtils.hasText(query.getKeyword()), User::getUsername, query.getKeyword())
            .orderByDesc(User::getCreatedAt);

        return userMapper.selectPage(page, wrapper);
    }

    /**
     * 链式更新
     */
    public boolean updateStatus(Long id, Integer status) {
        return new LambdaUpdateChainWrapper<>(userMapper)
            .eq(User::getId, id)
            .set(User::getStatus, status)
            .set(User::getUpdatedAt, LocalDateTime.now())
            .update();
    }

    /**
     * 批量插入
     */
    @Transactional
    public void batchSave(List<User> users) {
        // 分批插入，每批 1000 条
        List<List<User>> batches = Lists.partition(users, 1000);
        for (List<User> batch : batches) {
            userMapper.batchInsert(batch);
        }
    }

    /**
     * 乐观锁更新
     */
    @Transactional
    public void updateWithOptimisticLock(User user) {
        int rows = userMapper.updateById(user);
        if (rows == 0) {
            throw new BusinessException(ErrorCode.CONCURRENT_UPDATE);
        }
    }
}
```

### 分页查询优化

```java
/**
 * 游标分页 - 大数据量优化
 */
public List<User> findByLastId(Long lastId, int limit) {
    return new LambdaQueryChainWrapper<>(userMapper)
        .gt(lastId != null, User::getId, lastId)
        .orderByAsc(User::getId)
        .last("LIMIT " + limit)
        .list();
}

/**
 * 流式查询 - 大结果集处理
 */
public void processAllUsers(Consumer<User> consumer) {
    try (Cursor<User> cursor = userMapper.selectCursor(
            new LambdaQueryWrapper<User>().orderByAsc(User::getId))) {
        cursor.forEach(consumer);
    }
}
```

### 自定义类型处理器

```java
@MappedTypes(List.class)
@MappedJdbcTypes(JdbcType.VARCHAR)
public class StringListTypeHandler extends BaseTypeHandler<List<String>> {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
            List<String> parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, toJson(parameter));
    }

    @Override
    public List<String> getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return fromJson(rs.getString(columnName));
    }

    @Override
    public List<String> getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return fromJson(rs.getString(columnIndex));
    }

    @Override
    public List<String> getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return fromJson(cs.getString(columnIndex));
    }

    private String toJson(List<String> list) {
        try {
            return MAPPER.writeValueAsString(list);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }

    private List<String> fromJson(String json) {
        if (json == null) return null;
        try {
            return MAPPER.readValue(json, new TypeReference<>() {});
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
}
```

## 最佳实践清单

- [ ] 使用 MyBatis-Plus 简化 CRUD
- [ ] 复杂 SQL 使用 XML 映射
- [ ] 启用分页插件
- [ ] 使用乐观锁处理并发
- [ ] 逻辑删除配置
- [ ] 自动填充创建/更新时间
- [ ] 批量操作分批执行
- [ ] 大数据量使用游标查询
