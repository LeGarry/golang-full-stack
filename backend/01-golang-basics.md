# Golang 基础知识

## 1. 语言特性
- 静态类型
- 编译型语言
- 垃圾回收
- 并发支持（goroutine和channel）
- 简单易学

## 2. 基础语法
### 2.1 变量声明
```go
var name string
name := "golang"
const PI = 3.14
```

### 2.2 数据类型
- 基本类型：bool, string, int, float64
- 复合类型：array, slice, map, struct
- 引用类型：pointer, interface, channel

### 2.3 控制结构
```go
// if 语句
if condition {
    // code
} else {
    // code
}

// for 循环
for i := 0; i < 10; i++ {
    // code
}

// switch 语句
switch value {
case 1:
    // code
default:
    // code
}
```

## 3. 函数
```go
func add(x, y int) int {
    return x + y
}

// 多返回值
func divide(x, y float64) (float64, error) {
    if y == 0 {
        return 0, errors.New("除数不能为零")
    }
    return x / y, nil
}
```

## 4. 结构体和方法
```go
type Person struct {
    Name string
    Age  int
}

func (p Person) SayHello() string {
    return fmt.Sprintf("Hello, I'm %s", p.Name)
}
```

## 5. 接口
```go
type Animal interface {
    Speak() string
}

type Dog struct {
    Name string
}

func (d Dog) Speak() string {
    return "Woof!"
}
```

## 6. 并发编程
### 6.1 Goroutine
```go
go func() {
    // 并发执行的代码
}()
```

### 6.2 Channel
```go
ch := make(chan int)
go func() {
    ch <- 42
}()
value := <-ch
```

## 7. 错误处理
```go
func readFile(filename string) error {
    if _, err := os.Open(filename); err != nil {
        return fmt.Errorf("打开文件失败: %v", err)
    }
    return nil
}
```

## 8. 包管理
- go mod init
- go get
- go mod tidy
- go mod vendor

## 9. 最佳实践
1. 使用 gofmt 格式化代码
2. 错误处理要明确
3. 接口要小而精确
4. 优先使用组合而非继承
5. 充分利用并发特性 