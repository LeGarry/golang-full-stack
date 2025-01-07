# Golang Web 框架

## 1. 主流 Web 框架介绍

### 1.1 Gin
- 轻量级 Web 框架
- 高性能
- 优秀的路由系统
- 中间件支持
- 参数验证
- 错误管理

基本使用示例：
```go
package main

import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()
    
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "pong",
        })
    })
    
    r.Run(":8080")
}
```

### 1.2 Echo
- 高性能、轻量级
- 可扩展
- 数据绑定
- 中间件支持

### 1.3 Beego
- 全功能框架
- MVC 架构
- ORM 支持
- 丰富的工具集

## 2. 路由系统

### 2.1 基本路由
```go
// GET 请求
r.GET("/users", GetUsers)

// POST 请求
r.POST("/users", CreateUser)

// 路由参数
r.GET("/users/:id", GetUser)

// 路由组
v1 := r.Group("/api/v1")
{
    v1.GET("/users", GetUsers)
    v1.POST("/users", CreateUser)
}
```

### 2.2 中间件
```go
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        t := time.Now()
        c.Next()
        latency := time.Since(t)
        log.Printf("耗时: %v", latency)
    }
}

// 使用中间件
r.Use(Logger())
```

## 3. 请求处理

### 3.1 参数获取
```go
// URL 参数
id := c.Param("id")

// 查询参数
name := c.Query("name")

// 表单参数
username := c.PostForm("username")

// JSON 数据
var json struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}
c.BindJSON(&json)
```

### 3.2 响应处理
```go
// JSON 响应
c.JSON(200, gin.H{
    "message": "success",
    "data": user,
})

// XML 响应
c.XML(200, user)

// HTML 响应
c.HTML(200, "index.html", gin.H{
    "title": "首页",
})
```

## 4. 视图渲染

### 4.1 HTML 模板
```go
r.LoadHTMLGlob("templates/*")

r.GET("/", func(c *gin.Context) {
    c.HTML(200, "index.html", gin.H{
        "title": "首页",
    })
})
```

### 4.2 静态文件服务
```go
r.Static("/static", "./static")
```

## 5. 错误处理
```go
func handleError(c *gin.Context) {
    defer func() {
        if err := recover(); err != nil {
            c.JSON(500, gin.H{
                "error": "服务器内部错误",
            })
        }
    }()
    c.Next()
}
```

## 6. 文件上传
```go
r.POST("/upload", func(c *gin.Context) {
    file, _ := c.FormFile("file")
    c.SaveUploadedFile(file, "uploads/"+file.Filename)
    c.String(200, "上传成功")
})
```

## 7. 安全性考虑
- CORS 配置
- XSS 防护
- CSRF 防护
- Rate Limiting

## 8. 性能优化
1. 使用适当的并发控制
2. 合理使用缓存
3. 数据库连接池管理
4. 请求超时控制
5. 负载均衡 