# SQL 基础

## 1. 基本查询

### 1.1 SELECT语句
```sql
-- 基本查询
SELECT * FROM users;

-- 指定列查询
SELECT id, username, email FROM users;

-- 条件查询
SELECT * FROM users 
WHERE status = 1 
    AND created_at >= '2023-01-01';

-- 排序
SELECT * FROM users 
ORDER BY created_at DESC, id ASC;

-- 分页
SELECT * FROM users 
LIMIT 10 OFFSET 0;

-- 分组
SELECT status, COUNT(*) as count 
FROM users 
GROUP BY status;

-- 去重
SELECT DISTINCT status FROM users;
```

### 1.2 聚合函数
```sql
-- 计数
SELECT COUNT(*) FROM users;
SELECT COUNT(DISTINCT status) FROM users;

-- 求和
SELECT SUM(amount) FROM orders;

-- 平均值
SELECT AVG(amount) FROM orders;

-- 最大值和最小值
SELECT 
    MAX(amount) as max_amount,
    MIN(amount) as min_amount
FROM orders;

-- 组合使用
SELECT 
    user_id,
    COUNT(*) as order_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 5;
```

## 2. 多表查询

### 2.1 JOIN查询
```sql
-- 内连接
SELECT u.username, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- 左连接
SELECT u.username, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- 右连接
SELECT u.username, o.amount
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- 多表连接
SELECT 
    u.username,
    o.order_number,
    p.name as product_name
FROM users u
    INNER JOIN orders o ON u.id = o.user_id
    INNER JOIN order_products op ON o.id = op.order_id
    INNER JOIN products p ON op.product_id = p.id;
```

### 2.2 子查询
```sql
-- IN子查询
SELECT * FROM users
WHERE id IN (
    SELECT user_id 
    FROM orders 
    WHERE amount > 1000
);

-- EXISTS子查询
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 
    FROM orders o 
    WHERE o.user_id = u.id 
        AND o.amount > 1000
);

-- 相关子查询
SELECT *,
    (SELECT COUNT(*) 
     FROM orders o 
     WHERE o.user_id = u.id) as order_count
FROM users u;
```

## 3. 数据操作

### 3.1 INSERT语句
```sql
-- 单行插入
INSERT INTO users (username, email, status)
VALUES ('张三', 'zhangsan@example.com', 1);

-- 多行插入
INSERT INTO users (username, email, status)
VALUES 
    ('张三', 'zhangsan@example.com', 1),
    ('李四', 'lisi@example.com', 1);

-- 插入查询结果
INSERT INTO user_archive
SELECT * FROM users
WHERE created_at < '2023-01-01';
```

### 3.2 UPDATE语句
```sql
-- 基本更新
UPDATE users 
SET status = 0 
WHERE id = 1;

-- 多字段更新
UPDATE users 
SET 
    status = 0,
    updated_at = NOW()
WHERE id = 1;

-- 使用子查询更新
UPDATE orders
SET total_amount = (
    SELECT SUM(price * quantity)
    FROM order_products
    WHERE order_id = orders.id
);
```

### 3.3 DELETE语句
```sql
-- 基本删除
DELETE FROM users 
WHERE status = 0;

-- 使用JOIN删除
DELETE u
FROM users u
    INNER JOIN blacklist b ON u.id = b.user_id;

-- 截断表
TRUNCATE TABLE temp_users;
```

## 4. 事务处理

### 4.1 事务控制
```sql
-- 开始事务
START TRANSACTION;

-- 创建订单
INSERT INTO orders (user_id, amount) 
VALUES (1, 100);

-- 更新用户余额
UPDATE users 
SET balance = balance - 100 
WHERE id = 1;

-- 提交或回滚
COMMIT;
-- ROLLBACK;

-- 使用保存点
SAVEPOINT point1;
-- 执行一些操作
ROLLBACK TO point1;
```

### 4.2 事务隔离级别
```sql
-- 设置隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 示例：处理并发问题
START TRANSACTION;
SELECT @current_stock := stock FROM products WHERE id = 1 FOR UPDATE;
UPDATE products SET stock = @current_stock - 1 WHERE id = 1;
COMMIT;
```

## 5. 高级特性

### 5.1 窗口函数
```sql
-- ROW_NUMBER()
SELECT 
    *,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) as rank
FROM orders;

-- 移动平均
SELECT 
    *,
    AVG(amount) OVER (
        PARTITION BY user_id 
        ORDER BY created_at 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg
FROM orders;

-- 累计总和
SELECT 
    *,
    SUM(amount) OVER (
        PARTITION BY user_id 
        ORDER BY created_at
    ) as running_total
FROM orders;
```

### 5.2 公共表表达式(CTE)
```sql
-- 基本CTE
WITH user_orders AS (
    SELECT 
        user_id,
        COUNT(*) as order_count,
        SUM(amount) as total_amount
    FROM orders
    GROUP BY user_id
)
SELECT 
    u.username,
    uo.order_count,
    uo.total_amount
FROM users u
    LEFT JOIN user_orders uo ON u.id = uo.user_id;

-- 递归CTE
WITH RECURSIVE category_tree AS (
    -- 基础查询
    SELECT id, name, parent_id, 1 as level
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- 递归查询
    SELECT 
        c.id,
        c.name,
        c.parent_id,
        ct.level + 1
    FROM categories c
        INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree;
```

## 6. 性能优化

### 6.1 查询优化
```sql
-- 使用EXPLAIN
EXPLAIN SELECT * FROM users WHERE status = 1;

-- 强制使用索引
SELECT * FROM users FORCE INDEX (idx_status)
WHERE status = 1;

-- 避免使用函数索引
-- 不好的写法
SELECT * FROM users WHERE YEAR(created_at) = 2023;
-- 好的写法
SELECT * FROM users 
WHERE created_at >= '2023-01-01' 
    AND created_at < '2024-01-01';

-- 使用UNION ALL代替UNION
SELECT * FROM orders WHERE status = 'pending'
UNION ALL
SELECT * FROM orders WHERE status = 'processing';
```

### 6.2 索引优化
```sql
-- 创建复合索引
CREATE INDEX idx_user_status_created 
ON users(status, created_at);

-- 删除不必要的索引
SHOW INDEX FROM users;
DROP INDEX idx_unused ON users;

-- 分析表
ANALYZE TABLE users;

-- 优化表
OPTIMIZE TABLE users;
``` 