# 高级性能优化

## 1. 并发优化

### 1.1 Goroutine池
```go
package pool

import (
    "context"
    "sync"
)

type Pool struct {
    workers chan struct{}
    wg      sync.WaitGroup
}

func NewPool(size int) *Pool {
    return &Pool{
        workers: make(chan struct{}, size),
    }
}

func (p *Pool) Submit(task func()) {
    p.workers <- struct{}{} // 获取worker
    p.wg.Add(1)
    
    go func() {
        defer func() {
            <-p.workers // 释放worker
            p.wg.Done()
        }()
        
        task()
    }()
}

func (p *Pool) Wait() {
    p.wg.Wait()
}

// 使用示例
func ProcessItems(items []Item) error {
    pool := NewPool(10) // 创建大小为10的goroutine池
    
    for _, item := range items {
        item := item // 创建副本避免闭包问题
        pool.Submit(func() {
            processItem(item)
        })
    }
    
    pool.Wait()
    return nil
}
```

### 1.2 并发控制
```go
package concurrency

import (
    "context"
    "golang.org/x/sync/semaphore"
)

type ConcurrencyLimiter struct {
    sem *semaphore.Weighted
}

func NewConcurrencyLimiter(limit int64) *ConcurrencyLimiter {
    return &ConcurrencyLimiter{
        sem: semaphore.NewWeighted(limit),
    }
}

func (l *ConcurrencyLimiter) Execute(ctx context.Context, fn func() error) error {
    if err := l.sem.Acquire(ctx, 1); err != nil {
        return err
    }
    defer l.sem.Release(1)
    
    return fn()
}

// 使用示例
func ProcessRequests(ctx context.Context, requests []Request) error {
    limiter := NewConcurrencyLimiter(100) // 限制并发数为100
    
    for _, req := range requests {
        req := req
        if err := limiter.Execute(ctx, func() error {
            return processRequest(req)
        }); err != nil {
            return err
        }
    }
    
    return nil
}
```

## 2. 内存优化

### 2.1 对象池
```go
package pool

import (
    "sync"
)

type Buffer struct {
    data []byte
}

var bufferPool = sync.Pool{
    New: func() interface{} {
        return &Buffer{
            data: make([]byte, 0, 4096),
        }
    },
}

func GetBuffer() *Buffer {
    return bufferPool.Get().(*Buffer)
}

func PutBuffer(b *Buffer) {
    b.data = b.data[:0] // 清空但保留容量
    bufferPool.Put(b)
}

// 使用示例
func ProcessData(data []byte) error {
    buf := GetBuffer()
    defer PutBuffer(buf)
    
    // 使用buf进行数据处理
    buf.data = append(buf.data, data...)
    return nil
}
```

### 2.2 内存分配优化
```go
package optimization

// 预分配内存
func ProcessLargeData(data []int) []int {
    // 预分配足够的容量
    result := make([]int, 0, len(data))
    
    for _, item := range data {
        if item > 0 {
            result = append(result, item)
        }
    }
    
    return result
}

// 避免不必要的内存分配
type Request struct {
    data []byte
}

func (r *Request) Process() error {
    // 重用已有的切片而不是创建新的
    r.data = r.data[:0]
    
    // 追加数据
    r.data = append(r.data, newData...)
    return nil
}
```

## 3. 数据库优化

### 3.1 连接池优化
```go
package database

import (
    "gorm.io/gorm"
    "time"
)

func SetupDB() (*gorm.DB, error) {
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        return nil, err
    }
    
    sqlDB, err := db.DB()
    if err != nil {
        return nil, err
    }
    
    // 设置连接池参数
    sqlDB.SetMaxIdleConns(10)           // 最大空闲连接数
    sqlDB.SetMaxOpenConns(100)          // 最大打开连接数
    sqlDB.SetConnMaxLifetime(time.Hour) // 连接最大生命周期
    
    return db, nil
}
```

### 3.2 查询优化
```go
package database

// 批量插入
func BulkInsert(users []User) error {
    const batchSize = 1000
    
    for i := 0; i < len(users); i += batchSize {
        end := i + batchSize
        if end > len(users) {
            end = len(users)
        }
        
        if err := db.CreateInBatches(users[i:end], batchSize).Error; err != nil {
            return err
        }
    }
    
    return nil
}

// 优化查询
type UserRepository struct {
    db *gorm.DB
}

func (r *UserRepository) GetUserWithPreload(id uint) (*User, error) {
    var user User
    
    // 使用预加载优化N+1问题
    err := r.db.Preload("Posts").
        Preload("Profile").
        Where("id = ?", id).
        First(&user).Error
    
    return &user, err
}

// 使用索引优化
func (r *UserRepository) SearchUsers(query string) ([]User, error) {
    var users []User
    
    // 使用索引字段进行查询
    err := r.db.Where("indexed_field LIKE ?", query+"%").
        Limit(100).
        Find(&users).Error
    
    return users, err
}
```

