# Golang API 设计

## 1. RESTful API 设计原则

### 1.1 基本原则
1. 使用HTTP方法表示操作
   - GET：获取资源
   - POST：创建资源
   - PUT：更新资源
   - DELETE：删除资源
   - PATCH：部分更新

2. 资源命名规范
   - 使用名词复数形式
   - 使用小写字母
   - 使用连字符（-）连接单词
   - 避免使用动词

### 1.2 URL设计示例
```
GET    /api/v1/users          # 获取用户列表
GET    /api/v1/users/:id      # 获取特定用户
POST   /api/v1/users          # 创建用户
PUT    /api/v1/users/:id      # 更新用户
DELETE /api/v1/users/:id      # 删除用户
```

## 2. API版本控制

### 2.1 URL版本控制
```go
r := gin.Default()

v1 := r.Group("/api/v1")
{
    v1.GET("/users", GetUsers)
    v1.POST("/users", CreateUser)
}

v2 := r.Group("/api/v2")
{
    v2.GET("/users", GetUsersV2)
    v2.POST("/users", CreateUserV2)
}
```

### 2.2 Header版本控制
```go
func VersionMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        version := c.GetHeader("API-Version")
        c.Set("version", version)
        c.Next()
    }
}
```

## 3. 请求和响应处理

### 3.1 请求验证
```go
type CreateUserRequest struct {
    Username string `json:"username" binding:"required,min=3,max=32"`
    Email    string `json:"email" binding:"required,email"`
    Age      int    `json:"age" binding:"required,gte=0,lte=130"`
}

func CreateUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{
            "error": "请求参数验证失败",
            "details": err.Error(),
        })
        return
    }
    // 处理请求...
}
```

### 3.2 统一响应格式
```go
type Response struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

func Success(c *gin.Context, data interface{}) {
    c.JSON(200, Response{
        Code:    0,
        Message: "success",
        Data:    data,
    })
}

func Error(c *gin.Context, code int, message string) {
    c.JSON(code, Response{
        Code:    code,
        Message: message,
    })
}
```

## 4. 错误处理

### 4.1 错误码定义
```go
const (
    ErrCodeSuccess       = 0
    ErrCodeParamInvalid = 400
    ErrCodeUnauthorized = 401
    ErrCodeForbidden    = 403
    ErrCodeNotFound     = 404
    ErrCodeServerError  = 500
)

var errorMessages = map[int]string{
    ErrCodeParamInvalid: "请求参数无效",
    ErrCodeUnauthorized: "未授权访问",
    ErrCodeForbidden:    "禁止访问",
    ErrCodeNotFound:     "资源不存在",
    ErrCodeServerError:  "服务器内部错误",
}
```

### 4.2 错误处理中间件
```go
func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()

        if len(c.Errors) > 0 {
            err := c.Errors.Last()
            if customErr, ok := err.Err.(*CustomError); ok {
                Error(c, customErr.Code, customErr.Message)
                return
            }
            Error(c, ErrCodeServerError, "服务器内部错误")
        }
    }
}
```

## 5. 分页处理

### 5.1 分页参数
```go
type Pagination struct {
    Page     int `form:"page" binding:"required,min=1"`
    PageSize int `form:"page_size" binding:"required,min=1,max=100"`
}

type PaginatedResponse struct {
    Total    int64       `json:"total"`
    Page     int         `json:"page"`
    PageSize int         `json:"page_size"`
    Data     interface{} `json:"data"`
}
```

### 5.2 分页查询实现
```go
func GetUserList(c *gin.Context) {
    var pagination Pagination
    if err := c.ShouldBindQuery(&pagination); err != nil {
        Error(c, ErrCodeParamInvalid, "分页参数无效")
        return
    }

    var total int64
    var users []User
    
    offset := (pagination.Page - 1) * pagination.PageSize
    
    db.Model(&User{}).Count(&total)
    db.Offset(offset).Limit(pagination.PageSize).Find(&users)
    
    Success(c, PaginatedResponse{
        Total:    total,
        Page:     pagination.Page,
        PageSize: pagination.PageSize,
        Data:     users,
    })
}
```

## 6. API文档

### 6.1 Swagger配置
```go
import "github.com/swaggo/gin-swagger"
import "github.com/swaggo/files"

// @title User Service API
// @version 1.0
// @description This is a user service API
// @host localhost:8080
// @BasePath /api/v1
func main() {
    r := gin.Default()
    r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
    // ...
}
```

### 6.2 API注释示例
```go
// @Summary 创建用户
// @Description 创建新用户
// @Accept json
// @Produce json
// @Param user body CreateUserRequest true "用户信息"
// @Success 200 {object} Response
// @Failure 400 {object} Response
// @Router /users [post]
func CreateUser(c *gin.Context) {
    // ...
}
```

## 7. 性能优化

### 7.1 缓存实现
```go
import "github.com/go-redis/redis/v8"

func GetUserWithCache(c *gin.Context, id string) {
    // 尝试从缓存获取
    key := "user:" + id
    if cached, err := redis.Get(ctx, key).Result(); err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        Success(c, user)
        return
    }
    
    // 从数据库获取
    var user User
    if err := db.First(&user, id).Error; err != nil {
        Error(c, ErrCodeNotFound, "用户不存在")
        return
    }
    
    // 存入缓存
    cached, _ := json.Marshal(user)
    redis.Set(ctx, key, cached, time.Hour)
    
    Success(c, user)
}
```

### 7.2 并发处理
```go
func GetUserBatch(c *gin.Context, ids []string) {
    users := make(map[string]User)
    var wg sync.WaitGroup
    var mutex sync.Mutex
    
    for _, id := range ids {
        wg.Add(1)
        go func(id string) {
            defer wg.Done()
            var user User
            if err := db.First(&user, id).Error; err == nil {
                mutex.Lock()
                users[id] = user
                mutex.Unlock()
            }
        }(id)
    }
    
    wg.Wait()
    Success(c, users)
}
```

## 8. 安全性考虑

### 8.1 输入验证
```go
func ValidateInput(input string) bool {
    // 移除危险字符
    input = bluemonday.UGCPolicy().Sanitize(input)
    
    // 验证长度
    if len(input) > 1000 {
        return false
    }
    
    // 其他验证规则...
    return true
}
```

### 8.2 API限流
```go
func RateLimit(c *gin.Context) {
    ip := c.ClientIP()
    key := fmt.Sprintf("ratelimit:%s", ip)
    
    count, _ := redis.Incr(ctx, key).Result()
    if count == 1 {
        redis.Expire(ctx, key, time.Minute)
    }
    
    if count > 100 {
        Error(c, 429, "请求过于频繁")
        c.Abort()
        return
    }
    
    c.Next()
}
``` 