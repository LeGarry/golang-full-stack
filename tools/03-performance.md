# 性能优化

## 1. 代码优化

### 1.1 内存优化
```go
// 使用对象池
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func processData(data []byte) {
    buffer := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buffer.Reset()
        bufferPool.Put(buffer)
    }()
    
    // 使用buffer处理数据
}

// 减少内存分配
type User struct {
    ID   int64
    Name string
}

// 使用指针接收者避免复制
func (u *User) Process() {
    // 处理用户数据
}

// 使用切片预分配
users := make([]User, 0, expectedSize)
```

### 1.2 并发优化
```go
// 使用工作池
type WorkerPool struct {
    workers int
    tasks   chan func()
}

func NewWorkerPool(workers int) *WorkerPool {
    pool := &WorkerPool{
        workers: workers,
        tasks:   make(chan func(), workers),
    }
    
    for i := 0; i < workers; i++ {
        go pool.worker()
    }
    
    return pool
}

func (p *WorkerPool) worker() {
    for task := range p.tasks {
        task()
    }
}

func (p *WorkerPool) Submit(task func()) {
    p.tasks <- task
}

// 使用示例
pool := NewWorkerPool(runtime.NumCPU())
for i := 0; i < 100; i++ {
    pool.Submit(func() {
        // 执行任务
    })
}
```

## 2. 数据库优化

### 2.1 查询优化
```go
// 使用索引
db.Exec(`
    CREATE INDEX idx_user_status ON users(status);
    CREATE INDEX idx_created_at ON orders(created_at);
`)

// 批量操作
func BulkInsert(users []User) error {
    const batchSize = 1000
    for i := 0; i < len(users); i += batchSize {
        end := i + batchSize
        if end > len(users) {
            end = len(users)
        }
        
        err := db.Create(&users[i:end]).Error
        if err != nil {
            return err
        }
    }
    return nil
}

// 使用预加载
db.Preload("Orders").Preload("Profile").Find(&users)
```

### 2.2 连接池优化
```go
// 配置连接池
sqlDB, err := db.DB()
if err != nil {
    log.Fatal(err)
}

// 最大空闲连接数
sqlDB.SetMaxIdleConns(10)

// 最大打开连接数
sqlDB.SetMaxOpenConns(100)

// 连接最大复用时间
sqlDB.SetConnMaxLifetime(time.Hour)

// 监控连接池状态
type DBStats struct {
    MaxOpenConnections int
    OpenConnections    int
    InUse             int
    Idle              int
    WaitCount         int64
    WaitDuration      time.Duration
}

func monitorDBStats(db *sql.DB) {
    ticker := time.NewTicker(time.Minute)
    for range ticker.C {
        stats := db.Stats()
        log.Printf("DB Stats: %+v", stats)
    }
}
```

## 3. 缓存优化

### 3.1 Redis缓存
```go
import "github.com/go-redis/redis/v8"

// Redis配置
rdb := redis.NewClient(&redis.Options{
    Addr:     "localhost:6379",
    Password: "",
    DB:       0,
    PoolSize: 100,
})

// 缓存查询结果
func GetUserWithCache(ctx context.Context, id int64) (*User, error) {
    key := fmt.Sprintf("user:%d", id)
    
    // 尝试从缓存获取
    val, err := rdb.Get(ctx, key).Result()
    if err == nil {
        var user User
        if err := json.Unmarshal([]byte(val), &user); err == nil {
            return &user, nil
        }
    }
    
    // 从数据库获取
    var user User
    if err := db.First(&user, id).Error; err != nil {
        return nil, err
    }
    
    // 存入缓存
    if data, err := json.Marshal(user); err == nil {
        rdb.Set(ctx, key, data, time.Hour)
    }
    
    return &user, nil
}
```

