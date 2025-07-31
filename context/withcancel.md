# 类型

## type CancelCauseFunc

```go
type CancelCauseFunc func(cause error)
```

`CancelCauseFunc`类似[`CancelFunc`](#type-cancelfunc)但增加了对取消原因的设置。该取消原因可通过在被取消的`Context`或其派生`Context`之一上调用[`Cause`](context.md#func-causego120引入)检索到。

如果`Context`已经被取消，`CancelCauseFunc`不会设置取消原因。例如，`childContext`派生自`parentContext`：

- 如果`parentContext`因`cause1`被取消，此后`childContext`因`cause2`被取消，则`Cause(parentContext) == Cause(childContext) == cause1`。
- 如果`childContext`因`cause2`被取消，此后`parentContext`因`cause1`被取消，则`Cause(parentContext) == cause1`而`Cause(childContext) == cause2`。

## type CancelFunc

```go
type CancelFunc func()
```

`CancelFunc`通知一个操作终止目前任务。`CancelFunc`不会等待任务结束。`CancelFunc`可由多个goroutine同时调用。首次调用之后，对`CancelFunc`的后续调用无效。

# 函数

## func WithCancel

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

`WithCancel`返回一个关联父`Context`的拥有新的`Done`信道的派生`Context`。在返回的`cancel`函数被调用或父`Context`的`Done`信道被关闭（以先发生者为准）时，派生`Context`的`Done`信道被关闭。

取消此`Context`将释放其关联资源，因此一旦运行在该[`Context`](context.md#type-context)内的操作完成，代码就应当立即调用`cancel`。

### 示例

以下示例演示如何使用一个可取消`Context`防止goroutine泄露。在示例函数结束时，由`gen`函数启动的goroutine将正常退出而不泄露。

```go
import (
    "context"
    "fmt"
)

func main() {
    // gen generates integers in a separate goroutine ans
    // send them to the returned channel.
    // The callers of gen need to cancel the context once 
    // they are done consuming generated integers not to leak
    // the internal goroutine started by gen.
    gen := func(ctx context.Context) <-chan int {
        dst := make(chan int)
        n := 1
        go func() {
            for {
                select {
                case <-ctx.Done():
                    return // returning not to leak the goroutine
                case dst <- n:
                    n++
                }
            }
        }()
        return dst
    }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel() // cancel when we are finished consuming integers

    for n := range gen(ctx) {
        fmt.Println(n)
        if n == 5 {
            break
        }
    }
}
```

```text
Output:

1
2
3
4
5
```

## func WithCancelCause

```go
func WithCancelCause(parent Context) (ctx Context, cancel CancelCauseFunc)
```

`WithCancelCause`类似`WithCancel`，但返回`CancelCauseFunc`而非`CancelFunc`。调用传入非nil错误（即取消原因（`cause`））的`cancel`函数将把该错误记录在返回的`ctx`中，之后可通过`Cause(ctx)`检索。调用传入nil错误的`cancel`函数将把原因设置为`Canceled`。

用例：

```go
ctx, cancel := context.WithCancelCause(parent)
cancel(myError)
ctx.Err() // returns context.Canceled
context.Cause(ctx) // returns myError
```
