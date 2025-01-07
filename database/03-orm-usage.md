# ORM 使用指南

## 1. GORM基础

### 1.1 模型定义
```go
import "gorm.io/gorm"

// 基本模型
type User struct {
    gorm.Model               // 包含ID、CreatedAt、UpdatedAt、DeletedAt
    Name      string         `gorm:"size:255;not null"`
    Email     string         `gorm:"uniqueIndex;not null"`
    Age       uint8         
    Birthday  *time.Time
    Active    bool          `gorm:"default:true"`
}

// 关联模型
type Order struct {
    ID        uint
    UserID    uint
    User      User      `gorm:"foreignKey:UserID"`
    Products  []Product `gorm:"many2many:order_products;"`
    Amount    float64
    Status    string    `gorm:"default:'pending'"`
}

// 自定义表名
func (Order) TableName() string {
    return "orders"
}
```

### 1.2 数据库连接
```go
import (
    "gorm.io/gorm"
    "gorm.io/driver/mysql"
)

// MySQL连接
dsn := "user:password@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})

// 连接池设置
sqlDB, err := db.DB()
sqlDB.SetMaxIdleConns(10)
sqlDB.SetMaxOpenConns(100)
sqlDB.SetConnMaxLifetime(time.Hour)
```

## 2. 基本操作

### 2.1 创建记录
```go
// 创建单条记录
user := User{Name: "张三", Email: "zhangsan@example.com"}
result := db.Create(&user)
if result.Error != nil {
    // 处理错误
}

// 批量创建
users := []User{
    {Name: "张三", Email: "zhangsan@example.com"},
    {Name: "李四", Email: "lisi@example.com"},
}
db.Create(&users)

// 使用Map创建
db.Model(&User{}).Create(map[string]interface{}{
    "Name": "张三",
    "Email": "zhangsan@example.com",
})
```

### 2.2 查询记录
```go
// 查询单条记录
var user User
db.First(&user, 1) // 根据主键查询
db.First(&user, "name = ?", "张三")

// 查询多条记录
var users []User
db.Find(&users)
db.Where("age > ?", 20).Find(&users)

// 条件查询
db.Where("name LIKE ?", "%张%").Find(&users)
db.Where("age BETWEEN ? AND ?", 20, 30).Find(&users)
db.Where("email IN ?", []string{"zhangsan@example.com", "lisi@example.com"}).Find(&users)

// 分页查询
var users []User
db.Offset(10).Limit(10).Find(&users)

// 排序
db.Order("age desc, name").Find(&users)
```

### 2.3 更新记录
```go
// 更新单个字段
db.Model(&user).Update("name", "新名字")

// 更新多个字段
db.Model(&user).Updates(User{
    Name: "新名字",
    Age: 18,
})

// 使用Map更新
db.Model(&user).Updates(map[string]interface{}{
    "name": "新名字",
    "age": 18,
})

// 批量更新
db.Model(User{}).Where("age < ?", 20).Update("status", "inactive")
```

### 2.4 删除记录
```go
// 软删除
db.Delete(&user)

// 永久删除
db.Unscoped().Delete(&user)

// 批量删除
db.Where("age < ?", 20).Delete(&User{})

// 删除所有记录
db.Unscoped().Delete(&User{})
```

## 3. 关联操作

### 3.1 一对一关系
```go
type User struct {
    gorm.Model
    Profile   Profile
    ProfileID uint
}

type Profile struct {
    gorm.Model
    UserID   uint
    Address  string
    Phone    string
}

// 预加载
var user User
db.Preload("Profile").First(&user)

// 关联创建
user.Profile = Profile{Address: "北京市"}
db.Create(&user)
```

### 3.2 一对多关系
```go
type User struct {
    gorm.Model
    Orders []Order
}

type Order struct {
    gorm.Model
    UserID uint
    Amount float64
}

// 预加载
var user User
db.Preload("Orders").First(&user)

// 关联查询
var orders []Order
db.Model(&user).Association("Orders").Find(&orders)

// 添加关联
db.Model(&user).Association("Orders").Append(&Order{Amount: 100})

// 删除关联
db.Model(&user).Association("Orders").Delete(&orders)
```

### 3.3 多对多关系
```go
type Product struct {
    gorm.Model
    Name   string
    Orders []Order `gorm:"many2many:order_products;"`
}

// 添加关联
db.Model(&order).Association("Products").Append(&products)

// 替换关联
db.Model(&order).Association("Products").Replace(&newProducts)

// 删除关联
db.Model(&order).Association("Products").Clear()
```

## 4. 高级特性

### 4.1 事务处理
```go
// 自动事务
err := db.Transaction(func(tx *gorm.DB) error {
    // 在事务中执行操作
    if err := tx.Create(&user).Error; err != nil {
        return err
    }
    
    if err := tx.Create(&order).Error; err != nil {
        return err
    }
    
    return nil
})

// 手动事务
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

return tx.Commit().Error
```

### 4.2 钩子方法
```go
type User struct {
    gorm.Model
    Password string
}

// 创建前的钩子
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
    // 密码加密
    hashedPassword, err := bcrypt.GenerateFromPassword(
        []byte(u.Password), 
        bcrypt.DefaultCost,
    )
    if err != nil {
        return err
    }
    u.Password = string(hashedPassword)
    return nil
}

// 其他钩子
func (u *User) AfterCreate(tx *gorm.DB) (err error)
func (u *User) BeforeUpdate(tx *gorm.DB) (err error)
func (u *User) AfterUpdate(tx *gorm.DB) (err error)
func (u *User) BeforeDelete(tx *gorm.DB) (err error)
func (u *User) AfterDelete(tx *gorm.DB) (err error)
```

## 5. 性能优化

### 5.1 批量操作
```go
// 批量插入
var users []User
// 每次插入100条记录
db.CreateInBatches(users, 100)

// 批量更新
db.Session(&gorm.Session{AllowGlobalUpdate: true}).
    Model(&User{}).
    Where("role = ?", "user").
    Updates(map[string]interface{}{"status": "inactive"})
```

### 5.2 预加载优化
```go
// 选择性预加载
db.Preload("Orders", "status = ?", "paid").Find(&users)

// 嵌套预加载
db.Preload("Orders.Products").Find(&users)

// 自定义预加载
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
    return db.Order("created_at DESC").Limit(5)
}).Find(&users)
```

## 6. 最佳实践

### 6.1 仓储模式
```go
type UserRepository interface {
    Create(user *User) error
    FindByID(id uint) (*User, error)
    Update(user *User) error
    Delete(id uint) error
}

type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Create(user *User) error {
    return r.db.Create(user).Error
}

func (r *userRepository) FindByID(id uint) (*User, error) {
    var user User
    err := r.db.First(&user, id).Error
    return &user, err
}
```

### 6.2 错误处理
```go
// 自定义错误
type NotFoundError struct {
    Entity string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s not found", e.Entity)
}

// 错误处理
func (r *userRepository) FindByID(id uint) (*User, error) {
    var user User
    err := r.db.First(&user, id).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, &NotFoundError{Entity: "User"}
        }
        return nil, err
    }
    return &user, nil
}
``` 