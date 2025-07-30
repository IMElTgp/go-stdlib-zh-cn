# 类型

## type WaitGroup

```go
type WaitGroup struct {
    // contains filtered or unexported fields
}
```

一个`WaitGroup`等待一组goroutine完成执行。主goroutine调用`WaitGroup.Add`以设置需要等待的goroutine的数量。之后，每个goroutine运行，并在完成时调用`WaitGroup.Done`方法。同时，可以使用`WaitGroup.Wait`方法来阻塞直至所有goroutine完成。

`WaitGroup`首次使用后禁止复制。

在Go内存模型术语中，`WaitGroup.Done`方法的一次调用*同步发生于*任何其所解除阻塞的`WaitGroup.Wait`方法调用之前。

### 示例

以下示例并发地抓取多个URL，并使用`WaitGroup`来阻塞直到所有抓取操作都已结束。

```go
package main

import (
    "sync"
)

type httpPkg struct{}

func (httpPkg) Get(url string) {}

var http httpPkg

func main() {
    var wg sync.WaitGroup
    var urls = []string{
        "http://www.golang.org/",
        "http://www.google.com/",
        "http://www.example.com/",
    }
    for _, url := range urls {
        // Increment the WaitGroup counter.
        wg.Add(1)
        // Launch a goroutine to fetch the URL.
        go func(url string) {
            // Decrement the counter when the goroutine completes.
            defer wg.Done()
            // Fetch the URL.
            http.Get(url)
        }(url)
    }
    // Wait for all HTTP fetches to complete.
    wg.Wait()
}
```

## func (wg *WaitGroup) Add

```go
func (wg *WaitGroup) Add(delta int)
```

`Add`方法向`WaitGroup`计数器增加`delta`的增量（可能是负数）。一旦计数器归零，所有阻塞于`WaitGroup.Wait`方法的goroutine都被释放。如果计数器变为负数，`Add`方法将会发生panic。

注意，当计数器为0时，调用增量为正的`Add`方法必须在`Wait`方法之前。当计数器为正时，可以在任何时候调用增量为正或负的`Add`方法。一般来说，这意味着对`Add`方法的调用应该在创建goroutine的语句或其他需要等待的事件之前执行。如果一个`WaitGroup`被重用以等待若干组相互独立的事件，新的`Add`方法调用必须发生在所有之前的`Wait`方法调用返回之后。详见[示例](#示例)。

## func (wg *WaitGroup) Done

```go
func (wg *WaitGroup) Done()
```

调用`Done`将使`WaitGroup`计数器的值减1。

## func (wg *WaitGroup) Wait

```go
func (wg *WaitGroup) Wait() 
```

调用`Wait`将会阻塞直至`WaitGroup`计数器归零。