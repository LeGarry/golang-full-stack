# 调试技巧

## 1. Go调试工具

### 1.1 Delve调试器
```bash
# 安装Delve
go install github.com/go-delve/delve/cmd/dlv@latest

# 基本命令
dlv debug main.go    # 启动调试
dlv test ./...      # 调试测试
dlv attach pid      # 附加到进程

# 调试命令
(dlv) break main.go:20    # 设置断点
(dlv) continue           # 继续执行
(dlv) next              # 下一步
(dlv) step              # 步入
(dlv) stepout           # 步出
(dlv) print variable    # 打印变量
(dlv) locals            # 显示局部变量
(dlv) goroutines        # 显示所有goroutine
```

### 1.2 VSCode调试配置
```json
// launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch Package",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceFolder}/cmd/main.go",
            "env": {
                "ENV": "development"
            },
            "args": []
        },
        {
            "name": "Debug Test",
            "type": "go",
            "request": "launch",
            "mode": "test",
            "program": "${workspaceFolder}/${relativeFileDirname}",
            "showLog": true
        }
    ]
}
```

## 2. 日志调试

### 2.1 日志配置
```go
import (
    "github.com/sirupsen/logrus"
    "os"
)

func initLogger() {
    // 设置输出格式
    logrus.SetFormatter(&logrus.JSONFormatter{
        TimestampFormat: "2006-01-02 15:04:05",
    })
    
    // 设置输出
    file, err := os.OpenFile("app.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
    if err == nil {
        logrus.SetOutput(file)
    }
    
    // 设置日志级别
    logrus.SetLevel(logrus.DebugLevel)
}

// 使用日志
func someFunction() {
    logrus.WithFields(logrus.Fields{
        "user_id": 123,
        "action": "login",
    }).Info("User logged in")
    
    // 错误日志
    if err != nil {
        logrus.WithError(err).Error("Failed to process request")
    }
}
```

### 2.2 请求日志中间件
```go
func LoggerMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        
        // 处理请求
        c.Next()
        
        // 记录请求信息
        latency := time.Since(start)
        statusCode := c.Writer.Status()
        clientIP := c.ClientIP()
        
        logrus.WithFields(logrus.Fields{
            "status_code": statusCode,
            "latency": latency,
            "client_ip": clientIP,
            "method": c.Request.Method,
            "path": path,
        }).Info("HTTP Request")
    }
}
```

## 3. 性能分析

### 3.1 pprof工具
```go
import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    // 启动pprof服务
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    
    // 应用主逻辑
}

// 使用pprof
// CPU分析
go tool pprof http://localhost:6060/debug/pprof/profile

// 内存分析
go tool pprof http://localhost:6060/debug/pprof/heap

// Goroutine分析
go tool pprof http://localhost:6060/debug/pprof/goroutine

// 在pprof交互式界面中
(pprof) top        # 显示最耗时的函数
(pprof) web        # 在浏览器中查看图形
(pprof) list func  # 查看函数代码
```

### 3.2 性能基准测试
```go
// benchmark_test.go
func BenchmarkFunction(b *testing.B) {
    // 重置计时器
    b.ResetTimer()
    
    // 运行n次
    for i := 0; i < b.N; i++ {
        function()
    }
}

// 运行基准测试
go test -bench=. -benchmem

// 生成CPU分析
go test -bench=. -cpuprofile=cpu.prof

// 生成内存分析
go test -bench=. -memprofile=mem.prof
```

## 4. 错误追踪

### 4.1 错误处理
```go
import "github.com/pkg/errors"

func doSomething() error {
    if err := someOperation(); err != nil {
        return errors.Wrap(err, "failed to do something")
    }
    return nil
}

// 错误处理中间件
func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        
        if len(c.Errors) > 0 {
            err := c.Errors.Last()
            
            // 记录错误堆栈
            logrus.WithError(errors.WithStack(err.Err)).Error("Request failed")
            
            // 返回错误响应
            c.JSON(500, gin.H{
                "error": err.Error(),
            })
        }
    }
}
```

