# MySQL 数据库最佳实践

## 角色设定

你是一位精通 MySQL 8.0 的数据库专家，擅长表结构设计、SQL 优化、索引策略和高可用架构。

## 提示词模板

### 表结构设计

```
请帮我设计数据库表结构：
- 业务场景：[描述业务]
- 主要实体：[列出实体]
- 关系类型：[一对一/一对多/多对多]
- 数据量预估：[预估数据量]
- 性能要求：[读写比例/响应时间]

请包含：
1. 建表语句
2. 索引设计
3. 字段注释
4. 分表策略（如需要）
```

### SQL 优化

```
请帮我优化以下 SQL：
[粘贴 SQL]

执行计划：
[粘贴 EXPLAIN 结果]

表结构：
[粘贴表结构]

请分析问题并提供优化方案。
```

## 核心代码示例

### 建表规范

```sql
-- 用户表
CREATE TABLE `user` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `username` VARCHAR(50) NOT NULL COMMENT '用户名',
    `email` VARCHAR(100) NOT NULL COMMENT '邮箱',
    `password_hash` VARCHAR(255) NOT NULL COMMENT '密码哈希',
    `phone` VARCHAR(20) DEFAULT NULL COMMENT '手机号',
    `avatar` VARCHAR(255) DEFAULT NULL COMMENT '头像URL',
    `status` TINYINT NOT NULL DEFAULT 1 COMMENT '状态: 0-禁用, 1-正常',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    `deleted_at` DATETIME DEFAULT NULL COMMENT '删除时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_username` (`username`),
    UNIQUE KEY `uk_email` (`email`),
    KEY `idx_phone` (`phone`),
    KEY `idx_status_created` (`status`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户表';

-- 订单表
CREATE TABLE `order` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no` VARCHAR(32) NOT NULL COMMENT '订单号',
    `user_id` BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    `total_amount` DECIMAL(10,2) NOT NULL COMMENT '订单总金额',
    `pay_amount` DECIMAL(10,2) NOT NULL COMMENT '实付金额',
    `status` TINYINT NOT NULL DEFAULT 0 COMMENT '状态: 0-待支付, 1-已支付, 2-已发货, 3-已完成, 4-已取消',
    `pay_time` DATETIME DEFAULT NULL COMMENT '支付时间',
    `ship_time` DATETIME DEFAULT NULL COMMENT '发货时间',
    `receive_time` DATETIME DEFAULT NULL COMMENT '收货时间',
    `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_id` (`user_id`),
    KEY `idx_status_created` (`status`, `created_at`),
    KEY `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='订单表';

-- 订单明细表
CREATE TABLE `order_item` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_id` BIGINT UNSIGNED NOT NULL COMMENT '订单ID',
    `product_id` BIGINT UNSIGNED NOT NULL COMMENT '商品ID',
    `product_name` VARCHAR(200) NOT NULL COMMENT '商品名称',
    `product_image` VARCHAR(255) DEFAULT NULL COMMENT '商品图片',
    `price` DECIMAL(10,2) NOT NULL COMMENT '商品单价',
    `quantity` INT UNSIGNED NOT NULL COMMENT '数量',
    `total_price` DECIMAL(10,2) NOT NULL COMMENT '小计金额',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    PRIMARY KEY (`id`),
    KEY `idx_order_id` (`order_id`),
    KEY `idx_product_id` (`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='订单明细表';
```

### 索引设计原则

```sql
-- 1. 联合索引最左前缀原则
-- 索引 (a, b, c) 可以支持: a / a,b / a,b,c 查询
CREATE INDEX idx_abc ON table_name (a, b, c);

-- ✅ 可以使用索引
SELECT * FROM table_name WHERE a = 1;
SELECT * FROM table_name WHERE a = 1 AND b = 2;
SELECT * FROM table_name WHERE a = 1 AND b = 2 AND c = 3;

-- ❌ 无法使用索引
SELECT * FROM table_name WHERE b = 2;
SELECT * FROM table_name WHERE c = 3;
SELECT * FROM table_name WHERE b = 2 AND c = 3;

-- 2. 覆盖索引 - 避免回表
CREATE INDEX idx_user_status_name ON user (status, name);

-- ✅ 覆盖索引，不需要回表
SELECT status, name FROM user WHERE status = 1;

-- ❌ 需要回表查询其他字段
SELECT * FROM user WHERE status = 1;

-- 3. 索引下推 (ICP)
-- 可以在索引遍历过程中对索引包含的字段先做判断
CREATE INDEX idx_name_age ON user (name, age);

-- age > 20 会在索引层过滤
SELECT * FROM user WHERE name LIKE '张%' AND age > 20;

-- 4. 前缀索引 - 对长字符串建立索引
CREATE INDEX idx_email_prefix ON user (email(10));
```

### SQL 优化技巧

```sql
-- 1. 避免 SELECT *
-- ❌ 差
SELECT * FROM user WHERE id = 1;
-- ✅ 好
SELECT id, username, email FROM user WHERE id = 1;

-- 2. 避免在 WHERE 子句中对字段进行函数操作
-- ❌ 差 - 无法使用索引
SELECT * FROM order WHERE YEAR(created_at) = 2024;
-- ✅ 好 - 可以使用索引
SELECT * FROM order WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- 3. 使用 LIMIT 分页优化
-- ❌ 差 - 深度分页性能差
SELECT * FROM order ORDER BY id LIMIT 1000000, 10;
-- ✅ 好 - 使用游标分页
SELECT * FROM order WHERE id > 1000000 ORDER BY id LIMIT 10;

-- 4. 批量插入
-- ❌ 差 - 多次网络往返
INSERT INTO user (name, email) VALUES ('user1', 'user1@example.com');
INSERT INTO user (name, email) VALUES ('user2', 'user2@example.com');
-- ✅ 好 - 批量插入
INSERT INTO user (name, email) VALUES
    ('user1', 'user1@example.com'),
    ('user2', 'user2@example.com'),
    ('user3', 'user3@example.com');

-- 5. 避免使用 OR，改用 IN 或 UNION
-- ❌ 可能导致全表扫描
SELECT * FROM user WHERE status = 0 OR status = 1;
-- ✅ 使用 IN
SELECT * FROM user WHERE status IN (0, 1);

-- 6. EXISTS 代替 IN (子查询数据量大时)
-- ❌ 子查询结果集大时性能差
SELECT * FROM user WHERE id IN (SELECT user_id FROM order);
-- ✅ 使用 EXISTS
SELECT * FROM user u WHERE EXISTS (SELECT 1 FROM order o WHERE o.user_id = u.id);

-- 7. 使用 EXPLAIN 分析查询
EXPLAIN SELECT * FROM order WHERE user_id = 1 AND status = 1;
```

### 分页查询优化

```sql
-- 方案1: 游标分页 (推荐)
SELECT * FROM order
WHERE id > #{lastId}
ORDER BY id
LIMIT 20;

-- 方案2: 延迟关联
SELECT o.* FROM order o
INNER JOIN (
    SELECT id FROM order
    ORDER BY created_at DESC
    LIMIT 1000000, 20
) t ON o.id = t.id;

-- 方案3: 覆盖索引 + 子查询
SELECT * FROM order
WHERE id >= (
    SELECT id FROM order
    ORDER BY id
    LIMIT 1000000, 1
)
LIMIT 20;
```

### 事务与锁

```sql
-- 事务隔离级别
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 行锁 - FOR UPDATE
START TRANSACTION;
SELECT * FROM account WHERE id = 1 FOR UPDATE;
UPDATE account SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- 乐观锁 - 版本号
UPDATE product
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = #{version};

-- 间隙锁防止幻读 (RR 隔离级别)
SELECT * FROM order WHERE user_id = 1 FOR UPDATE;
-- 会锁住 user_id = 1 的所有记录及其间隙

-- 死锁检测
SHOW ENGINE INNODB STATUS;
```

### 分库分表

```sql
-- 按用户ID分表 (水平分表)
-- user_0, user_1, user_2, user_3 (4张表)
-- 路由规则: table_index = user_id % 4

-- 分表后的查询需要带上分片键
SELECT * FROM user_{user_id % 4} WHERE user_id = 12345;

-- 订单表按时间分表
-- order_202401, order_202402, order_202403...
-- 需要知道时间范围才能定位表
SELECT * FROM order_202403 WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';
```

## 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 表名 | 小写下划线 | user_order |
| 字段名 | 小写下划线 | created_at |
| 主键 | id | id |
| 外键 | 表名_id | user_id |
| 唯一索引 | uk_字段 | uk_email |
| 普通索引 | idx_字段 | idx_status |
| 联合索引 | idx_字段1_字段2 | idx_status_created |

## 最佳实践清单

- [ ] 使用 InnoDB 引擎
- [ ] 主键使用自增 BIGINT
- [ ] 字符集使用 utf8mb4
- [ ] 字段添加 NOT NULL 和默认值
- [ ] 合理设计索引，避免冗余
- [ ] 大表考虑分库分表
- [ ] 使用 EXPLAIN 分析慢查询
- [ ] 避免深度分页，使用游标