## 4. 缓存优化

### 4.1 多级缓存
```go
package cache

import (
    "context"
    "time"
    "github.com/go-redis/redis/v8"
    "github.com/patrickmn/go-cache"
)

type MultiLevelCache struct {
    localCache  *cache.Cache
    redisClient *redis.Client
}

func NewMultiLevelCache(redisClient *redis.Client) *MultiLevelCache {
    return &MultiLevelCache{
        localCache:  cache.New(5*time.Minute, 10*time.Minute),
        redisClient: redisClient,
    }
}

func (c *MultiLevelCache) Get(ctx context.Context, key string) (interface{}, error) {
    // 先查本地缓存
    if value, found := c.localCache.Get(key); found {
        return value, nil
    }
    
    // 查Redis缓存
    value, err := c.redisClient.Get(ctx, key).Result()
    if err == nil {
        // 写入本地缓存
        c.localCache.Set(key, value, cache.DefaultExpiration)
        return value, nil
    }
    
    return nil, err
}

func (c *MultiLevelCache) Set(ctx context.Context, key string, value interface{}, expiration time.Duration) error {
    // 写入本地缓存
    c.localCache.Set(key, value, expiration)
    
    // 写入Redis缓存
    return c.redisClient.Set(ctx, key, value, expiration).Err()
}
```

### 4.2 缓存策略
```go
package cache

import (
    "context"
    "encoding/json"
    "time"
)

type CacheStrategy interface {
    Get(ctx context.Context, key string) (interface{}, error)
    Set(ctx context.Context, key string, value interface{}, expiration time.Duration) error
}

// Write-Through缓存
type WriteThroughCache struct {
    cache  CacheStrategy
    db     *gorm.DB
}

func (c *WriteThroughCache) Save(ctx context.Context, key string, value interface{}) error {
    // 同时写入数据库和缓存
    if err := c.db.Create(value).Error; err != nil {
        return err
    }
    
    return c.cache.Set(ctx, key, value, 24*time.Hour)
}

// Write-Back缓存
type WriteBackCache struct {
    cache     CacheStrategy
    db        *gorm.DB
    writeChan chan writeOperation
}

type writeOperation struct {
    key   string
    value interface{}
}

func (c *WriteBackCache) startWriter() {
    go func() {
        for op := range c.writeChan {
            // 异步写入数据库
            if err := c.db.Create(op.value).Error; err != nil {
                // 处理错误，可以重试或记录日志
                continue
            }
        }
    }()
}

func (c *WriteBackCache) Save(ctx context.Context, key string, value interface{}) error {
    // 先写入缓存
    if err := c.cache.Set(ctx, key, value, 24*time.Hour); err != nil {
        return err
    }
    
    // 异步写入数据库
    c.writeChan <- writeOperation{key, value}
    return nil
}
```

## 5. 网络优化

### 5.1 连接复用
```go
package http

import (
    "net"
    "net/http"
    "time"
)

func NewHTTPClient() *http.Client {
    return &http.Client{
        Transport: &http.Transport{
            Proxy: http.ProxyFromEnvironment,
            DialContext: (&net.Dialer{
                Timeout:   30 * time.Second,
                KeepAlive: 30 * time.Second,
            }).DialContext,
            ForceAttemptHTTP2:     true,
            MaxIdleConns:          100,
            IdleConnTimeout:       90 * time.Second,
            TLSHandshakeTimeout:   10 * time.Second,
            ExpectContinueTimeout: 1 * time.Second,
            MaxIdleConnsPerHost:   100,
        },
        Timeout: 30 * time.Second,
    }
}
```

### 5.2 压缩优化
```go
package middleware

import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/compress"
)

func SetupCompression(app *fiber.App) {
    app.Use(compress.New(compress.Config{
        Level: compress.LevelBestSpeed,
    }))
}

// 自定义压缩中间件
func CustomCompressionMiddleware() fiber.Handler {
    return func(c *fiber.Ctx) error {
        // 检查Accept-Encoding
        if c.Get("Accept-Encoding") != "" {
            c.Set("Content-Encoding", "gzip")
            
            // 使用压缩写入器
            return c.Next()
        }
        
        return c.Next()
    }
}
```

