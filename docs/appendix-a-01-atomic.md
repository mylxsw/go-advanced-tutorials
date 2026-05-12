[返回首页](../README.md) / [返回上级](appendix-a-concurrency-primitives.md)

# A.1 atomic：无锁计数与标志位

`sync/atomic` 提供底层原子操作。适合单个数值的并发读写，例如计数器、开关、版本号。Go 1.19 引入了面向对象的 `atomic.Int64`、`atomic.Pointer[T]` 等类型，推荐优先使用。

```go
type Counter struct {
	n atomic.Int64
}

func (c *Counter) Inc()          { c.n.Add(1) }
func (c *Counter) Value() int64  { return c.n.Load() }
```

使用 atomic 的几点注意：

- 只适合保护单个字段。多个字段构成不变量时，仍需锁或原子替换整个结构。
- `atomic.Value` 和 `atomic.Pointer[T]` 适合整体替换一个不可变快照，是实现 copy-on-write 的基础。
- 不要对 `int` 或 `int64` 裸字段混用 atomic 和普通读写，会被 race detector 检测为竞争。

