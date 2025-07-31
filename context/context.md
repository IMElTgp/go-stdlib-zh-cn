# 函数

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
