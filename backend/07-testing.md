# Golang 测试指南

## 1. 单元测试

### 1.1 基本测试
```go
// user_test.go
package user

import "testing"

func TestCreateUser(t *testing.T) {
    user := &User{
        Name:  "张三",
        Email: "zhangsan@example.com",
    }
    
    err := user.Create()
    if err != nil {
        t.Errorf("创建用户失败: %v", err)
    }
    
    if user.ID == 0 {
        t.Error("用户ID不应该为0")
    }
}
```

### 1.2 表驱动测试
```go
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name  string
        email string
        want  bool
    }{
        {"有效邮箱", "test@example.com", true},
        {"无效邮箱-无@", "testexample.com", false},
        {"无效邮箱-无域名", "test@", false},
        {"无效邮箱-特殊字符", "test#@example.com", false},
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
```

## 2. 性能测试

### 2.1 基准测试
```go
func BenchmarkCreateUser(b *testing.B) {
    for i := 0; i < b.N; i++ {
        user := &User{
            Name:  "张三",
            Email: fmt.Sprintf("test%d@example.com", i),
        }
        user.Create()
    }
}
```

### 2.2 并行基准测试
```go
func BenchmarkCreateUserParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            user := &User{
                Name:  "张三",
                Email: fmt.Sprintf("test%d@example.com", i),
            }
            user.Create()
            i++
        }
    })
}
```

## 3. 模拟测试

### 3.1 接口模拟
```go
type UserRepository interface {
    Create(user *User) error
    FindByID(id uint) (*User, error)
}

type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) Create(user *User) error {
    args := m.Called(user)
    return args.Error(0)
}

func (m *MockUserRepository) FindByID(id uint) (*User, error) {
    args := m.Called(id)
    return args.Get(0).(*User), args.Error(1)
}

func TestUserService_Create(t *testing.T) {
    mockRepo := new(MockUserRepository)
    service := NewUserService(mockRepo)
    
    user := &User{Name: "张三"}
    mockRepo.On("Create", user).Return(nil)
    
    err := service.Create(user)
    assert.NoError(t, err)
    mockRepo.AssertExpectations(t)
}
```

### 3.2 HTTP测试
```go
func TestUserHandler_Create(t *testing.T) {
    router := gin.Default()
    router.POST("/users", CreateUser)
    
    user := &User{
        Name:  "张三",
        Email: "test@example.com",
    }
    
    body, _ := json.Marshal(user)
    req, _ := http.NewRequest("POST", "/users", bytes.NewBuffer(body))
    req.Header.Set("Content-Type", "application/json")
    
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)
    
    assert.Equal(t, 200, w.Code)
    
    var response map[string]interface{}
    json.Unmarshal(w.Body.Bytes(), &response)
    assert.Equal(t, "success", response["message"])
}
```

## 4. 集成测试

### 4.1 数据库测试
```go
func TestUserRepository_Integration(t *testing.T) {
    // 设置测试数据库
    db, err := gorm.Open(sqlite.Open(":memory:"))
    if err != nil {
        t.Fatalf("连接数据库失败: %v", err)
    }
    defer db.Close()
    
    db.AutoMigrate(&User{})
    
    repo := NewUserRepository(db)
    
    // 测试创建用户
    user := &User{Name: "张三"}
    err = repo.Create(user)
    assert.NoError(t, err)
    assert.NotZero(t, user.ID)
    
    // 测试查询用户
    found, err := repo.FindByID(user.ID)
    assert.NoError(t, err)
    assert.Equal(t, user.Name, found.Name)
}
```

### 4.2 API集成测试
```go
func TestAPI_Integration(t *testing.T) {
    app := NewApp()
    server := httptest.NewServer(app.Router)
    defer server.Close()
    
    // 测试创建用户
    user := &User{Name: "张三"}
    resp, err := http.Post(
        server.URL+"/users",
        "application/json",
        bytes.NewBuffer(mustMarshal(user)),
    )
    assert.NoError(t, err)
    assert.Equal(t, 200, resp.StatusCode)
    
    // 测试获取用户
    var created User
    mustUnmarshal(resp.Body, &created)
    
    resp, err = http.Get(server.URL + "/users/" + fmt.Sprint(created.ID))
    assert.NoError(t, err)
    assert.Equal(t, 200, resp.StatusCode)
}
```

## 5. 测试覆盖率

### 5.1 运行测试覆盖率
```bash
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### 5.2 测试覆盖率要求
```go
// 确保测试覆盖关键路径
func TestUserService_Create_Error(t *testing.T) {
    tests := []struct {
        name    string
        user    *User
        wantErr bool
    }{
        {"正常情况", &User{Name: "张三"}, false},
        {"名字为空", &User{}, true},
        {"邮箱无效", &User{Name: "张三", Email: "invalid"}, true},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := service.Create(tt.user)
            if (err != nil) != tt.wantErr {
                t.Errorf("Create() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

## 6. 测试工具和框架

### 6.1 断言库
```go
import "github.com/stretchr/testify/assert"

func TestUser_Validate(t *testing.T) {
    user := &User{Name: "张三"}
    
    assert.NoError(t, user.Validate())
    assert.Equal(t, "张三", user.Name)
    assert.NotEmpty(t, user.Name)
    assert.True(t, user.IsValid())
}
```

### 6.2 测试套件
```go
type UserTestSuite struct {
    suite.Suite
    db   *gorm.DB
    repo UserRepository
}

func (s *UserTestSuite) SetupTest() {
    s.db = setupTestDB()
    s.repo = NewUserRepository(s.db)
}

func (s *UserTestSuite) TearDownTest() {
    s.db.Close()
}

func (s *UserTestSuite) TestCreateUser() {
    user := &User{Name: "张三"}
    err := s.repo.Create(user)
    s.NoError(err)
    s.NotZero(user.ID)
}

func TestUserSuite(t *testing.T) {
    suite.Run(t, new(UserTestSuite))
}
```

## 7. 测试最佳实践

### 7.1 测试文件组织
```
project/
  ├── user/
  │   ├── user.go
  │   ├── user_test.go
  │   ├── repository.go
  │   ├── repository_test.go
  │   ├── service.go
  │   └── service_test.go
  └── test/
      ├── integration/
      └── fixtures/
```

### 7.2 测试命名规范
```go
// 功能测试
func TestUser_Create(t *testing.T)
func TestUser_Update(t *testing.T)

// 边界条件测试
func TestUser_Create_EmptyName(t *testing.T)
func TestUser_Create_InvalidEmail(t *testing.T)

// 性能测试
func BenchmarkUser_Create(b *testing.B)
func BenchmarkUser_Find(b *testing.B)
```

### 7.3 测试环境配置
```go
func setupTestEnv() (*gorm.DB, func()) {
    // 设置测试环境
    db, err := gorm.Open(sqlite.Open(":memory:"))
    if err != nil {
        panic(err)
    }
    
    // 迁移数据库
    db.AutoMigrate(&User{})
    
    // 返回清理函数
    cleanup := func() {
        db.Close()
    }
    
    return db, cleanup
} 