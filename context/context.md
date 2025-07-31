# 变量

```go
var Canceled = errors.New("context canceled")
```

`Canceled`是`Context`不因超出截止时间被取消时由[`Context.Err`]返回的错误。

```go
var DeadlineExceeded error = deadlineExceededError{}
```

`DeadlineExceeded`是`Context`因超出截止时间被取消时由[`Context.Err`]返回的错误。

# 函数

## func AfterFunc（go1.21.0引入）

```go
func AfterFunc(ctx Context, f func()) (stop func() bool)
```

`AfterFunc`在`ctx`被取消后于其拥有的goroutine上调用`f`。如果`ctx`已经被取消，则立即在其goroutine上调用`f`。

对于同一个`Context`的多个`AfterFunc`调用独立运行，彼此互不影响。

调用返回的`stop`函数将解除`ctx`和`f`之间的关联。当成功阻止`f`的执行时该函数返回`true`。若`ctx`已被取消且`f`已在其goroutine上启动运行，或`f`已停止执行，`stop`返回`false`。`stop`函数在返回前不会等待`f`完成执行。如果调用者需确认`f`是否完成运行，需要显式（自行）与`f`协调。

如果`ctx`实现了`AfterFunc(func()) func() bool`方法，`AfterFunc`将使用其调度调用。

### 示例（Cond）

以下示例使用`AfterFunc`定义一个在`sync.Cond`上等待的函数，该函数在`Context`被取消时停止等待：

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func main() {
    waitOnCond := func(ctx context.Context, cond *sync.Cond, conditionMet func() bool) error {
        stopf := context.AfterFunc(ctx, func() {
            // We need to acquire cond.L here to be sure that the Broadcast
            // below won't occur before the call to Wait, which would result
            // in a missed signal (and deadlock).
            cond.L.Lock()
            defer cond.L.Unlock()

            // If multiple goroutines are waiting on cond simutaneously,
            // we need to make sure we wake up exactly this one.
            // That means we need to Broadcast to all of the goroutines,
            // which will wake them all up.
            //
            // If there are N concurrent calls to waitOnCond, each of the goroutines
            // will spuriously wake up O(N) other goroutines that aren't ready yet,
            // so this will cause the overall CPU cost to be O(N²).
            cond.Broadcast()
        })
        defer stopf()

        // Since the wakeups are using Broadcast instead of Signal, this call to
        // Wait may unblock due to some other goroutine's context being canceled,
        // so to be sure that ctx is actually canceled we need to check it in a loop.
        for !conditionMet() {
            cond.Wait()
            if ctx.Err() != nil {
                return ctx.Err()
            }
        }

        return nil
    }

    cond := sync.NewCond(new(sync.Mutex))

    var wg sync.WaitGroup
    for i := 0; i < 4; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()

            ctx, cancel := context.WithTimeout(context.Background(), 1*time.Millisecond)
            defer cancel()

            cond.L.Lock()
            defer cond.L.Unlock()

            err := waitOnCond(ctx, cond, func() bool { return false })
            fmt.Println(err)
        }()
    }
    wg.Wait()

}
```

```text
Output:

context deadline exceeded
context deadline exceeded
context deadline exceeded
context deadline exceeded
```

### 示例（Connection）

以下示例使用`AfterFunc`定义了一个从`net.Conn`读取的函数，该函数在`Context`被取消时停止读取：

```go
package main() 

import (
    "context"
    "fmt"
    "net"
    "time"
)

func main() {
    readFromConn := func(ctx context.Context, conn net.Conn, b []byte) (n int, err error) {
        stopc := make(chan struct{})
        stop := context.AfterFunc(ctx, func() {
            conn.SetReadDeadline(time.Now())
            close(stopc)
        })
        n, err = conn.Read(b)
        if !stop() {
            // The AfterFunc was started.
            // Wait for it to complete, and reset the Conn's deadline.
            <-stopc
            conn.SetReadDeadline(time.Time{})
            return n, ctx.Err()
        }
        return n, err
    }

    listener, err := net.Listen("tcp", "localhost:0")
    if err != nil {
        fmt.Println(err)
        return 
    }
    defer conn.Close()

    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Millisecond)
    defer cancel()

    b := make([]byte, 1024)
    _, err = readFromConn(ctx, conn, b)
    fmt.Println(err)

}
```

```text
Output:

context deadline exceeded
```

### 示例（Merge）

以下示例使用`AfterFunc`定义了一个组合两个`Context`的取消信号的函数：

```go
package main

import (
    "context"
    "errors"
    "fmt"
)

