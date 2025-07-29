# 函数

## func OnceFunc（go1.21.0引入）

```go
func OnceFunc(f func()) func()
```

`OnceFunc`返回一个只会调用一次`f`的函数。由它返回的函数可以被并发地调用。

如果`f`发生了panic，`OnceFunc(f)`所返回的函数在每次调用时也会panic并带有相同的panic值。

## func OnceValue（go1.21.0引入）

```go
func OnceValue[T any](f func() T) func() T
```

`OnceValue`返回一个只会调用一次`f`并返回`f`的返回值的函数。由它返回的函数可以被并发地调用。

（译者注：`OnceValue(f)`所返回的函数满足

- 只会调用`f`一次
- 总是返回`f`的返回值（计算出首次结果之后即缓存供之后调用时使用）
- 是并发安全的

下文的`OnceValues`同理）

如果`f`发生了panic，`OnceValue(f)`所返回的函数在每次调用时也会panic并带有相同的panic值。

### 示例

以下示例通过`OnceValue`函数使得*高开销*的计算行为只进行一次，即使是在并发调用的情况下：

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    once := sync.OnceValue(func() int {
        sum := 0
        for i := 0; i < 1000; i++ {
            sum += i
        }
        fmt.Println("Computed once:", sum)
        return sum
    })
    done := make(chan bool)
    for i := 0; i < 10; i++ {
        go func() {
            const want = 499500
            got := once()
            if got != want {
                fmt.Println("want", want, "got", got)
            }
            done <- true
        }()
    }
    for i := 0; i < 10; i++ {
        <-done
    }
}
```

```text
Output:

Computed once: 499500
```

## func OnceValues（go1.21.0引入）

```go
func OnceValues[T1, T2 any](f func() (T1, T2)) func() (T1, T2)
```

`OnceValues`返回一个只会调用一次`f`并返回`f`的一对返回值的函数。由它返回的函数可以被并发地调用。

如果`f`发生了panic，`OnceValues(f)`所返回的函数在每次调用时也会panic并带有相同的panic值。

### 示例

以下示例通过调用`OnceValues`实现文件的单次读取：

```go
package main

import (
    "fmt"
    "os"
    "sync"
)

func main() {
    once := sync.OnceValues(func() ([]byte, error) {
        fmt.Println("Reading file once")
        return os.ReadFile("example_test.go")
    })
    done := make(chan bool)
    for i := 0; i < 10; i++ {
        go func() {
            data, err := once()
            if err != nil {
                fmt.Println("error:", err)
            }
            _ = data // Ignore the data for this example
            done <- true
        }()
    }
    for i := 0; i < 10; i++ {
        <-done
    }
}
```

```text
Output:

Reading file once
```

# 类型

## type Once

```go
type Once struct {
    // contains filtered or unexported fields
}
```

`Once`是一个确保某个操作恰好只执行一次的对象。

一个`Once`对象在首次使用后禁止复制。

在Go内存模型术语中，函数`f`的返回*同步发生于*任何对`once.Do(f)`的调用返回之前（译者注：即保证当`once.Do(f)`返回时，`f`任何对内存的修改都对当前Goroutine可见，无论有多少并发调用）。

### 示例

```go
import (
    "fmt"
    "sync"
)

func main() {
    var once sync.Once
    onceBody := func() {
        fmt.Println("Only once")
    }
    done := make(chan bool)
    for i := 0; i < 10; i++ {
        go func() {
            once.Do(onceBody)
            done <- true
        }()
    }
    for i := 0; i < 10; i++ {
        <-done
    }
}
```

```text
Output:

Only once
```

## func (*Once) Do

```go
func (o *Once) Do(f func())
```

对于该`Once`实例，`Do`方法在且仅在其被首次调用时调用函数`f`。换言之，令

```go
var once Once
```

如果`once.Do(f)`被多次调用，只有第一次会调用`f`，即使每次传入的`f`都不同。如果需要执行不同函数，就需要新的`Once`实例。

`Do`方法适用于需要恰好执行一次的初始化工作。因为`f`本身应当没有参数，有必要使用匿名函数来捕获实际由`Do`方法调用的函数的参数（译者注：即将原本有参数的待调用函数包装成无参的匿名函数，从而可以被需要参数是无参函数的`Do`方法调用，即所谓“闭包”）：

```go
config.once.Do(func() {
    config.init(filename)
})
```

因为对`Do`方法的调用返回总是在`f`返回之后，如果`f`中出现了对`Do`方法的调用，就会造成死锁（deadlock）。

如果`f`发生了panic，`Do`方法会将其视为已经返回；之后对`Do`的调用会直接返回而不会调用`f`。
