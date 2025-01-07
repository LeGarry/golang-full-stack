# Golang 中间件开发

## 1. 中间件基础

### 1.1 中间件概念
- 中间件是一个函数，用于处理请求和响应
- 可以在请求处理前后执行特定操作
- 常用于日志记录、认证、性能监控等

### 1.2 基本结构
```go
func MiddlewareFunc() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 请求前的处理
        
        c.Next()  // 处理请求
        
        // 请求后的处理
    }
}
```

## 2. 常用中间件实现

### 2.1 日志中间件
```go
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        
        c.Next()
        
        latency := time.Since(start)
        statusCode := c.Writer.Status()
        
        log.Printf("[%d] %s %s %v", statusCode, c.Request.Method, path, latency)
    }
}
```

### 2.2 认证中间件
```go
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.JSON(http.StatusUnauthorized, gin.H{
                "error": "未授权访问",
            })
            c.Abort()
            return
        }
        
        // 验证token
        claims, err := validateToken(token)
        if err != nil {
            c.JSON(http.StatusUnauthorized, gin.H{
                "error": "无效的token",
            })
            c.Abort()
            return
        }
        
        // 将用户信息存储到上下文
        c.Set("userId", claims.UserId)
        c.Next()
    }
}
```

### 2.3 错误处理中间件
```go
func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("Panic: %v", err)
                c.JSON(http.StatusInternalServerError, gin.H{
                    "error": "服务器内部错误",
                })
            }
        }()
        
        c.Next()
    }
}
```

### 2.4 CORS中间件
```go
func CORSMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
        c.Writer.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        
        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(204)
            return
        }
        
        c.Next()
    }
}
```

## 3. 中间件使用方式

### 3.1 全局使用
```go
r := gin.New()
r.Use(Logger())
r.Use(AuthMiddleware())
```

### 3.2 路由组使用
```go
api := r.Group("/api")
api.Use(AuthMiddleware())
{
    api.GET("/users", GetUsers)
    api.POST("/users", CreateUser)
}
```

### 3.3 单个路由使用
```go
r.GET("/admin", AuthMiddleware(), AdminHandler)
```

## 4. 自定义中间件开发

### 4.1 性能监控中间件
```go
func MetricsMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        
        c.Next()
        
        duration := time.Since(start)
        path := c.Request.URL.Path
        status := c.Writer.Status()
        
        // 记录指标
        metrics.RecordRequest(path, status, duration)
    }
}
```

### 4.2 限流中间件
```go
func RateLimiter(limit int) gin.HandlerFunc {
    limiter := rate.NewLimiter(rate.Limit(limit), limit)
    
    return func(c *gin.Context) {
        if !limiter.Allow() {
            c.JSON(http.StatusTooManyRequests, gin.H{
                "error": "请求过于频繁",
            })
            c.Abort()
            return
        }
        c.Next()
    }
}
```

## 5. 中间件最佳实践

### 5.1 性能考虑
1. 避免在中间件中进行耗时操作
2. 合理使用goroutine处理异步任务
3. 使用缓存减少重复计算

### 5.2 错误处理
1. 统一的错误处理方式
2. 合适的错误日志记录
3. 友好的错误信息返回

### 5.3 安全性
1. 输入验证和清理
2. 敏感信息保护
3. 适当的超时控制

### 5.4 可维护性
1. 模块化设计
2. 清晰的文档注释
3. 单元测试覆盖

## 6. 常见问题和解决方案

### 6.1 中间件执行顺序
- 按照注册顺序执行
- Next()之前的代码在请求阶段执行
- Next()之后的代码在响应阶段执行

### 6.2 中间件中断
```go
func AbortMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        if someCondition {
            c.Abort()
            return
        }
        c.Next()
    }
}
```

### 6.3 数据传递
```go
// 设置数据
c.Set("key", value)

// 获取数据
value, exists := c.Get("key")
``` 