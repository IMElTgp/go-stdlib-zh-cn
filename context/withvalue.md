# 类型

## func WithValue

```go
func WithValue(parent Context, key, val any) Context
```

`WithValue`返回一个关联父`Context`的派生`Context`。在派生`Context`中，关联`key`的值为`val`。

`Context`的`Value`机制仅用于跨进程和API边界的请求域数据，而不可用于向函数传递可选参数。

传入的`key`必须是可比较的，且不应是`string`等内置类型，以避免使用了`Context`的包之间的冲突。使用`WithValue`的用户应当自定义`key`的类型。为避免向`interface{}`复制时的内存分配，`Context`的`key`通常使用具体类型`struct{}`。或者，导出的`Context`的`key`变量的静态类型应采用指针或接口。

### 示例

以下示例展示了如何将一个值传递给`Context`以及在其存在的前提下如何检索：

```go
import (
    "context"
    "fmt"
)

func main() {
    type favContextKey string

    f := func(ctx context.Context, k favContextKey) {
        if v := ctx.Value(k); v != nil {
            fmt.Println("found value:", v)
            return
        }
        fmt.Println("key not found:", k)
    }

    k := favContextKey("language")
    ctx := context.WithValue(context.Background(), k, "Go")

    f(ctx, k)
    f(ctx, favContextKey("color"))

}
```

```text
Output:

found value: Go
key not found: color
```
