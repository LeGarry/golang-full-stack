# 项目结构

## 1. 标准项目结构

### 1.1 目录结构
```
project/
├── cmd/                    # 主要应用程序入口
│   └── server/
│       └── main.go        # 主程序入口
├── internal/              # 私有应用程序代码
│   ├── api/              # API处理器
│   ├── config/           # 配置管理
│   ├── middleware/       # HTTP中间件
│   ├── model/            # 数据模型
│   ├── repository/       # 数据访问层
│   ├── service/          # 业务逻辑层
│   └── utils/            # 工具函数
├── pkg/                  # 可以被外部应用程序使用的库代码
├── api/                  # OpenAPI/Swagger 规范，JSON schema 文件等
├── web/                  # Web应用程序特定的组件
├── configs/              # 配置文件模板或默认配置
├── scripts/              # 构建、安装、分析等操作的脚本
├── build/                # 打包和持续集成
├── deployments/          # IaaS，PaaS，系统和容器编排部署配置和模板
├── test/                 # 额外的外部测试应用程序和测试数据
├── docs/                 # 设计和用户文档
├── tools/               # 项目的支持工具
├── examples/            # 应用程序和/或公共库的示例
├── third_party/         # 外部辅助工具，分叉代码和其他第三方工具
├── vendor/              # 应用程序依赖项
├── .gitignore
├── go.mod
├── go.sum
├── LICENSE
├── Makefile
└── README.md
```

### 1.2 文件命名
```go
// 包名使用小写
package user

// 文件名使用小写下划线
// user_repository.go
// user_service.go
// error_handler.go

// 测试文件
// user_test.go
// user_service_test.go
```

## 2. 代码组织

### 2.1 包组织
```go
// internal/model/user.go
package model

type User struct {
    ID        uint      `json:"id"`
    Username  string    `json:"username"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

// internal/repository/user.go
package repository

type UserRepository interface {
    Create(user *model.User) error
    FindByID(id uint) (*model.User, error)
    Update(user *model.User) error
    Delete(id uint) error
}

// internal/service/user.go
package service

type UserService struct {
    repo repository.UserRepository
}

func NewUserService(repo repository.UserRepository) *UserService {
    return &UserService{repo: repo}
}

// internal/api/user.go
package api

type UserHandler struct {
    service *service.UserService
}

func NewUserHandler(service *service.UserService) *UserHandler {
    return &UserHandler{service: service}
}
```

### 2.2 依赖注入
```go
// internal/app/app.go
package app

type App struct {
    config     *config.Config
    db         *gorm.DB
    redis      *redis.Client
    router     *gin.Engine
    userRepo   repository.UserRepository
    userSvc    *service.UserService
    userHandler *api.UserHandler
}

func NewApp(config *config.Config) *App {
    app := &App{
        config: config,
    }
    
    app.initDB()
    app.initRedis()
    app.initRepositories()
    app.initServices()
    app.initHandlers()
    app.initRouter()
    
    return app
}

func (a *App) initRepositories() {
    a.userRepo = repository.NewUserRepository(a.db)
}

func (a *App) initServices() {
    a.userSvc = service.NewUserService(a.userRepo)
}

func (a *App) initHandlers() {
    a.userHandler = api.NewUserHandler(a.userSvc)
}
```

## 3. 配置管理

### 3.1 配置结构
```go
// internal/config/config.go
package config

type Config struct {
    Server   ServerConfig   `mapstructure:"server"`
    Database DatabaseConfig `mapstructure:"database"`
    Redis    RedisConfig    `mapstructure:"redis"`
    JWT      JWTConfig     `mapstructure:"jwt"`
}

type ServerConfig struct {
    Port         int    `mapstructure:"port"`
    Environment  string `mapstructure:"environment"`
    LogLevel     string `mapstructure:"log_level"`
}

type DatabaseConfig struct {
    Host     string `mapstructure:"host"`
    Port     int    `mapstructure:"port"`
    User     string `mapstructure:"user"`
    Password string `mapstructure:"password"`
    DBName   string `mapstructure:"dbname"`
}

// configs/config.yaml
server:
  port: 8080
  environment: development
  log_level: debug

database:
  host: localhost
  port: 5432
  user: postgres
  password: password
  dbname: myapp
```

### 3.2 配置加载
```go
// internal/config/loader.go
package config

import "github.com/spf13/viper"

func LoadConfig(path string) (*Config, error) {
    viper.SetConfigFile(path)
    viper.AutomaticEnv()
    
    if err := viper.ReadInConfig(); err != nil {
        return nil, err
    }
    
    var config Config
    if err := viper.Unmarshal(&config); err != nil {
        return nil, err
    }
    
    return &config, nil
}

// cmd/server/main.go
func main() {
    config, err := config.LoadConfig("configs/config.yaml")
    if err != nil {
        log.Fatal(err)
    }
    
    app := app.NewApp(config)
    app.Run()
}
```

## 4. 错误处理

### 4.1 错误定义
```go
// internal/errors/errors.go
package errors

import "fmt"

type AppError struct {
    Code    int
    Message string
    Err     error
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %v", e.Message, e.Err)
    }
    return e.Message
}

var (
    ErrNotFound = &AppError{
        Code:    404,
        Message: "Resource not found",
    }
    
    ErrUnauthorized = &AppError{
        Code:    401,
        Message: "Unauthorized access",
    }
    
    ErrInvalidInput = &AppError{
        Code:    400,
        Message: "Invalid input",
    }
)

// 错误包装
func WrapError(err error, message string) *AppError {
    return &AppError{
        Code:    500,
        Message: message,
        Err:     err,
    }
}
```

### 4.2 错误处理中间件
```go
// internal/middleware/error.go
package middleware

func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        
        if len(c.Errors) > 0 {
            err := c.Errors.Last()
            
            if appErr, ok := err.Err.(*errors.AppError); ok {
                c.JSON(appErr.Code, gin.H{
                    "error": appErr.Message,
                })
                return
            }
            
            c.JSON(500, gin.H{
                "error": "Internal Server Error",
            })
        }
    }
}
```

## 5. 测试组织

### 5.1 单元测试
```go
// internal/service/user_test.go
package service

import (
    "testing"
    "github.com/stretchr/testify/mock"
)

type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) FindByID(id uint) (*model.User, error) {
    args := m.Called(id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*model.User), args.Error(1)
}

func TestUserService_GetUser(t *testing.T) {
    mockRepo := new(MockUserRepository)
    service := NewUserService(mockRepo)
    
    user := &model.User{ID: 1, Username: "test"}
    mockRepo.On("FindByID", uint(1)).Return(user, nil)
    
    result, err := service.GetUser(1)
    assert.NoError(t, err)
    assert.Equal(t, user, result)
    mockRepo.AssertExpectations(t)
}
```

### 5.2 集成测试
```go
// test/integration/user_test.go
package integration

import (
    "testing"
    "net/http/httptest"
)

func TestUserAPI(t *testing.T) {
    app := setupTestApp()
    server := httptest.NewServer(app.Router())
    defer server.Close()
    
    t.Run("CreateUser", func(t *testing.T) {
        resp, err := http.Post(
            server.URL+"/api/users",
            "application/json",
            strings.NewReader(`{"username":"test","email":"test@example.com"}`),
        )
        assert.NoError(t, err)
        assert.Equal(t, 200, resp.StatusCode)
    })
}

func setupTestApp() *app.App {
    config := &config.Config{
        Database: config.DatabaseConfig{
            Host: "localhost",
            Port: 5432,
            User: "test",
            Password: "test",
            DBName: "test",
        },
    }
    
    return app.NewApp(config)
}
``` 