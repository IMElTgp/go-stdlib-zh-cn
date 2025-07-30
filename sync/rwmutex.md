# 类型

## type Locker

```go
type Locker interface {
    Lock()
    Unlock()
}
```

`Locker`表示一个可加锁和解锁的对象。

## type RWMutex

```go
type RWMutex struct {
    // contains filtered or unexported fields
}
```

`RWMutex`是一个读/写互斥锁。该锁可由数量不限的读取者（reader）或单个写入者（writer）所持有。`RWMutex`的零值是一个未上锁的互斥锁（`Mutex`）。

`RWMutex`首次使用后禁止复制。

如果有Goroutine对已由一个或多个读取者持有的互斥锁调用了`RWMutex.Lock`方法，后续对`RWMutex.RLock`方法的并发调用会阻塞直至写入者获取（并释放）了该锁，以确保写入者最终能获取锁（译者注：防止写入者饿死）。注意，此设计禁止了递归读锁（read-locking）（译者注：已持有读锁的Goroutine再次尝试获取读锁将导致死锁）。`RWMutex.RLock`不能升级为`RWMutex.Lock`，`RWMutex.Lock`也不能降级为`RWMutex.RLock`。

在Go内存模型术语中，对`RWMutex.Unlock`方法的第`n`次调用*同步发生于*任何满足`m > n`的第`m`次`RWMutex.Lock`方法调用之前（和`Mutex`一样）。对于任何`RWMutex.RLock`调用，总存在`n`使得第`n`次`RWMutex.Unlock`调用*同步发生于*本次`RWMutex.RLock`调用之前，且对应的`RWMutex.RUnlock`调用*同步发生于*第`n+1`次`RWMutex.Lock`调用之前。

## func (*RWMutex) Lock

```go
func (rw *RWMutex) Lock()
```

`Lock`为rw加写锁（lock for writing）。如果该互斥锁已被加读锁或写锁，`Lock`会持续阻塞直到锁可用。

## func (*RWMutex) RLock

```go
func (rw *RWMutex) RLock()
```

`RLock`为rw加读锁。

`RLock`不应用于递归读锁；阻塞的`Lock`调用（译者注：写入者等待）将使得新读取者无法获取该锁。见[RWMutex类型](#type-rwmutex)。

## func (*RWMutex) RLocker

```go
func (rw *RWMutex) RLocker() Locker
```

`RLocker`返回一个[`Locker`](#type-locker)接口，并通过调用`rw.RLock`和`rw.RUnlock`来实现`Locker.Lock`和`Locker.Unlock`方法。

## func (*RWMutex) RUnlock

```go
func (rw *RWMutex) RUnlock()
```

`RUnlock`撤销单次`RWMutex.RLock`调用；它不会影响其他并发读取者。对未加读锁的`rw`调用`RUnlock`会导致运行时错误。

## func (*RWMutex) Trylock

```go
func (rw *RWMutex) TryLock() bool
```

`TryLock`尝试为`rw`加写锁并报告加锁是否成功。

注意，对`TryLock`的正确使用虽然确实存在但十分少见。`TryLock`的使用通常是特定场景下互斥锁深层设计问题的标志。

## func (*RWMutex) TryRLock

```go
func (rw *RWMutex) TryRLock() bool
```

`TryLock`尝试为`rw`加读锁并报告加锁是否成功。

注意，对`TryRLock`的正确使用虽然确实存在但十分少见。`TryRLock`的使用通常是特定场景下互斥锁深层设计问题的标志。

## func (*RWMutex) Unlock

```go
func (rw *RWMutex) Unlock()
```

`Unlock`将`rw`解除写锁。尝试对未加写锁的`rw`解除写锁将会导致运行时错误。

与互斥锁一致，加锁的[读写锁](#type-rwmutex)与某个特定的Goroutine并无关联。可以由一个Goroutine通过[`RWMutex.RLock`](#func-rwmutex-rlock)（[`RWMutex.Lock`](#func-rwmutex-lock)）锁定读写锁，并由另一个Goroutine通过[`RWMutex.RUnlock`](#func-rwmutex-runlock)（[`RWMutex.Unlock`](#func-rwmutex-unlock)）解锁。
