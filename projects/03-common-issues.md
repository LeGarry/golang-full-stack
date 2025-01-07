# 常见问题与解决方案

## 1. 内存问题

### 1.1 内存泄漏
```go
// 问题：goroutine泄漏
func leakyGoroutine() {
    ch := make(chan int)
    go func() {
        val := <-ch  // 通道永远不会收到数据，goroutine泄漏
    }()
}

// 解决方案：使用context或done通道
func fixedGoroutine(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            // 处理数据
        case <-ctx.Done():
            return
        }
    }()
}

// 问题：defer在循环中可能导致内存泄漏
func leakyLoop() {
    for {
        f, _ := os.Open("file.txt")
        defer f.Close()  // defer会在函数返回时才执行
        // 处理文件
    }
}

// 解决方案：在循环内关闭资源
func fixedLoop() {
    for {
        func() {
            f, _ := os.Open("file.txt")
            defer f.Close()
            // 处理文件
        }()
    }
}
```

### 1.2 内存占用
```go
// 问题：大对象分配
func loadLargeData() []byte {
    data := make([]byte, 100*1024*1024)  // 一次性分配大内存
    return data
}

// 解决方案：使用缓冲区
func streamLargeData(w io.Writer) error {
    buf := make([]byte, 32*1024)  // 32KB缓冲区
    for {
        n, err := source.Read(buf)
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
        w.Write(buf[:n])
    }
    return nil
}

// 使用对象池
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 32*1024)
    },
}

func processWithPool() {
    buf := bufferPool.Get().([]byte)
    defer bufferPool.Put(buf)
    // 使用buf处理数据
}
```

## 2. 并发问题

### 2.1 竞态条件
```go
// 问题：map并发访问
var cache = make(map[string]string)

func updateCache(key, value string) {
    cache[key] = value  // 并发访问map会导致panic
}

// 解决方案：使用sync.Map或互斥锁
var syncCache sync.Map

func updateSyncCache(key, value string) {
    syncCache.Store(key, value)
}

// 使用互斥锁
type SafeCache struct {
    mu    sync.RWMutex
    cache map[string]string
}

func (c *SafeCache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.cache[key] = value
}

func (c *SafeCache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.cache[key]
    return val, ok
}
```

### 2.2 死锁问题
```go
// 问题：嵌套锁导致死锁
type Resource struct {
    mu sync.Mutex
}

func (r *Resource) Deadlock(other *Resource) {
    r.mu.Lock()
    defer r.mu.Unlock()
    other.mu.Lock()    // 可能导致死锁
    defer other.mu.Unlock()
}

// 解决方案：按固定顺序获取锁
func (r *Resource) Fixed(other *Resource) {
    if uintptr(unsafe.Pointer(r)) < uintptr(unsafe.Pointer(other)) {
        r.mu.Lock()
        other.mu.Lock()
    } else {
        other.mu.Lock()
        r.mu.Lock()
    }
    defer r.mu.Unlock()
    defer other.mu.Unlock()
}
```

## 3. 性能问题

### 3.1 CPU占用
```go
// 问题：频繁的字符串拼接
func buildString(items []string) string {
    var result string
    for _, item := range items {
        result += item  // 每次拼接都会创建新的字符串
    }
    return result
}

// 解决方案：使用strings.Builder
func buildStringOptimized(items []string) string {
    var builder strings.Builder
    builder.Grow(len(items) * 8)  // 预分配内存
    for _, item := range items {
        builder.WriteString(item)
    }
    return builder.String()
}

// 问题：过度使用反射
func setField(obj interface{}, name string, value interface{}) {
    reflect.ValueOf(obj).Elem().FieldByName(name).Set(reflect.ValueOf(value))
}

// 解决方案：使用代码生成或类型断言
type User struct {
    Name string
    Age  int
}

func (u *User) SetField(name string, value interface{}) error {
    switch name {
    case "Name":
        if v, ok := value.(string); ok {
            u.Name = v
            return nil
        }
    case "Age":
        if v, ok := value.(int); ok {
            u.Age = v
            return nil
        }
    }
    return fmt.Errorf("invalid field or value type")
}
```

