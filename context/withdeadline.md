# 函数

## func WithDeadline

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
```

`WithDeadline`返回一个关联父`Context`的派生`Context`，其截止时间不会晚于`d`。如果其父`Context`的截止时间已早于`d`，`WithDeadline(parent, d)`语义上等价于`parent`（父`Context`）。返回的`[Context.Done]`信道在超过截止时间、调用返回的`cancel`函数或父`Context`的`Done`信道被关闭（以先发生的为准）时关闭。

取消此`Context`将释放其关联资源，因此一旦运行在该[`Context`](context.md#type-context)内的操作完成，代码就应当立即调用`cancel`。

### 示例

以下示例传递一个带有任意设定截止时间的`Context`以通知阻塞函数一旦获得处理能力就应立即放弃当前任务。

```go
d := time.Now().Add(shortDuration)
ctx, cancel := context.WithDeadline(context.Background(), d)

// Even though ctx will be expired, it is good practice to call its
// cancellation function in any case. Failure to do so may keep the
// context and its parent alive longer than necessary.
defer cancel()

select {
case <-neverReady:
    fmt.Println("ready")
case <-ctx.Done():
    fmt.Println(ctx.Err())
}
```

```text
Output:

context deadline exceeded
```

## func WithDeadlineCause

```go
func WithDeadlineCause(parent Context, d time.Time, cause error) (Context, CancelFunc)
```

`WithDeadlineCause`类似[`WithDeadline`](#func-withdeadline)但还在超过截止时间时设置返回`Context`的取消原因。返回的[`CancelFunc`](withcancel.md#type-cancelfunc)调用时不会设置取消原因。
