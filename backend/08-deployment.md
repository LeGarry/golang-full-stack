# Golang 部署指南

## 1. 构建和打包

### 1.1 交叉编译
```bash
# 编译 Linux 64位
GOOS=linux GOARCH=amd64 go build -o app

# 编译 Windows 64位
GOOS=windows GOARCH=amd64 go build -o app.exe

# 编译 MacOS 64位
GOOS=darwin GOARCH=amd64 go build -o app
```

### 1.2 Docker打包
```dockerfile
# Dockerfile
FROM golang:1.19-alpine AS builder

WORKDIR /app
COPY . .
RUN go mod download
RUN CGO_ENABLED=0 GOOS=linux go build -o main

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .
COPY --from=builder /app/config ./config

EXPOSE 8080
CMD ["./main"]
```

## 2. 容器化部署

### 2.1 Docker Compose
```yaml
# docker-compose.yml
version: '3'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
      - DB_USER=postgres
      - DB_PASSWORD=password
      - DB_NAME=myapp
    depends_on:
      - db
      - redis
  
  db:
    image: postgres:13
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  redis:
    image: redis:6
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### 2.2 Kubernetes部署
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: myapp-config
              key: db_host
        resources:
          limits:
            cpu: "1"
            memory: "512Mi"
          requests:
            cpu: "0.5"
            memory: "256Mi"

---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

## 3. 持续集成/持续部署 (CI/CD)

### 3.1 GitHub Actions
```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: 1.19
    
    - name: Build
      run: go build -v ./...
    
    - name: Test
      run: go test -v ./...
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v2
      with:
        push: true
        tags: myapp:latest
    
    - name: Deploy to Kubernetes
      uses: steebchen/kubectl@v2
      with:
        config: ${{ secrets.KUBE_CONFIG_DATA }}
        command: apply -f k8s/
```

## 4. 监控和日志

### 4.1 Prometheus监控
```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "HTTP请求总数",
        },
        []string{"method", "endpoint"},
    )
    
    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Help: "HTTP请求处理时间",
        },
        []string{"method", "endpoint"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal)
    prometheus.MustRegister(httpRequestDuration)
}

func metricsMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        
        c.Next()
        
        duration := time.Since(start).Seconds()
        httpRequestsTotal.WithLabelValues(c.Request.Method, c.Request.URL.Path).Inc()
        httpRequestDuration.WithLabelValues(c.Request.Method, c.Request.URL.Path).Observe(duration)
    }
}
```

### 4.2 ELK日志收集
```go
import "github.com/olivere/elastic/v7"

type LogEntry struct {
    Timestamp time.Time `json:"timestamp"`
    Level     string    `json:"level"`
    Message   string    `json:"message"`
    Service   string    `json:"service"`
}

func setupElasticsearch() (*elastic.Client, error) {
    client, err := elastic.NewClient(
        elastic.SetURL("http://elasticsearch:9200"),
        elastic.SetSniff(false),
    )
    if err != nil {
        return nil, err
    }
    return client, nil
}

func logToElasticsearch(client *elastic.Client, entry LogEntry) error {
    _, err := client.Index().
        Index("logs").
        Type("_doc").
        BodyJson(entry).
        Do(context.Background())
    return err
}
```

## 5. 性能优化

### 5.1 性能分析
```go
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    
    // 应用主逻辑
}

// 使用方法:
// go tool pprof http://localhost:6060/debug/pprof/heap
// go tool pprof http://localhost:6060/debug/pprof/profile
```

### 5.2 负载均衡
```nginx
# nginx.conf
upstream myapp {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
    server 127.0.0.1:8083;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://myapp;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## 6. 安全配置

### 6.1 HTTPS配置
```go
func main() {
    r := gin.Default()
    // ... 路由配置
    
    // 配置TLS
    err := r.RunTLS(":443", "cert.pem", "key.pem")
    if err != nil {
        log.Fatal(err)
    }
}
```

### 6.2 安全头部配置
```go
func securityHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("Content-Security-Policy", "default-src 'self'")
        c.Header("Referrer-Policy", "strict-origin-when-cross-origin")
        c.Next()
    }
}
```

## 7. 灾备和恢复

### 7.1 数据备份
```go
func backupDatabase() error {
    cmd := exec.Command("pg_dump", 
        "-h", os.Getenv("DB_HOST"),
        "-U", os.Getenv("DB_USER"),
        "-d", os.Getenv("DB_NAME"),
        "-f", fmt.Sprintf("backup_%s.sql", time.Now().Format("20060102")))
    
    cmd.Env = append(os.Environ(),
        fmt.Sprintf("PGPASSWORD=%s", os.Getenv("DB_PASSWORD")))
    
    return cmd.Run()
}
```

### 7.2 自动恢复
```go
func autoRecover() gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("Panic: %v", err)
                
                // 通知管理员
                notifyAdmin(err)
                
                // 尝试恢复服务
                restartService()
                
                c.JSON(500, gin.H{
                    "error": "服务暂时不可用，请稍后重试",
                })
            }
        }()
        c.Next()
    }
}
```

## 8. 部署检查清单

### 8.1 上线前检查
1. 配置文件检查
2. 数据库连接测试
3. 日志配置验证
4. 性能测试结果
5. 安全漏洞扫描
6. SSL证书检查
7. 防火墙规则确认
8. 监控系统配置

### 8.2 部署流程
1. 备份当前版本
2. 停止服务
3. 部署新版本
4. 执行数据库迁移
5. 启动服务
6. 验证服务状态
7. 检查监控指标
8. 更新文档 