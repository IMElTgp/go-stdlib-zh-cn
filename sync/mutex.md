# 类型

## type Mutex

```go
type Mutex struct {
    // contains filtered or unexported fields
}
```

`Mutex`是一种互斥的锁（mutual exclusive lock，互斥锁）。`Mutex`的零值是一个未上锁的`Mutex`。

`Mutex`首次使用后禁止复制。

在Go内存模型术语中，对`Mutex.Unlock`方法的第`n`次调用*同步发生于*任何满足`m > n`的第`m`次`Mutex.Lock`方法调用之前。对`Mutex.Trylock`方法的一次成功调用等价于调用一次`Lock`方法。对`Mutex.Trylock`方法的一次失败调用将不会建立任何“同步发生于”的关系。

## func (m *Mutex) Lock

```go
func (m *Mutex) Lock()
```

`Lock`函数锁定`m`。如果该互斥锁已被锁定，调用的goroutine将会阻塞直至解锁。

## func (m *Mutex) TryLock

```go
func (m *Mutex) TryLock() bool
```

`TryLock`尝试锁定`m`并报告锁定是否成功。

注意，对`TryLock`的正确使用虽然确实存在但十分少见。`TryLock`的使用通常是特定场景下互斥锁深层设计问题的标志。

## func (m *Mutex) Unlock

```go
func (m *Mutex) Unlock()
```

`Unlock`将`m`解锁。尝试对未加锁的互斥锁解锁将会导致运行时错误。

加锁的互斥锁与某个特定的goroutine并无关联。可以由一个goroutine锁定互斥锁，并由另一个goroutine解锁。
