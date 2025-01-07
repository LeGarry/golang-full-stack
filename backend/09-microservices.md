# 微服务架构

## 1. 微服务基础

### 1.1 服务拆分
```go
// 领域驱动设计（DDD）示例
package user

// 用户聚合根
type User struct {
    ID        string
    Profile   Profile
    Addresses []Address
}

// 值对象
type Profile struct {
    Name     string
    Email    string
    Birthday time.Time
}

type Address struct {
    Street  string
    City    string
    Country string
}

// 仓储接口
type UserRepository interface {
    Save(user *User) error
    FindByID(id string) (*User, error)
}
```

### 1.2 服务通信
```go
// gRPC服务定义
syntax = "proto3";

service UserService {
    rpc CreateUser (CreateUserRequest) returns (CreateUserResponse);
    rpc GetUser (GetUserRequest) returns (GetUserResponse);
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message CreateUserResponse {
    string id = 1;
    string name = 2;
    string email = 3;
}

// gRPC服务实现
type UserServer struct {
    userService *service.UserService
}

func (s *UserServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserResponse, error) {
    user, err := s.userService.CreateUser(req.Name, req.Email)
    if err != nil {
        return nil, err
    }
    
    return &pb.CreateUserResponse{
        Id: user.ID,
        Name: user.Name,
        Email: user.Email,
    }, nil
}
```

## 2. 服务发现与注册

### 2.1 使用Consul
```go
import "github.com/hashicorp/consul/api"

func registerService() error {
    config := api.DefaultConfig()
    client, err := api.NewClient(config)
    if err != nil {
        return err
    }
    
    registration := &api.AgentServiceRegistration{
        ID:      "user-service-1",
        Name:    "user-service",
        Port:    8080,
        Address: "localhost",
        Check: &api.AgentServiceCheck{
            HTTP:     "http://localhost:8080/health",
            Interval: "10s",
            Timeout:  "5s",
        },
    }
    
    return client.Agent().ServiceRegister(registration)
}

func discoverService(serviceName string) (string, error) {
    config := api.DefaultConfig()
    client, err := api.NewClient(config)
    if err != nil {
        return "", err
    }
    
    services, _, err := client.Health().Service(serviceName, "", true, nil)
    if err != nil {
        return "", err
    }
    
    if len(services) == 0 {
        return "", fmt.Errorf("service not found")
    }
    
    service := services[0].Service
    return fmt.Sprintf("%s:%d", service.Address, service.Port), nil
}
```

## 3. 分布式配置

### 3.1 配置中心
```go
// 使用etcd作为配置中心
import "go.etcd.io/etcd/client/v3"

type ConfigCenter struct {
    client *clientv3.Client
}

func NewConfigCenter() (*ConfigCenter, error) {
    cli, err := clientv3.New(clientv3.Config{
        Endpoints:   []string{"localhost:2379"},
        DialTimeout: 5 * time.Second,
    })
    if err != nil {
        return nil, err
    }
    
    return &ConfigCenter{client: cli}, nil
}

func (c *ConfigCenter) WatchConfig(key string, callback func([]byte)) {
    rch := c.client.Watch(context.Background(), key)
    for wresp := range rch {
        for _, ev := range wresp.Events {
            callback(ev.Kv.Value)
        }
    }
}
```

## 4. 分布式追踪

### 4.1 Jaeger集成
```go
import (
    "github.com/opentracing/opentracing-go"
    jaeger "github.com/uber/jaeger-client-go"
)

func initJaeger(service string) (opentracing.Tracer, io.Closer) {
    cfg := jaeger.Configuration{
        ServiceName: service,
        Sampler: &jaeger.SamplerConfig{
            Type:  jaeger.SamplerTypeConst,
            Param: 1,
        },
        Reporter: &jaeger.ReporterConfig{
            LogSpans:           true,
            LocalAgentHostPort: "localhost:6831",
        },
    }
    
    tracer, closer, err := cfg.NewTracer()
    if err != nil {
        log.Fatal(err)
    }
    
    opentracing.SetGlobalTracer(tracer)
    return tracer, closer
}

// 中间件使用
func TracingMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        spCtx, _ := opentracing.GlobalTracer().Extract(
            opentracing.HTTPHeaders,
            opentracing.HTTPHeadersCarrier(c.Request.Header),
        )
        
        sp := opentracing.GlobalTracer().StartSpan(
            c.Request.URL.Path,
            opentracing.ChildOf(spCtx),
        )
        defer sp.Finish()
        
        c.Next()
    }
}
```

