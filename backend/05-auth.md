# Golang 认证与授权

## 1. JWT认证

### 1.1 JWT基础
```go
import "github.com/golang-jwt/jwt"

// JWT声明结构
type Claims struct {
    UserId uint   `json:"userId"`
    Role   string `json:"role"`
    jwt.StandardClaims
}

// 生成JWT Token
func GenerateToken(userId uint, role string) (string, error) {
    claims := Claims{
        UserId: userId,
        Role:   role,
        StandardClaims: jwt.StandardClaims{
            ExpiresAt: time.Now().Add(24 * time.Hour).Unix(),
            IssuedAt:  time.Now().Unix(),
            Issuer:    "your-app-name",
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte("your-secret-key"))
}

// 验证JWT Token
func ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return []byte("your-secret-key"), nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, errors.New("invalid token")
}
```

## 2. Session认证

### 2.1 Session管理
```go
import "github.com/gorilla/sessions"

var store = sessions.NewCookieStore([]byte("your-secret-key"))

func SetSession(c *gin.Context, userId uint) error {
    session, _ := store.Get(c.Request, "session-name")
    session.Values["userId"] = userId
    return session.Save(c.Request, c.Writer)
}

func GetSession(c *gin.Context) (uint, error) {
    session, _ := store.Get(c.Request, "session-name")
    userId, ok := session.Values["userId"].(uint)
    if !ok {
        return 0, errors.New("未登录")
    }
    return userId, nil
}
```

## 3. OAuth2.0认证

### 3.1 OAuth2配置
```go
import "golang.org/x/oauth2"
import "golang.org/x/oauth2/google"

var oauth2Config = &oauth2.Config{
    ClientID:     "your-client-id",
    ClientSecret: "your-client-secret",
    RedirectURL:  "http://localhost:8080/callback",
    Scopes: []string{
        "https://www.googleapis.com/auth/userinfo.email",
    },
    Endpoint: google.Endpoint,
}

// 生成授权URL
func GetAuthURL() string {
    return oauth2Config.AuthCodeURL("state")
}

// 处理回调
func HandleCallback(code string) (*oauth2.Token, error) {
    return oauth2Config.Exchange(context.Background(), code)
}
```

## 4. 密码加密

### 4.1 bcrypt加密
```go
import "golang.org/x/crypto/bcrypt"

// 加密密码
func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

// 验证密码
func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

## 5. 权限控制

### 5.1 RBAC实现
```go
type Role struct {
    ID          uint   `gorm:"primaryKey"`
    Name        string `gorm:"unique"`
    Permissions []Permission `gorm:"many2many:role_permissions;"`
}

type Permission struct {
    ID          uint   `gorm:"primaryKey"`
    Name        string `gorm:"unique"`
    Description string
}

// 检查权限
func CheckPermission(userId uint, permissionName string) bool {
    var user User
    db.Preload("Role.Permissions").First(&user, userId)
    
    for _, permission := range user.Role.Permissions {
        if permission.Name == permissionName {
            return true
        }
    }
    return false
}
```

### 5.2 权限中间件
```go
func RequirePermission(permissionName string) gin.HandlerFunc {
    return func(c *gin.Context) {
        userId := c.GetUint("userId")
        if !CheckPermission(userId, permissionName) {
            c.JSON(http.StatusForbidden, gin.H{
                "error": "没有权限访问",
            })
            c.Abort()
            return
        }
        c.Next()
    }
}
```

## 6. 安全最佳实践

### 6.1 防止XSS攻击
```go
import "github.com/microcosm-cc/bluemonday"

func SanitizeInput(input string) string {
    p := bluemonday.UGCPolicy()
    return p.Sanitize(input)
}
```

### 6.2 防止CSRF攻击
```go
import "github.com/utrack/gin-csrf"

r.Use(csrf.Middleware(csrf.Options{
    Secret: "secret123",
    ErrorFunc: func(c *gin.Context) {
        c.JSON(400, gin.H{
            "error": "CSRF token mismatch",
        })
        c.Abort()
    },
}))
```

### 6.3 Rate Limiting
```go
import "golang.org/x/time/rate"

func RateLimitMiddleware(r rate.Limit) gin.HandlerFunc {
    limiter := rate.NewLimiter(r, int(r))
    return func(c *gin.Context) {
        if !limiter.Allow() {
            c.JSON(429, gin.H{
                "error": "请求太频繁",
            })
            c.Abort()
            return
        }
        c.Next()
    }
}
```

## 7. 多因素认证

### 7.1 TOTP实现
```go
import "github.com/pquerna/otp/totp"

// 生成密钥
func GenerateTOTPSecret() string {
    key, _ := totp.Generate(totp.GenerateOpts{
        Issuer:      "YourApp",
        AccountName: "user@example.com",
    })
    return key.Secret()
}

// 验证TOTP
func ValidateTOTP(secret, code string) bool {
    return totp.Validate(code, secret)
}
```

## 8. 登录注册流程

### 8.1 注册流程
```go
func Register(c *gin.Context) {
    var user User
    if err := c.ShouldBindJSON(&user); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    
    // 加密密码
    hashedPassword, _ := HashPassword(user.Password)
    user.Password = hashedPassword
    
    // 保存用户
    if err := db.Create(&user).Error; err != nil {
        c.JSON(500, gin.H{"error": "注册失败"})
        return
    }
    
    c.JSON(200, gin.H{"message": "注册成功"})
}
```

### 8.2 登录流程
```go
func Login(c *gin.Context) {
    var loginForm LoginForm
    if err := c.ShouldBindJSON(&loginForm); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    
    var user User
    if err := db.Where("email = ?", loginForm.Email).First(&user).Error; err != nil {
        c.JSON(401, gin.H{"error": "用户不存在"})
        return
    }
    
    if !CheckPassword(loginForm.Password, user.Password) {
        c.JSON(401, gin.H{"error": "密码错误"})
        return
    }
    
    // 生成Token
    token, _ := GenerateToken(user.ID, user.Role)
    c.JSON(200, gin.H{"token": token})
}
``` 