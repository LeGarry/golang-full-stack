# 安全最佳实践

## 1. 身份认证

### 1.1 JWT认证
```go
package auth

import (
    "time"
    "github.com/golang-jwt/jwt"
)

type Claims struct {
    UserID uint   `json:"user_id"`
    Role   string `json:"role"`
    jwt.StandardClaims
}

func GenerateToken(userID uint, role string, secret string) (string, error) {
    claims := Claims{
        UserID: userID,
        Role:   role,
        StandardClaims: jwt.StandardClaims{
            ExpiresAt: time.Now().Add(24 * time.Hour).Unix(),
            IssuedAt:  time.Now().Unix(),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(secret))
}

func ValidateToken(tokenString string, secret string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return []byte(secret), nil
    })

    if err != nil {
        return nil, err
    }

    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }

    return nil, jwt.ErrSignatureInvalid
}
```

### 1.2 OAuth2认证
```go
package oauth

import (
    "golang.org/x/oauth2"
    "golang.org/x/oauth2/google"
)

var (
    googleOauthConfig = &oauth2.Config{
        ClientID:     "your-client-id",
        ClientSecret: "your-client-secret",
        RedirectURL:  "http://localhost:8080/callback",
        Scopes: []string{
            "https://www.googleapis.com/auth/userinfo.email",
            "https://www.googleapis.com/auth/userinfo.profile",
        },
        Endpoint: google.Endpoint,
    }
)

func HandleGoogleLogin(c *fiber.Ctx) error {
    url := googleOauthConfig.AuthCodeURL("state")
    return c.Redirect(url)
}

func HandleGoogleCallback(c *fiber.Ctx) error {
    code := c.Query("code")
    token, err := googleOauthConfig.Exchange(c.Context(), code)
    if err != nil {
        return err
    }

    client := googleOauthConfig.Client(c.Context(), token)
    // 使用client获取用户信息
    return nil
}
```

## 2. 授权

### 2.1 RBAC实现
```go
package rbac

type Permission string

const (
    CreateUser Permission = "create:user"
    ReadUser   Permission = "read:user"
    UpdateUser Permission = "update:user"
    DeleteUser Permission = "delete:user"
)

type Role struct {
    Name        string
    Permissions []Permission
}

var (
    roles = map[string]Role{
        "admin": {
            Name: "admin",
            Permissions: []Permission{
                CreateUser,
                ReadUser,
                UpdateUser,
                DeleteUser,
            },
        },
        "user": {
            Name: "user",
            Permissions: []Permission{
                ReadUser,
            },
        },
    }
)

func HasPermission(role string, permission Permission) bool {
    if r, exists := roles[role]; exists {
        for _, p := range r.Permissions {
            if p == permission {
                return true
            }
        }
    }
    return false
}

// 权限中间件
func RequirePermission(permission Permission) fiber.Handler {
    return func(c *fiber.Ctx) error {
        claims := c.Locals("user").(*Claims)
        if !HasPermission(claims.Role, permission) {
            return c.Status(403).JSON(fiber.Map{
                "error": "Forbidden",
            })
        }
        return c.Next()
    }
}
```

## 3. 数据安全

### 3.1 密码加密
```go
package security

import (
    "golang.org/x/crypto/bcrypt"
)

func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// 密码验证器
type PasswordValidator struct {
    MinLength  int
    MaxLength  int
    MinDigits  int
    MinSymbols int
}

func (v *PasswordValidator) Validate(password string) error {
    if len(password) < v.MinLength {
        return fmt.Errorf("password must be at least %d characters long", v.MinLength)
    }
    if len(password) > v.MaxLength {
        return fmt.Errorf("password must not exceed %d characters", v.MaxLength)
    }
    // 更多验证规则...
    return nil
}
```