## 5. 熔断与限流

### 5.1 熔断器实现
```go
import "github.com/sony/gobreaker"

func newCircuitBreaker() *gobreaker.CircuitBreaker {
    return gobreaker.NewCircuitBreaker(gobreaker.Settings{
        Name:        "HTTP GET",
        MaxRequests: 3,
        Interval:    10 * time.Second,
        Timeout:     60 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 3 && failureRatio >= 0.6
        },
        OnStateChange: func(name string, from gobreaker.State, to gobreaker.State) {
            log.Printf("Circuit Breaker '%s' changed from '%s' to '%s'", name, from, to)
        },
    })
}

// 使用熔断器
func callService(cb *gobreaker.CircuitBreaker, url string) ([]byte, error) {
    body, err := cb.Execute(func() (interface{}, error) {
        resp, err := http.Get(url)
        if err != nil {
            return nil, err
        }
        defer resp.Body.Close()
        
        return ioutil.ReadAll(resp.Body)
    })
    
    if err != nil {
        return nil, err
    }
    
    return body.([]byte), nil
}
```

### 5.2 限流器实现
```go
import "golang.org/x/time/rate"

type RateLimiter struct {
    limiter *rate.Limiter
}

func NewRateLimiter(r rate.Limit, b int) *RateLimiter {
    return &RateLimiter{
        limiter: rate.NewLimiter(r, b),
    }
}

func (l *RateLimiter) Allow() bool {
    return l.limiter.Allow()
}

// 中间件使用
func RateLimitMiddleware(limiter *RateLimiter) gin.HandlerFunc {
    return func(c *gin.Context) {
        if !limiter.Allow() {
            c.JSON(429, gin.H{
                "error": "Too many requests",
            })
            c.Abort()
            return
        }
        c.Next()
    }
}
```

## 6. 消息队列集成

### 6.1 Kafka集成
```go
import "github.com/Shopify/sarama"

type KafkaProducer struct {
    producer sarama.SyncProducer
}

func NewKafkaProducer(brokers []string) (*KafkaProducer, error) {
    config := sarama.NewConfig()
    config.Producer.RequiredAcks = sarama.WaitForAll
    config.Producer.Retry.Max = 5
    config.Producer.Return.Successes = true
    
    producer, err := sarama.NewSyncProducer(brokers, config)
    if err != nil {
        return nil, err
    }
    
    return &KafkaProducer{producer: producer}, nil
}

func (p *KafkaProducer) SendMessage(topic string, key string, value []byte) error {
    msg := &sarama.ProducerMessage{
        Topic: topic,
        Key:   sarama.StringEncoder(key),
        Value: sarama.ByteEncoder(value),
    }
    
    _, _, err := p.producer.SendMessage(msg)
    return err
}

// 消费者实现
type KafkaConsumer struct {
    consumer sarama.Consumer
}

func (c *KafkaConsumer) Consume(topic string, handler func([]byte)) error {
    partitionConsumer, err := c.consumer.ConsumePartition(topic, 0, sarama.OffsetNewest)
    if err != nil {
        return err
    }
    
    for msg := range partitionConsumer.Messages() {
        handler(msg.Value)
    }
    
    return nil
}
```

## 7. 服务网格

### 7.1 Istio集成
```yaml
# 服务定义
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  ports:
  - port: 8080
    name: http
  selector:
    app: user-service

# 虚拟服务配置
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts:
  - user-service
  http:
  - route:
    - destination:
        host: user-service
        subset: v1
      weight: 90
    - destination:
        host: user-service
        subset: v2
      weight: 10
```

## 8. 监控与告警

### 8.1 Prometheus集成
```go
import "github.com/prometheus/client_golang/prometheus"

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

func init() {
    prometheus.MustRegister(requestDuration)
    prometheus.MustRegister(requestTotal)
}

// 监控中间件
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