# 函数

## func OnceFunc（go1.21.0引入）

```go
func OnceFunc(f func()) func()
```

OnceFunc返回一个只会调用一次f的函数。由它返回的函数可以被并发地调用。

如果f发生了panic，OnceFunc(f)所返回的函数在每次调用时也会panic并带有相同的panic值。

## func OnceValue（go1.21.0引入）

```go
func OnceValue[T any](f func() T) func() T
```

OnceValue返回一个只会调用一次f并返回f的返回值的函数。由它返回的函数可以被并发地调用。

（译者注：OnceValue(f)所返回的函数满足

- 只会调用f一次
- 总是返回f的返回值（计算出首次结果之后即缓存供之后调用时使用）
- 是并发安全的

下文的OnceValues同理）

如果f发生了panic，OnceValue(f)所返回的函数在每次调用时也会panic并带有相同的panic值。

### 示例

以下示例通过OnceValue函数使得*高开销*的计算行为只进行一次，即使是在并发调用的情况下：

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

OnceValues返回一个只会调用一次f并返回f的一对返回值的函数。由它返回的函数可以被并发地调用。

如果f发生了panic，OnceValues(f)所返回的函数在每次调用时也会panic并带有相同的panic值。

### 示例

以下示例通过调用OnceValues实现文件的单次读取：

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
