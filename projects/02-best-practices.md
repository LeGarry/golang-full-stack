# Go 最佳实践

## 1. 代码规范

### 1.1 命名规范
```go
// 包名使用小写单个单词
package user

// 接口名使用动词+er
type Reader interface {
    Read(p []byte) (n int, err error)
}

// 结构体名使用驼峰命名
type UserService struct {
    repo UserRepository
}

// 方法名使用驼峰命名
func (s *UserService) CreateUser(user *User) error {
    return s.repo.Create(user)
}

// 常量使用全大写下划线分隔
const (
    STATUS_ACTIVE   = "active"
    STATUS_INACTIVE = "inactive"
)

// 错误变量使用Err前缀
var (
    ErrNotFound = errors.New("resource not found")
    ErrInvalid  = errors.New("invalid input")
)
```

### 1.2 注释规范
```go
// Package user 提供用户相关的功能
package user

// User 表示系统中的用户实体
type User struct {
    ID       uint      `json:"id"`
    Username string    `json:"username"`
    Email    string    `json:"email"`
    Password string    `json:"-"`
}

// CreateUser 创建新用户
// 如果用户名已存在，返回 ErrDuplicateUsername
// 如果邮箱已存在，返回 ErrDuplicateEmail
func (s *UserService) CreateUser(user *User) error {
    // 验证用户输入
    if err := s.validateUser(user); err != nil {
        return err
    }
    
    // 创建用户
    return s.repo.Create(user)
}
```

## 2. 错误处理

### 2.1 错误包装
```go
import "github.com/pkg/errors"

func (s *UserService) GetUser(id uint) (*User, error) {
    user, err := s.repo.FindByID(id)
    if err != nil {
        // 包装错误并添加上下文
        return nil, errors.Wrapf(err, "failed to get user with id %d", id)
    }
    return user, nil
}

// 错误判断
func IsNotFound(err error) bool {
    return errors.Is(err, ErrNotFound)
}

// 自定义错误类型
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}
```

### 2.2 错误处理模式
```go
// 使用类型断言处理特定错误
func handleError(err error) {
    switch e := err.(type) {
    case *ValidationError:
        log.Printf("Validation error: %v", e)
    case *NotFoundError:
        log.Printf("Not found error: %v", e)
    default:
        log.Printf("Unknown error: %v", e)
    }
}

// 使用defer处理资源清理
func processFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return errors.Wrap(err, "failed to open file")
    }
    defer f.Close()
    
    // 处理文件
    return nil
}
```

## 3. 并发处理

### 3.1 Goroutine管理
```go
// 使用WaitGroup等待goroutine完成
func processItems(items []Item) error {
    var wg sync.WaitGroup
    errChan := make(chan error, len(items))
    
    for _, item := range items {
        wg.Add(1)
        go func(item Item) {
            defer wg.Done()
            if err := processItem(item); err != nil {
                errChan <- err
            }
        }(item)
    }
    
    wg.Wait()
    close(errChan)
    
    // 收集错误
    for err := range errChan {
        if err != nil {
            return err
        }
    }
    
    return nil
}

// 使用context控制goroutine生命周期
func worker(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // 执行工作
        }
    }
}
```

### 3.2 并发安全
```go
// 使用互斥锁保护共享资源
type SafeCounter struct {
    mu    sync.Mutex
    count map[string]int
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count[key]++
}

// 使用原子操作
type AtomicCounter struct {
    count int64
}

func (c *AtomicCounter) Inc() {
    atomic.AddInt64(&c.count, 1)
}

func (c *AtomicCounter) Get() int64 {
    return atomic.LoadInt64(&c.count)
}
```

## 4. 性能优化

### 4.1 内存优化
```go
// 使用对象池
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func processLargeData(data []byte) error {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()
    
    // 使用buffer处理数据
    return nil
}

// 避免不必要的内存分配
func processString(s string) string {
    // 预分配内存
    var builder strings.Builder
    builder.Grow(len(s))
    
    for _, ch := range s {
        builder.WriteRune(ch)
    }
    
    return builder.String()
}
```

### 4.2 CPU优化
```go
// 使用批处理
func processBatch(items []Item) error {
    const batchSize = 1000
    
    for i := 0; i < len(items); i += batchSize {
        end := i + batchSize
        if end > len(items) {
            end = len(items)
        }
        
        if err := processBatchItems(items[i:end]); err != nil {
            return err
        }
    }
    
    return nil
}

// 避免不必要的计算
var (
    computeOnce sync.Once
    result      int
)

func expensiveComputation() int {
    computeOnce.Do(func() {
        // 执行昂贵的计算
        result = compute()
    })
    return result
}
```

## 5. 测试实践

### 5.1 单元测试
```go
// 表驱动测试
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name  string
        email string
        want  bool
    }{
        {"valid email", "test@example.com", true},
        {"invalid email", "test@", false},
        {"empty email", "", false},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := ValidateEmail(tt.email)
            if got != tt.want {
                t.Errorf("ValidateEmail() = %v, want %v", got, tt.want)
            }
        })
    }
}

// 使用测试辅助函数
func TestUserService(t *testing.T) {
    // 设置测试环境
    cleanup := setupTest(t)
    defer cleanup()
    
    // 运行测试
    t.Run("CreateUser", testCreateUser)
    t.Run("UpdateUser", testUpdateUser)
}
```

### 5.2 基准测试
```go
// 基准测试
func BenchmarkProcessLargeData(b *testing.B) {
    data := generateLargeData()
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        processLargeData(data)
    }
}

// 并行基准测试
func BenchmarkProcessLargeDataParallel(b *testing.B) {
    data := generateLargeData()
    b.ResetTimer()
    
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            processLargeData(data)
        }
    })
}
```

## 6. 文档实践

### 6.1 代码文档
```go
// Package service 提供业务逻辑层的实现
package service

// UserService 处理用户相关的业务逻辑
//
// UserService 实现了用户的CRUD操作，以及其他业务逻辑，
// 如用户认证、权限验证等。
type UserService struct {
    repo UserRepository
    cache Cache
}

// NewUserService 创建一个新的UserService实例
//
// repo 参数提供数据访问层的实现
// cache 参数提供缓存实现
func NewUserService(repo UserRepository, cache Cache) *UserService {
    return &UserService{
        repo:  repo,
        cache: cache,
    }
}
```

### 6.2 示例代码
```go
// Example_createUser 展示如何创建新用户
func Example_createUser() {
    service := NewUserService(repo, cache)
    user := &User{
        Username: "test",
        Email:    "test@example.com",
    }
    
    err := service.CreateUser(user)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Printf("Created user: %v", user)
    // Output: Created user: {ID:1 Username:test Email:test@example.com}
}
``` 