### 3.2 本地缓存
```go
import "github.com/patrickmn/go-cache"

// 创建本地缓存
c := cache.New(5*time.Minute, 10*time.Minute)

// 缓存接口
type Cache interface {
    Get(key string) (interface{}, bool)
    Set(key string, value interface{}, d time.Duration)
    Delete(key string)
}

// 实现缓存接口
type LocalCache struct {
    cache *cache.Cache
}

func (c *LocalCache) Get(key string) (interface{}, bool) {
    return c.cache.Get(key)
}

func (c *LocalCache) Set(key string, value interface{}, d time.Duration) {
    c.cache.Set(key, value, d)
}

func (c *LocalCache) Delete(key string) {
    c.cache.Delete(key)
}
```

## 4. HTTP优化

### 4.1 请求优化
```go
// HTTP客户端优化
client := &http.Client{
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 100,
        IdleConnTimeout:     90 * time.Second,
        TLSHandshakeTimeout: 10 * time.Second,
        DisableCompression:  true,
    },
    Timeout: 30 * time.Second,
}

// 使用压缩
func GzipMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Writer = &gzipWriter{c.Writer}
        c.Next()
    }
}

type gzipWriter struct {
    gin.ResponseWriter
}

func (g *gzipWriter) Write(data []byte) (int, error) {
    return g.ResponseWriter.Write(data)
}
```

### 4.2 响应优化
```go
// JSON序列化优化
import "github.com/json-iterator/go"

var json = jsoniter.ConfigCompatibleWithStandardLibrary

// 响应缓存
func CacheMiddleware(duration time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        key := c.Request.URL.String()
        
        // 检查缓存
        if cached, ok := cache.Get(key); ok {
            c.Data(200, "application/json", cached.([]byte))
            c.Abort()
            return
        }
        
        // 捕获响应
        w := &responseWriter{body: bytes.NewBuffer(nil), ResponseWriter: c.Writer}
        c.Writer = w
        
        c.Next()
        
        // 存储响应
        if w.Status() == 200 {
            cache.Set(key, w.body.Bytes(), duration)
        }
    }
}
```

## 5. 系统优化

### 5.1 系统配置
```bash
# 系统限制配置
# /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535

# 内核参数优化
# /etc/sysctl.conf
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.ip_local_port_range = 1024 65000
net.core.somaxconn = 8192
net.core.netdev_max_backlog = 8192
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 8192
```

### 5.2 监控指标
```go
import "github.com/prometheus/client_golang/prometheus"

// 定义指标
var (
    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )
    
    requestTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )
)

// 注册指标
func init() {
    prometheus.MustRegister(requestDuration)
    prometheus.MustRegister(requestTotal)
}

// 使用指标
func MetricsMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        
        c.Next()
        
        duration := time.Since(start).Seconds()
        requestDuration.WithLabelValues(
            c.Request.Method,
            c.Request.URL.Path,
        ).Observe(duration)
        
        requestTotal.WithLabelValues(
            c.Request.Method,
            c.Request.URL.Path,
            strconv.Itoa(c.Writer.Status()),
        ).Inc()
    }
}
```

## 6. 性能测试

### 6.1 负载测试
```go
// 使用wrk进行负载测试
// wrk -t12 -c400 -d30s http://localhost:8080/api/v1/users

// 测试脚本
func BenchmarkAPI(b *testing.B) {
    server := setupTestServer()
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        resp, err := http.Get(server.URL + "/api/v1/users")
        if err != nil {
            b.Fatal(err)
        }
        resp.Body.Close()
    }
}
```

### 6.2 性能分析
```go
import "runtime/pprof"

// CPU分析
f, err := os.Create("cpu.prof")
if err != nil {
    log.Fatal(err)
}
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()

// 内存分析
f, err := os.Create("mem.prof")
if err != nil {
    log.Fatal(err)
}
pprof.WriteHeapProfile(f)
defer f.Close()

// 使用go tool pprof分析
// go tool pprof cpu.prof
// go tool pprof mem.prof
``` 