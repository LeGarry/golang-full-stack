# 数据库设计

## 1. 数据库设计原则

### 1.1 范式设计
1. 第一范式（1NF）
   - 每个字段都是原子性的，不可再分
   - 每个字段都有唯一的名称
   - 同一字段的数据类型必须一致

2. 第二范式（2NF）
   - 满足1NF
   - 每个非主键字段完全依赖于主键

3. 第三范式（3NF）
   - 满足2NF
   - 消除传递依赖

### 1.2 命名规范
```sql
-- 表名使用复数形式
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 关联表使用下划线连接
CREATE TABLE user_roles (
    user_id BIGINT,
    role_id BIGINT,
    PRIMARY KEY (user_id, role_id)
);
```

## 2. 表结构设计

### 2.1 用户表设计
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    status TINYINT DEFAULT 1 COMMENT '1:active, 0:inactive',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_email (email),
    INDEX idx_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

### 2.2 关联表设计
```sql
-- 一对多关系
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 多对多关系
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL
);

CREATE TABLE order_products (
    order_id BIGINT,
    product_id BIGINT,
    quantity INT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

## 3. 索引设计

### 3.1 索引类型
```sql
-- 主键索引
CREATE TABLE articles (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(200) NOT NULL
);

-- 唯一索引
CREATE UNIQUE INDEX idx_email ON users(email);

-- 复合索引
CREATE INDEX idx_name_status ON products(name, status);

-- 全文索引
CREATE FULLTEXT INDEX idx_content ON articles(content);
```

### 3.2 索引优化
```sql
-- 选择性高的列创建索引
CREATE INDEX idx_mobile ON users(mobile);

-- 经常作为查询条件的列创建索引
CREATE INDEX idx_status_created ON orders(status, created_at);

-- 避免冗余索引
-- 不好的示例
CREATE INDEX idx_a ON table1(a);
CREATE INDEX idx_a_b ON table1(a, b); -- 冗余

-- 好的示例
CREATE INDEX idx_a_b ON table1(a, b); -- 可以同时满足a和a,b的查询
```

## 4. 分区设计

### 4.1 范围分区
```sql
CREATE TABLE orders (
    id BIGINT NOT NULL,
    created_at DATE NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (id, created_at)
) 
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### 4.2 列表分区
```sql
CREATE TABLE sales (
    id BIGINT NOT NULL,
    region_code INT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (id, region_code)
)
PARTITION BY LIST (region_code) (
    PARTITION p_east VALUES IN (1, 2, 3),
    PARTITION p_west VALUES IN (4, 5, 6),
    PARTITION p_north VALUES IN (7, 8, 9),
    PARTITION p_south VALUES IN (10, 11, 12)
);
```

## 5. 性能优化

### 5.1 表优化
```sql
-- 选择合适的数据类型
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT, -- 使用UNSIGNED节省空间
    name VARCHAR(100) NOT NULL,     -- 根据实际需求设置长度
    price DECIMAL(10,2) NOT NULL,   -- 使用DECIMAL存储金额
    description TEXT,               -- 大文本使用TEXT
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 添加合适的注释
ALTER TABLE users COMMENT '用户信息表';
ALTER TABLE users MODIFY COLUMN status TINYINT COMMENT '用户状态: 1-活跃, 0-禁用';
```

### 5.2 查询优化
```sql
-- 使用EXPLAIN分析查询
EXPLAIN SELECT * FROM users WHERE status = 1;

-- 优化JOIN查询
SELECT u.*, o.total_amount
FROM users u
    INNER JOIN orders o ON u.id = o.user_id
WHERE u.status = 1
    AND o.created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
USE INDEX (idx_status, idx_created_at);

-- 避免SELECT *
SELECT id, username, email 
FROM users 
WHERE status = 1;
```

## 6. 安全设计

### 6.1 权限控制
```sql
-- 创建只读用户
CREATE USER 'reader'@'localhost' IDENTIFIED BY 'password';
GRANT SELECT ON database.* TO 'reader'@'localhost';

-- 创建应用用户
CREATE USER 'app'@'%' IDENTIFIED BY 'password';
GRANT SELECT, INSERT, UPDATE, DELETE ON database.* TO 'app'@'%';

-- 收回权限
REVOKE ALL PRIVILEGES ON database.* FROM 'user'@'localhost';
```

### 6.2 数据加密
```sql
-- 使用内置加密函数
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(255) NOT NULL,
    credit_card VARCHAR(255) GENERATED ALWAYS AS
        (AES_ENCRYPT(credit_card_number, 'encryption_key')) STORED
);

-- 创建加密视图
CREATE VIEW user_safe_view AS
SELECT 
    id,
    username,
    CONCAT('****', RIGHT(mobile, 4)) as mobile
FROM users;
```

## 7. 备份策略

### 7.1 备份脚本
```bash
#!/bin/bash

# MySQL备份脚本
DB_USER="backup_user"
DB_PASS="password"
DB_NAME="database"
BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d)

# 创建备份目录
mkdir -p $BACKUP_DIR/$DATE

# 执行备份
mysqldump -u$DB_USER -p$DB_PASS $DB_NAME > $BACKUP_DIR/$DATE/$DB_NAME.sql

# 压缩备份文件
gzip $BACKUP_DIR/$DATE/$DB_NAME.sql

# 删除30天前的备份
find $BACKUP_DIR -type d -mtime +30 -exec rm -rf {} \;
```

### 7.2 恢复策略
```sql
-- 创建恢复点
CREATE TABLE backup_points (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    point_name VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) DEFAULT 'active'
);

-- 记录数据变更
CREATE TABLE data_changes (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    table_name VARCHAR(50) NOT NULL,
    record_id BIGINT NOT NULL,
    change_type VARCHAR(20) NOT NULL,
    old_data JSON,
    new_data JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
``` 