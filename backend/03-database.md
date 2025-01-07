# Golang 数据库操作

## 1. 数据库驱动

### 1.1 MySQL
```go
import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql"
)

// 连接数据库
db, err := sql.Open("mysql", "user:password@tcp(127.0.0.1:3306)/dbname")
if err != nil {
    log.Fatal(err)
}
defer db.Close()
```

### 1.2 PostgreSQL
```go
import (
    "database/sql"
    _ "github.com/lib/pq"
)

// 连接数据库
db, err := sql.Open("postgres", "host=localhost port=5432 user=postgres password=password dbname=mydb sslmode=disable")
```

## 2. GORM 使用

### 2.1 模型定义
```go
type User struct {
    gorm.Model
    Name     string
    Age      int
    Email    string `gorm:"type:varchar(100);unique_index"`
    Birthday *time.Time
}
```

### 2.2 数据库操作
```go
// 创建记录
user := User{Name: "张三", Age: 18}
db.Create(&user)

// 查询记录
var user User
db.First(&user, 1) // 查询id为1的用户
db.First(&user, "name = ?", "张三")

// 更新记录
db.Model(&user).Update("name", "李四")
db.Model(&user).Updates(map[string]interface{}{
    "name": "李四",
    "age": 20,
})

// 删除记录
db.Delete(&user)
```

### 2.3 关联关系
```go
type Order struct {
    gorm.Model
    UserID uint
    User   User
    Price  float64
}

// 预加载
db.Preload("User").Find(&orders)
```

## 3. 事务处理

### 3.1 原生SQL事务
```go
tx, err := db.Begin()
if err != nil {
    log.Fatal(err)
}

defer func() {
    if r := recover(); r != nil {
        tx.Rollback()
    }
}()

// 执行事务操作
stmt, err := tx.Prepare("INSERT INTO users(name) VALUES(?)")
if err != nil {
    tx.Rollback()
    log.Fatal(err)
}
defer stmt.Close()

_, err = stmt.Exec("张三")
if err != nil {
    tx.Rollback()
    log.Fatal(err)
}

// 提交事务
err = tx.Commit()
if err != nil {
    log.Fatal(err)
}
```

### 3.2 GORM事务
```go
// 方式1
tx := db.Begin()
defer func() {
    if r := recover(); r != nil {
        tx.Rollback()
    }
}()

if err := tx.Create(&user).Error; err != nil {
    tx.Rollback()
    return err
}

if err := tx.Create(&order).Error; err != nil {
    tx.Rollback()
    return err
}

tx.Commit()

// 方式2
db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err
    }
    if err := tx.Create(&order).Error; err != nil {
        return err
    }
    return nil
})
```

## 4. 连接池配置
```go
db.DB().SetMaxIdleConns(10)
db.DB().SetMaxOpenConns(100)
db.DB().SetConnMaxLifetime(time.Hour)
```

## 5. 查询优化

### 5.1 索引使用
```go
// 添加索引
db.Model(&User{}).AddIndex("idx_name", "name")
db.Model(&User{}).AddUniqueIndex("idx_email", "email")

// 复合索引
db.Model(&User{}).AddIndex("idx_name_age", "name", "age")
```

### 5.2 查询技巧
```go
// 使用Where条件
db.Where("name LIKE ?", "%张%").Find(&users)

// 使用Scopes
func Age(age int) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("age > ?", age)
    }
}

db.Scopes(Age(20)).Find(&users)
```

## 6. 数据迁移
```go
// 自动迁移
db.AutoMigrate(&User{}, &Order{})

// 创建表
db.CreateTable(&User{})

// 删除表
db.DropTable(&User{})
```

## 7. 最佳实践
1. 使用预编译语句防止SQL注入
2. 合理使用事务确保数据一致性
3. 正确配置连接池参数
4. 建立合适的索引提高查询效率
5. 大量数据操作时使用批量操作
6. 定期进行数据库维护和优化 