### 4.2 panic处理
```go
func RecoveryMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if err := recover(); err != nil {
                // 获取堆栈信息
                stack := make([]byte, 4096)
                stack = stack[:runtime.Stack(stack, false)]
                
                logrus.WithFields(logrus.Fields{
                    "error": err,
                    "stack": string(stack),
                }).Error("Panic recovered")
                
                c.AbortWithStatus(500)
            }
        }()
        
        c.Next()
    }
}
```

## 5. 网络调试

### 5.1 HTTP调试
```go
// HTTP客户端调试
client := &http.Client{
    Transport: &http.Transport{
        // 启用HTTP/2
        ForceAttemptHTTP2: true,
        // 设置超时
        DialContext: (&net.Dialer{
            Timeout:   30 * time.Second,
            KeepAlive: 30 * time.Second,
        }).DialContext,
    },
}

// 请求调试
req, _ := http.NewRequest("GET", "http://api.example.com", nil)
req.Header.Set("X-Debug", "true")

dump, _ := httputil.DumpRequestOut(req, true)
fmt.Printf("Request:\n%s\n", dump)

resp, err := client.Do(req)
if err != nil {
    log.Fatal(err)
}

dump, _ = httputil.DumpResponse(resp, true)
fmt.Printf("Response:\n%s\n", dump)
```

### 5.2 WebSocket调试
```go
import "github.com/gorilla/websocket"

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool {
        return true // 允许所有来源
    },
}

func handleWebSocket(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("WebSocket upgrade failed: %v", err)
        return
    }
    defer conn.Close()
    
    // 启用调试日志
    conn.SetPongHandler(func(string) error {
        log.Printf("Received pong")
        return nil
    })
    
    for {
        messageType, p, err := conn.ReadMessage()
        if err != nil {
            log.Printf("Read error: %v", err)
            return
        }
        
        log.Printf("Received message: %s", string(p))
        
        if err := conn.WriteMessage(messageType, p); err != nil {
            log.Printf("Write error: %v", err)
            return
        }
    }
}
```

## 6. 数据库调试

### 6.1 SQL日志
```go
// GORM日志配置
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: logger.New(
        log.New(os.Stdout, "\r\n", log.LstdFlags),
        logger.Config{
            SlowThreshold: time.Second,
            LogLevel:      logger.Info,
            Colorful:      true,
        },
    ),
})

// 自定义SQL记录器
type SQLLogger struct {
    logger.Interface
}

func (l SQLLogger) Trace(ctx context.Context, begin time.Time, fc func() (sql string, rowsAffected int64), err error) {
    sql, rows := fc()
    elapsed := time.Since(begin)
    
    logrus.WithFields(logrus.Fields{
        "sql": sql,
        "rows": rows,
        "elapsed": elapsed,
    }).Debug("SQL Query")
}
```

### 6.2 数据库连接调试
```go
// 连接池监控
go func() {
    for {
        stats := db.DB().Stats()
        logrus.WithFields(logrus.Fields{
            "max_open_connections": stats.MaxOpenConnections,
            "open_connections": stats.OpenConnections,
            "in_use": stats.InUse,
            "idle": stats.Idle,
            "wait_count": stats.WaitCount,
            "wait_duration": stats.WaitDuration,
        }).Info("DB Stats")
        
        time.Sleep(time.Minute)
    }
}()

// 慢查询监控
db.Callback().Query().Before("gorm:query").Register("slow_log", func(db *gorm.DB) {
    db.Statement.Settings.Store("start_time", time.Now())
})

db.Callback().Query().After("gorm:query").Register("slow_log", func(db *gorm.DB) {
    if v, ok := db.Statement.Settings.Load("start_time"); ok {
        elapsed := time.Since(v.(time.Time))
        if elapsed > time.Second {
            logrus.WithFields(logrus.Fields{
                "sql": db.Statement.SQL.String(),
                "elapsed": elapsed,
            }).Warn("Slow Query")
        }
    }
})
``` 