### 3.2 敏感数据加密
```go
package encryption

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/base64"
    "io"
)

type Encryptor struct {
    key []byte
}

func NewEncryptor(key string) *Encryptor {
    return &Encryptor{key: []byte(key)}
}

func (e *Encryptor) Encrypt(data []byte) (string, error) {
    block, err := aes.NewCipher(e.key)
    if err != nil {
        return "", err
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }

    nonce := make([]byte, gcm.NonceSize())
    if _, err = io.ReadFull(rand.Reader, nonce); err != nil {
        return "", err
    }

    ciphertext := gcm.Seal(nonce, nonce, data, nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

func (e *Encryptor) Decrypt(data string) ([]byte, error) {
    ciphertext, err := base64.StdEncoding.DecodeString(data)
    if err != nil {
        return nil, err
    }

    block, err := aes.NewCipher(e.key)
    if err != nil {
        return nil, err
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }

    nonceSize := gcm.NonceSize()
    if len(ciphertext) < nonceSize {
        return nil, errors.New("ciphertext too short")
    }

    nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]
    return gcm.Open(nil, nonce, ciphertext, nil)
}
```

## 4. API安全

### 4.1 请求验证
```go
package validation

import (
    "github.com/go-playground/validator/v10"
)

var validate = validator.New()

type CreateUserRequest struct {
    Username string `json:"username" validate:"required,min=3,max=32"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8,containsany=!@#$%^&*"`
}

func ValidateRequest(request interface{}) error {
    return validate.Struct(request)
}

// 验证中间件
func ValidationMiddleware(request interface{}) fiber.Handler {
    return func(c *fiber.Ctx) error {
        if err := c.BodyParser(request); err != nil {
            return c.Status(400).JSON(fiber.Map{
                "error": "Invalid request body",
            })
        }

        if err := ValidateRequest(request); err != nil {
            return c.Status(400).JSON(fiber.Map{
                "error": err.Error(),
            })
        }

        return c.Next()
    }
}
```

### 4.2 CORS配置
```go
package middleware

import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/cors"
)

func SetupCORS(app *fiber.App) {
    app.Use(cors.New(cors.Config{
        AllowOrigins:     "https://example.com",
        AllowMethods:     "GET,POST,PUT,DELETE,OPTIONS",
        AllowHeaders:     "Origin,Content-Type,Accept,Authorization",
        ExposeHeaders:    "Content-Length",
        AllowCredentials: true,
        MaxAge:           300,
    }))
}
```

## 5. 安全Headers

### 5.1 安全响应头
```go
package middleware

import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/helmet/v2"
)

func SetupSecurityHeaders(app *fiber.App) {
    app.Use(helmet.New())
    
    app.Use(func(c *fiber.Ctx) error {
        // CSP
        c.Set("Content-Security-Policy", "default-src 'self'")
        
        // HSTS
        c.Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        
        // X-Frame-Options
        c.Set("X-Frame-Options", "DENY")
        
        // X-Content-Type-Options
        c.Set("X-Content-Type-Options", "nosniff")
        
        // X-XSS-Protection
        c.Set("X-XSS-Protection", "1; mode=block")
        
        // Referrer-Policy
        c.Set("Referrer-Policy", "strict-origin-when-cross-origin")
        
        return c.Next()
    })
}
```

## 6. 安全中间件

### 6.1 Rate Limiting
```go
package middleware

import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/limiter"
    "time"
)

func RateLimiter() fiber.Handler {
    return limiter.New(limiter.Config{
        Max:        100,
        Expiration: 1 * time.Minute,
        KeyGenerator: func(c *fiber.Ctx) string {
            return c.IP()
        },
        LimitReached: func(c *fiber.Ctx) error {
            return c.Status(429).JSON(fiber.Map{
                "error": "Too many requests",
            })
        },
    })
}
```

### 6.2 SQL注入防护
```go
package middleware

import (
    "github.com/gofiber/fiber/v2"
    "regexp"
)

var sqlInjectionPattern = regexp.MustCompile(`(?i)(SELECT|INSERT|UPDATE|DELETE|DROP|UNION)`)

