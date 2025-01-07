# DevOps最佳实践

## 1. 持续集成 (CI)

### 1.1 代码质量控制
```go
// 使用golangci-lint进行代码检查
// .golangci.yml配置
linters:
  enable:
    - gofmt
    - golint
    - govet
    - errcheck
    - staticcheck
    - gosimple
    - ineffassign
    - deadcode

linters-settings:
  golint:
    min-confidence: 0.8
  govet:
    check-shadowing: true
  gofmt:
    simplify: true

run:
  deadline: 5m
  tests: true
  skip-dirs:
    - vendor/
    - third_party/
```

### 1.2 自动化测试
```go
// 单元测试示例
func TestUserService_Create(t *testing.T) {
    tests := []struct {
        name    string
        input   User
        wantErr bool
    }{
        {
            name: "valid user",
            input: User{
                Name:  "John Doe",
                Email: "john@example.com",
            },
            wantErr: false,
        },
        {
            name: "invalid email",
            input: User{
                Name:  "John Doe",
                Email: "invalid-email",
            },
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := userService.Create(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("Create() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}

// 集成测试示例
func TestIntegration_UserAPI(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    app := fiber.New()
    userHandler := NewUserHandler(userService)
    app.Post("/users", userHandler.Create)

    user := User{
        Name:  "John Doe",
        Email: "john@example.com",
    }

    body, _ := json.Marshal(user)
    req := httptest.NewRequest("POST", "/users", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    resp, err := app.Test(req)
    if err != nil {
        t.Fatalf("Failed to test request: %v", err)
    }

    if resp.StatusCode != http.StatusCreated {
        t.Errorf("Expected status code %d, got %d", http.StatusCreated, resp.StatusCode)
    }
}
```

## 2. 持续部署 (CD)

### 2.1 部署策略
```yaml
# 蓝绿部署配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:2.0.0

# 金丝雀部署配置
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.example.com
  http:
  - route:
    - destination:
        host: myapp-v1
        subset: v1
      weight: 90
    - destination:
        host: myapp-v2
        subset: v2
      weight: 10
```

### 2.2 自动化部署
```yaml
# Helm Chart配置
# Chart.yaml
apiVersion: v2
name: myapp
description: A Helm chart for MyApp
version: 0.1.0
appVersion: "1.0.0"

# values.yaml
replicaCount: 3
image:
  repository: myapp
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: myapp.example.com
      paths: ["/"]

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

## 3. 监控和告警

### 3.1 应用监控
```go
// Prometheus指标收集
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    RequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )

    RequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
)

// 中间件使用示例
func MetricsMiddleware() fiber.Handler {
    return func(c *fiber.Ctx) error {
        start := time.Now()
        
        err := c.Next()
        
        duration := time.Since(start).Seconds()
        status := c.Response().StatusCode()
        
        RequestsTotal.WithLabelValues(
            c.Method(),
            c.Path(),
            strconv.Itoa(status),
        ).Inc()
        
        RequestDuration.WithLabelValues(
            c.Method(),
            c.Path(),
        ).Observe(duration)
        
        return err
    }
}
```

### 3.2 告警配置
```yaml
# Prometheus告警规则
groups:
- name: myapp
  rules:
  - alert: HighErrorRate
    expr: |
      sum(rate(http_requests_total{status=~"5.."}[5m])) 
      / 
      sum(rate(http_requests_total[5m])) > 0.1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: High HTTP error rate
      description: Error rate is above 10% for the last 5 minutes

  - alert: HighLatency
    expr: |
      histogram_quantile(0.95, 
        sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
      ) > 2
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: High latency detected
      description: 95th percentile latency is above 2 seconds

# AlertManager配置
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'slack'

receivers:
- name: 'slack'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
    channel: '#alerts'
    send_resolved: true
```

## 4. 日志管理

### 4.1 日志收集
```go
// 结构化日志
package logger

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func NewLogger() (*zap.Logger, error) {
    config := zap.Config{
        Level:       zap.NewAtomicLevelAt(zap.InfoLevel),
        Development: false,
        Sampling: &zap.SamplingConfig{
            Initial:    100,
            Thereafter: 100,
        },
        Encoding: "json",
        EncoderConfig: zapcore.EncoderConfig{
            TimeKey:        "ts",
            LevelKey:      "level",
            NameKey:       "logger",
            CallerKey:     "caller",
            MessageKey:    "msg",
            StacktraceKey: "stacktrace",
            LineEnding:    zapcore.DefaultLineEnding,
            EncodeLevel:   zapcore.LowercaseLevelEncoder,
            EncodeTime:    zapcore.ISO8601TimeEncoder,
            EncodeDuration: zapcore.SecondsDurationEncoder,
            EncodeCaller:   zapcore.ShortCallerEncoder,
        },
        OutputPaths:      []string{"stdout"},
        ErrorOutputPaths: []string{"stderr"},
    }

    return config.Build()
}

