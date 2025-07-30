# 类型

## type Cond

```go
type Cond struct {

    // L is held while observing or changing the condition
    L Locker
    // contains filtered or unexported fields
}
```

`Cond`（条件变量）实现了一种同步机制，作为goroutine进行事件发生的等待和通知的汇合点。

每个`Cond`实例都关联一个`Locker`对象`L`（通常是[`*Mutex`](mutex.md#type-mutex)或[`*RWMutex`](rwmutex.md#type-rwmutex)类型），且必须在修改条件变量所依赖的状态或调用`Cond.Wait`方法时持有该对象。

`Cond`首次使用后禁止复制。

在Go内存模型术语中，`Cond`保证所有对`Cond.Broadcast`或`Cond.Signal`的调用*同步发生于*任何由本次调用解除阻塞的`Wait`调用之前。

对于很多简单场景，比起使用`Cond`，用户使用信道（channel）更合适（`Broadcast`对应信道的关闭，`Signal`对应向信道发送值）。

对于`sync.Cond`替换方案的更多讨论，请参阅[Roberto Clapis's series on advanced concurrency patterns](https://blogtitle.github.io/categories/concurrency/)及[Bryan Mills's talk on concurrency patterns](https://drive.google.com/file/u/0/d/1nPdvhB0PutEJzdCq5ms6UI58dp50fcAN/view?pli=1)。

## func NewCond

```go
func NewCond(l Locker) *Cond
```

`NewCond`返回一个新的`Cond`对象，其关联的互斥锁为`l`。

## func (*Cond) Broadcast

```go
func (c *Cond) Broadcast()
```

`Broadcast`唤醒所有等待`c`的goroutine。

调用者在调用时可以但不强制要求持有`c.L`的互斥锁。

## func (*Cond) Signal

```go
func (c *Cond) Signal()
```

如果存在goroutine正在等待`c`，`Signal`将唤醒其中一个。

调用者在调用时可以但不强制要求持有`c.L`的互斥锁。

`Signal`不会影响goroutine调度的优先级；如果其他goroutine尝试锁定`c.L`，它们可能在“正在等待”的goroutine之前被唤醒。

## func (*Cond) Wait

```go
func (c *Cond) Wait()
```

`Wait`通过原子性地解锁`c.L`（译者注：解锁和挂起操作不可分割），并挂起正在调用的goroutine的执行。当后续恢复执行时，`Wait`在返回前重新锁定`c.L`。不同于在其他系统中，除非被`Cond.Broadcast`或`Cond.Signal`唤醒，`Wait`不会返回。

因为`Wait`等待期间`c.L`处于未上锁状态，当`Wait`返回时调用者通常不能认定条件为真。相反，调用者应当在循环中调用`Wait`：

```go
c.L.Lock()
for !condition() {
    c.Wait()
}
... make use of condition ...
c.L.Unlock()
```

（译者注：必须循环等待的原因有

- 虚假唤醒防御：操作系统可能无故唤醒线程
- 条件状态时效性：即使被合法唤醒，其他goroutine可能在当前goroutine重新加锁之前再次修改条件
- `Broadcast`副作用：当多个goroutine等待相同条件时，`Broadcast`会唤醒全部，但实际资源可能只满足部分goroutine）
