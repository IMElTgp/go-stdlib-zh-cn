# 函数

## func WithTimeout

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

`WithTimeout`返回`WithDeadline(parent, time.Now().Add(timeout))`。

取消该`Context`将释放其关联资源，因此一旦运行在该[`Context`](context.md#type-context)内的操作完成，代码就应当立即调用`cancel`：

```go
func slowOperationWithTimeout(ctx context.Context) (Result, error) {
    ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
    defer cancel() // releases resourses if slowOperation completes before timeout elapses
    return slowOperation(ctx)
}
```

### 示例

以下示例传递带有超时（timeout）设置的`Context`以通知一个阻塞函数应在超时时间结束后终止当前任务：

```go
// Pass a context with a timeout to tell a blocking function that it 
// should abandon its work after the timeout elapses.
ctx, cancel := context.WithTimeout(context.Background(), shortDuration)
defer cancel()

select {
case <-neverReady:
    fmt.Println("ready")
case <-ctx.Done():
    fmt.Println(ctx.Err()) // prints "context deadline exceeded"
}
```

```text
Output:

context deadline exceeded
```

## func WithTimeoutCause

```go
func WithTimeoutCause(parent Context, timeout time.Duration, cause error) (Context, CancelFunc)
```

`WithTimeoutCause`类似[`WithTimeout`](#func-withtimeout)但还在超时时间结束后设置返回`Context`的取消原因。返回的[`CancelFunc`](withcancel.md#type-cancelfunc)调用时不会设置超时原因。