// 使用示例
func main() {
    logger, _ := NewLogger()
    defer logger.Sync()

    logger.Info("Server starting",
        zap.String("environment", "production"),
        zap.Int("port", 8080),
    )
}
```

### 4.2 ELK配置
```yaml
# Logstash配置
input {
  beats {
    port => 5044
  }
}

filter {
  json {
    source => "message"
  }
  
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "myapp-%{+YYYY.MM.dd}"
  }
}

# Filebeat配置
filebeat.inputs:
- type: container
  paths:
    - /var/lib/docker/containers/*/*.log
  processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
        matchers:
        - logs_path:
            logs_path: "/var/log/containers/"

output.logstash:
  hosts: ["logstash:5044"]
```

## 5. 基础设施即代码 (IaC)

### 5.1 Terraform配置
```hcl
# 主配置
provider "kubernetes" {
  config_path = "~/.kube/config"
}

provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

# 创建命名空间
resource "kubernetes_namespace" "myapp" {
  metadata {
    name = "myapp"
  }
}

# 部署应用
resource "helm_release" "myapp" {
  name       = "myapp"
  namespace  = kubernetes_namespace.myapp.metadata[0].name
  repository = "https://charts.example.com"
  chart      = "myapp"
  version    = "1.0.0"

  values = [
    file("values.yaml")
  ]

  set {
    name  = "replicaCount"
    value = "3"
  }

  set {
    name  = "image.tag"
    value = var.image_tag
  }
}

# 配置变量
variable "image_tag" {
  description = "Docker image tag for the application"
  type        = string
}

# 输出值
output "load_balancer_ip" {
  value = data.kubernetes_service.myapp.status.0.load_balancer.0.ingress.0.ip
}
```

### 5.2 Ansible配置
```yaml
# 主playbook
- name: Deploy MyApp
  hosts: all
  become: true
  vars_files:
    - vars/main.yml

  tasks:
    - name: Install required packages
      apt:
        name: "{{ item }}"
        state: present
        update_cache: yes
      loop:
        - docker.io
        - python3-pip

    - name: Install Docker Python SDK
      pip:
        name: docker
        state: present

    - name: Create application directory
      file:
        path: "{{ app_dir }}"
        state: directory
        mode: '0755'

    - name: Copy application files
      copy:
        src: files/
        dest: "{{ app_dir }}"

    - name: Start application containers
      docker_compose:
        project_src: "{{ app_dir }}"
        state: present
      vars:
        project_name: myapp

# 变量文件
app_dir: /opt/myapp
docker_registry: registry.example.com
app_version: 1.0.0

# 清单文件
[production]
app-server-1 ansible_host=10.0.0.1
app-server-2 ansible_host=10.0.0.2
app-server-3 ansible_host=10.0.0.3

[staging]
staging-server ansible_host=10.0.0.4
```

## 6. 安全实践

### 6.1 容器安全
```yaml
# Pod安全上下文
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: myapp
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        cpu: "1"
        memory: "512Mi"
      requests:
        cpu: "0.5"
        memory: "256Mi"
```

### 6.2 密钥管理
```yaml
# Vault配置
apiVersion: vault.banzaicloud.com/v1alpha1
kind: Vault
metadata:
  name: vault
spec:
  size: 3
  image: vault:1.9.0
  bankVaultsImage: banzaicloud/bank-vaults:latest

  # Vault配置
  config:
    storage:
      file:
        path: "/vault/file"
    listener:
      tcp:
        address: "0.0.0.0:8200"
        tls_disable: true
    ui: true

  # 示例密钥注入
  externalConfig:
    policies:
      - name: myapp-policy
        rules: |
          path "secret/data/myapp/*" {
            capabilities = ["read", "list"]
          }
    auth:
      - type: kubernetes
        roles:
          - name: myapp
            bound_service_account_names: ["myapp"]
            bound_service_account_namespaces: ["default"]
            policies: ["myapp-policy"]
            ttl: 1h
```

## 7. 性能优化

### 7.1 性能测试
```go
// 压力测试示例
func BenchmarkAPI(b *testing.B) {
    app := fiber.New()
    handler := NewHandler()
    app.Post("/api/users", handler.CreateUser)

    user := User{
        Name:  "John Doe",
        Email: "john@example.com",
    }
    body, _ := json.Marshal(user)

    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            req := httptest.NewRequest("POST", "/api/users", bytes.NewReader(body))
            req.Header.Set("Content-Type", "application/json")
            _, err := app.Test(req)
            if err != nil {
                b.Fatal(err)
            }
        }
    })
}
```

### 7.2 性能监控
```yaml
# Grafana Dashboard配置
{
  "dashboard": {
    "id": null,
    "title": "Performance Dashboard",
    "panels": [
      {
        "title": "API Latency",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "50th percentile"
          }
        ]
      },
      {
        "title": "Memory Usage",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "container_memory_usage_bytes{container=\"myapp\"}",
            "legendFormat": "{{pod}}"
          }
        ]
      }
    ]
  }
}
``` 