## 6. 性能监控

### 6.1 性能指标收集
```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "runtime"
)

var (
    // 内存指标
    memStats = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "app_memory_stats",
            Help: "Memory statistics",
        },
        []string{"type"},
    )
    
    // GC指标
    gcStats = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "app_gc_stats",
            Help: "Garbage collection statistics",
        },
        []string{"type"},
    )
)

func CollectRuntimeMetrics() {
    go func() {
        for {
            var stats runtime.MemStats
            runtime.ReadMemStats(&stats)
            
            memStats.WithLabelValues("alloc").Set(float64(stats.Alloc))
            memStats.WithLabelValues("sys").Set(float64(stats.Sys))
            memStats.WithLabelValues("heap_alloc").Set(float64(stats.HeapAlloc))
            
            gcStats.WithLabelValues("next_gc").Set(float64(stats.NextGC))
            gcStats.WithLabelValues("pause_total").Set(float64(stats.PauseTotalNs))
            
            time.Sleep(time.Second * 15)
        }
    }()
}
```

### 6.2 性能分析
```go
package profiling

import (
    "net/http"
    _ "net/http/pprof"
    "runtime"
    "runtime/pprof"
)

func StartProfiling() {
    // 开启CPU profile
    f, _ := os.Create("cpu.prof")
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()
    
    // 内存profile
    runtime.MemProfileRate = 1
    
    // 启动pprof http服务
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
}

// 使用示例
func main() {
    StartProfiling()
    
    // 应用逻辑...
    
    // 记录内存profile
    f, _ := os.Create("mem.prof")
    pprof.WriteHeapProfile(f)
    f.Close()
}
```

## 7. 负载均衡

### 7.1 客户端负载均衡
```go
package loadbalancer

import (
    "sync/atomic"
)

type LoadBalancer struct {
    backends []string
    current  uint64
}

func NewLoadBalancer(backends []string) *LoadBalancer {
    return &LoadBalancer{
        backends: backends,
    }
}

// 轮询算法
func (lb *LoadBalancer) NextRoundRobin() string {
    current := atomic.AddUint64(&lb.current, 1)
    return lb.backends[current%uint64(len(lb.backends))]
}

// 加权轮询
type WeightedBackend struct {
    URL    string
    Weight int
}

type WeightedLoadBalancer struct {
    backends []WeightedBackend
    current  int
    weights  []int
}

func (lb *WeightedLoadBalancer) Next() string {
    total := 0
    for _, backend := range lb.backends {
        total += backend.Weight
    }
    
    current := lb.current % total
    lb.current++
    
    for _, backend := range lb.backends {
        if current < backend.Weight {
            return backend.URL
        }
        current -= backend.Weight
    }
    
    return lb.backends[0].URL
}
```

### 7.2 服务发现
```go
package discovery

import (
    "context"
    "github.com/hashicorp/consul/api"
)

type ServiceDiscovery struct {
    client *api.Client
}

func NewServiceDiscovery(address string) (*ServiceDiscovery, error) {
    config := api.DefaultConfig()
    config.Address = address
    
    client, err := api.NewClient(config)
    if err != nil {
        return nil, err
    }
    
    return &ServiceDiscovery{client: client}, nil
}

func (sd *ServiceDiscovery) Register(service *api.AgentServiceRegistration) error {
    return sd.client.Agent().ServiceRegister(service)
}

func (sd *ServiceDiscovery) Deregister(serviceID string) error {
    return sd.client.Agent().ServiceDeregister(serviceID)
}

func (sd *ServiceDiscovery) GetHealthyServices(serviceName string) ([]*api.ServiceEntry, error) {
    services, _, err := sd.client.Health().Service(serviceName, "", true, nil)
    return services, err
}

// 使用示例
func main() {
    sd, _ := NewServiceDiscovery("localhost:8500")
    
    // 注册服务
    service := &api.AgentServiceRegistration{
        ID:      "service-1",
        Name:    "my-service",
        Port:    8080,
        Address: "localhost",
        Check: &api.AgentServiceCheck{
            HTTP:     "http://localhost:8080/health",
            Interval: "10s",
            Timeout:  "5s",
        },
    }
    
    sd.Register(service)
    
    // 获取健康的服务实例
    services, _ := sd.GetHealthyServices("my-service")
    for _, service := range services {
        // 使用服务实例
        println(service.Service.Address)
    }
}
``` 