### 3.2 I/O性能
```go
// 问题：频繁的小数据I/O
func writeData(w io.Writer, data []byte) {
    for _, b := range data {
        w.Write([]byte{b})  // 每次写入一个字节
    }
}

// 解决方案：使用缓冲I/O
func writeDataBuffered(w io.Writer, data []byte) {
    bufW := bufio.NewWriter(w)
    bufW.Write(data)
    bufW.Flush()
}

// 批量处理
func processBatch(items []Item) error {
    const batchSize = 1000
    for i := 0; i < len(items); i += batchSize {
        end := i + batchSize
        if end > len(items) {
            end = len(items)
        }
        if err := db.Create(&items[i:end]).Error; err != nil {
            return err
        }
    }
    return nil
}
```

## 4. 配置问题

### 4.1 配置管理
```go
// 问题：硬编码配置
var (
    dbHost = "localhost"
    dbPort = 5432
    dbUser = "postgres"
)

// 解决方案：使用配置管理
type Config struct {
    Database struct {
        Host     string `yaml:"host"`
        Port     int    `yaml:"port"`
        User     string `yaml:"user"`
        Password string `yaml:"password"`
    } `yaml:"database"`
    
    Redis struct {
        Host     string `yaml:"host"`
        Port     int    `yaml:"port"`
        Password string `yaml:"password"`
    } `yaml:"redis"`
}

func LoadConfig() (*Config, error) {
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")
    
    if err := viper.ReadInConfig(); err != nil {
        return nil, err
    }
    
    var config Config
    if err := viper.Unmarshal(&config); err != nil {
        return nil, err
    }
    
    return &config, nil
}
```

### 4.2 环境变量
```go
// 问题：环境变量管理
func getEnv(key, fallback string) string {
    if value, ok := os.LookupEnv(key); ok {
        return value
    }
    return fallback
}

// 解决方案：使用环境变量结构
type EnvConfig struct {
    DBHost     string `envconfig:"DB_HOST" default:"localhost"`
    DBPort     int    `envconfig:"DB_PORT" default:"5432"`
    DBUser     string `envconfig:"DB_USER" required:"true"`
    DBPassword string `envconfig:"DB_PASSWORD" required:"true"`
}

func LoadEnvConfig() (*EnvConfig, error) {
    var config EnvConfig
    if err := envconfig.Process("", &config); err != nil {
        return nil, err
    }
    return &config, nil
}
```

## 5. 测试问题

### 5.1 测试覆盖
```go
// 问题：难以测试的代码
func hardToTest() error {
    if time.Now().Hour() < 12 {
        return doMorningTask()
    }
    return doAfternoonTask()
}

// 解决方案：依赖注入
type Clock interface {
    Now() time.Time
}

type RealClock struct{}

func (RealClock) Now() time.Time {
    return time.Now()
}

type Task struct {
    clock Clock
}

func (t *Task) Execute() error {
    if t.clock.Now().Hour() < 12 {
        return doMorningTask()
    }
    return doAfternoonTask()
}

// 测试
type MockClock struct {
    current time.Time
}

func (m MockClock) Now() time.Time {
    return m.current
}

func TestTask(t *testing.T) {
    mock := MockClock{current: time.Date(2023, 1, 1, 10, 0, 0, 0, time.UTC)}
    task := &Task{clock: mock}
    err := task.Execute()
    assert.NoError(t, err)
}
```

### 5.2 集成测试
```go
// 问题：数据库测试
func TestWithDB(t *testing.T) {
    // 直接使用生产数据库
    db.Create(&User{Name: "test"})
}

// 解决方案：使用测试容器
func TestWithTestContainer(t *testing.T) {
    ctx := context.Background()
    req := testcontainers.ContainerRequest{
        Image:        "postgres:13",
        ExposedPorts: []string{"5432/tcp"},
        Env: map[string]string{
            "POSTGRES_PASSWORD": "test",
            "POSTGRES_DB":      "testdb",
        },
    }
    
    postgres, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: req,
        Started:         true,
    })
    if err != nil {
        t.Fatal(err)
    }
    defer postgres.Terminate(ctx)
    
    // 运行测试
    port, _ := postgres.MappedPort(ctx, "5432")
    db := setupTestDB(port.Int())
    // 执行测试...
}
``` 