func main() {
    // mergeCancel returns a context that contains the values of ctx,
    // and which is canceled when either ctx or cancelCtx is canceled.
    mergeCancel := func(ctx, cancelCtx context.Context) (context.Context, context.CancelFunc) {
        ctx, cancel := context.WithCancelCause(ctx)
        stop := context.AfterFunc(cancelCtx, func() {
            cancel(context.Cause(cancelCtx))
        })
        return ctx, func() {
            stop()
            cancel(context.Canceled)
        }
    }

    ctx1, cancel1 := context.WithCancelCause(context.Background())
    defer cancel1(errors.New("ctx1 canceled"))

    ctx2, cancel2 := context.WithCancelCause(context.Background())

    mergedCtx, mergedCancel := mergeCancel(ctx1, ctx2)
    defer mergeCancel()

    cancel2(errors.New("ctx2 canceled"))
    <-mergedCtx.Done()
    fmt.Println(context.Cause(mergedCtx))
    
}
```

## func Cause（go1.20引入）

```go
func Cause(c Context) error
```

`Cause`返回一个非nil的解释`c`被取消原因的错误。`Cause`由`c`或其任一父级`Context`的首次取消操作设置。如果这次取消是通过对`CancelCauseFunc(err)`的调用触发的，`Cause`将返回`err`。否则，`Cause(c)`将返回`c.Err()`的相同值。如果`c`尚未被取消，`Cause(c)`返回nil。

# 类型

## type Context

```go
type Context interface {
    // Deadline returns the time when work done on behalf of this context
    // should be canceled. Deadline returns ok==false when no deadline is
    // set. Successive calls to Deadline return the same results.
    Deadline() (deadline time.Time, ok bool)

    // Done returns a channel that's closed when work done on behalf of this
    // context should be canceled. Done may return nil if this context can
    // never be canceled. Successive calls to Done return the same value.
    // The close of the Done channel may happen asynchronously,
    // after the cancel function returns.
    //
    // WithCancel arranges for Done to be closed when cancel is called;
    // WithDeadline arranges for Done to be closed when the deadline
    // expires; WithTimeout arranges for Done to be closed when the timeout
    // elapses.
    //
    // Done is provided for use in select statements:
    //
    //  // Stream generates values with DoSomething and sends them to out
    //  // until DoSomething returns an error or ctx.Done is closed.
    //  func Stream(ctx context.Context, out chan<- Value) error {
    //      for {
    //          v, err := DoSomething(ctx)
    //          if err != nil {
    //              return err
    //          }
    //          select {
    //          case <-ctx.Done():
    //              return ctx.Err()
    //          case out <- v:
    //          }
    //      }
    //  }
    //
    // See https://blog.golang.org/pipelines for more examples of how to use
    // a Done channel for cancellation.
    Done() <-chan struct{}

    // If Done is not yet closed, Err returns nil.
    // If Done is closed, Err returns a non-nil error explaining why:
    // DeadlineExceeded if the context's deadline passed,
    // or Canceled if the context was canceled for some other reason.
    // After Err returns a non-nil error, successive calls to Err return the same error.
    Err() error

    // Value returns the value associated with this context for key, or nil
    // if no value is associated with key. Successive calls to Value with
    // the same key returns the same result.
    //
    // Use context values only for request-scoped data that transits
    // processes and API boundaries, not for passing optional parameters to
    // functions.
    //
    // A key identifies a specific value in a Context. Functions that wish
    // to store values in Context typically allocate a key in a global
    // variable then use that key as the argument to context.WithValue and
    // Context.Value. A key can be any type that supports equality;
    // packages should define keys as an unexported type to avoid
    // collisions.
    //
    // Packages that define a Context key should provide type-safe accessors
    // for the values stored using that key:
    //
    //  // Package user defines a User type that's stored in Contexts.
    //  package user
    //
    //  import "context"
    //
    //  // User is the type of value stored in the Contexts.
    //  type User struct {...}
    //
    //  // key is an unexported type for keys defined in this package.
    //  // This prevents collisions with keys defined in other packages.
    //  type key int
    //
    //  // userKey is the key for user.User values in Contexts. It is
    //  // unexported; clients use user.NewContext and user.FromContext
    //  // instead of using this key directly.
    //  var userKey key
    //
    //  // NewContext returns a new Context that carries value u.
    //  func NewContext(ctx context.Context, u *User) context.Context {
    //      return context.WithValue(ctx, userKey, u)
    //  }
    //
    //  // FromContext returns the User value stored in ctx, if any.
    //  func FromContext(ctx context.Context) (*User, bool) {
    //      u, ok := ctx.Value(userKey).(*User)
    //      return u, ok
    //  }
    Value(key any) any
}
```

一个`Context`（上下文）接口跨API边界传递截止时间（deadline）、取消信号以及其他关联值（value）。

`Context`的方法可由多个goroutine并发调用。

## func Background

```go
func Background() Context
```

`Background`返回一个非nil的空[`Context`](#type-context)接口。该接口不会被取消、不携带值也没有截止时间。`Background`通常由`main`函数、初始化逻辑和测试场景使用，或作为传入请求的顶级`Context`。

## func TODO

```go
func TODO() Context
```

`TODO`返回一个非nil的空`Context`。当不清楚应当使用哪个`Context`，或`Context`尚不可用（因为外围函数尚未扩展以接收`Context`参数）时，代码应使用`Context.TODO`。

## func WithoutCancel

```go
func WithoutCancel(parent Context) Context
```

`WithoutCancel`返回一个关联父`Context`的派生`Context`，该派生`Context`不会随父`Context`取消而取消。返回的`Context`不返回截止时间或取消错误，且其`Done`信道为nil。在返回的`Context`上调用[`Cause`](#func-cause)将返回nil（译者注：取消原因始终为nil）。
