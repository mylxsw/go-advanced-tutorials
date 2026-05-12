[返回首页](../README.md) / [返回上级](appendix-a-concurrency-primitives.md)

# A.4 errgroup：并发等待 + 错误短路

`golang.org/x/sync/errgroup` 是 fan-out 场景最常用的工具。它封装了 `WaitGroup` + 错误收集 + context 取消。

```go
import "golang.org/x/sync/errgroup"

func Aggregate(ctx context.Context, userID int64) (Profile, error) {
	g, ctx := errgroup.WithContext(ctx)

	var (
		base    BaseInfo
		orders  []Order
		credits int64
	)

	g.Go(func() (err error) {
		base, err = loadBase(ctx, userID)
		return
	})
	g.Go(func() (err error) {
		orders, err = loadOrders(ctx, userID)
		return
	})
	g.Go(func() (err error) {
		credits, err = loadCredits(ctx, userID)
		return
	})

	if err := g.Wait(); err != nil {
		return Profile{}, err
	}
	return Profile{Base: base, Orders: orders, Credits: credits}, nil
}
```

要点：

- 任何子任务返回 error，`errgroup` 会自动 cancel 关联的 ctx，其它子任务应尽快退出。
- `golang.org/x/sync/errgroup` 提供的 `errgroup.SetLimit(n)` 可以限制同时运行的 goroutine 数（需要 `golang.org/x/sync` v0.1.0 及以上），非常适合批量 fan-out。
- 子任务内部必须检查 `ctx.Done()`，否则 cancel 没有意义。