func SQLInjectionProtection() fiber.Handler {
    return func(c *fiber.Ctx) error {
        // 检查查询参数
        for _, param := range c.Queries() {
            if sqlInjectionPattern.MatchString(param) {
                return c.Status(400).JSON(fiber.Map{
                    "error": "Potential SQL injection detected",
                })
            }
        }

        // 检查请求体
        body := string(c.Body())
        if sqlInjectionPattern.MatchString(body) {
            return c.Status(400).JSON(fiber.Map{
                "error": "Potential SQL injection detected",
            })
        }

        return c.Next()
    }
}
```

## 7. 安全监控

### 7.1 安全日志
```go
package logging

import (
    "go.uber.org/zap"
    "time"
)

type SecurityLogger struct {
    logger *zap.Logger
}

func NewSecurityLogger() (*SecurityLogger, error) {
    logger, err := zap.NewProduction()
    if err != nil {
        return nil, err
    }

    return &SecurityLogger{
        logger: logger,
    }, nil
}

func (s *SecurityLogger) LogSecurityEvent(eventType string, details map[string]interface{}) {
    fields := []zap.Field{
        zap.String("event_type", eventType),
        zap.Time("timestamp", time.Now()),
    }

    for k, v := range details {
        fields = append(fields, zap.Any(k, v))
    }

    s.logger.Info("security_event", fields...)
}

// 使用示例
func SecurityLoggingMiddleware(securityLogger *SecurityLogger) fiber.Handler {
    return func(c *fiber.Ctx) error {
        start := time.Now()
        
        err := c.Next()
        
        details := map[string]interface{}{
            "ip":           c.IP(),
            "method":       c.Method(),
            "path":        c.Path(),
            "status":      c.Response().StatusCode(),
            "duration_ms": time.Since(start).Milliseconds(),
            "user_agent":  c.Get("User-Agent"),
        }
        
        if err != nil {
            details["error"] = err.Error()
            securityLogger.LogSecurityEvent("security_violation", details)
        } else {
            securityLogger.LogSecurityEvent("api_access", details)
        }
        
        return err
    }
}
```

### 7.2 安全审计
```go
package audit

import (
    "encoding/json"
    "time"
)

type AuditEvent struct {
    Timestamp   time.Time              `json:"timestamp"`
    UserID      string                 `json:"user_id"`
    Action      string                 `json:"action"`
    Resource    string                 `json:"resource"`
    ResourceID  string                 `json:"resource_id"`
    Changes     map[string]interface{} `json:"changes,omitempty"`
    IP          string                 `json:"ip"`
    UserAgent   string                 `json:"user_agent"`
    Status      string                 `json:"status"`
    Description string                 `json:"description"`
}

type AuditLogger interface {
    LogAuditEvent(event AuditEvent) error
}

type FileAuditLogger struct {
    filePath string
}

func (l *FileAuditLogger) LogAuditEvent(event AuditEvent) error {
    data, err := json.Marshal(event)
    if err != nil {
        return err
    }

    // 将审计日志写入文件
    // 实际实现应该考虑并发安全和文件轮转
    return nil
}

// 审计中间件
func AuditMiddleware(auditLogger AuditLogger) fiber.Handler {
    return func(c *fiber.Ctx) error {
        claims := c.Locals("user").(*Claims)
        
        event := AuditEvent{
            Timestamp:  time.Now(),
            UserID:     claims.UserID,
            Action:     c.Method(),
            Resource:   c.Path(),
            IP:        c.IP(),
            UserAgent: c.Get("User-Agent"),
        }
        
        err := c.Next()
        
        if err != nil {
            event.Status = "error"
            event.Description = err.Error()
        } else {
            event.Status = "success"
        }
        
        if err := auditLogger.LogAuditEvent(event); err != nil {
            // 处理审计日志错误
            return err
        }
        
        return err
    }
}
``` 