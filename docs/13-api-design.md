[返回首页](../README.md) / [返回上级](part-05-engineering-design.md)

# 13. 接口设计

好的接口表达能力边界，而不是暴露实现细节。

```go
type Entry struct {
	Member string
	Score  int64
	Rank   int64
}

type Store interface {
	AddScore(ctx context.Context, board string, member string, delta int64) (int64, error)
	TopN(ctx context.Context, board string, n int) ([]Entry, error)
	Rank(ctx context.Context, board string, member string) (Entry, error)
}
```

接口设计建议：

- 接口尽量小。
- 第一个参数使用 `context.Context`。
- 错误语义要稳定。
- 不暴露内部锁、Map、Skiplist。
- 对分页、过滤、排序使用请求对象，避免参数爆炸。

## 13.1 用请求对象表达查询语义

当参数超过三四个时，继续堆函数参数会让接口难以演进。请求对象可以清楚表达默认值、边界和兼容性。

```go
type TopNRequest struct {
	Board     string
	Limit     int
	Offset    int
	Consistent bool
}

type TopNResponse struct {
	Entries   []Entry
	SnapshotAt time.Time
	Stale      bool
}

type Leaderboard interface {
	AddScore(ctx context.Context, board string, member string, delta int64) (Entry, error)
	TopN(ctx context.Context, req TopNRequest) (TopNResponse, error)
	Rank(ctx context.Context, board string, member string) (Entry, error)
}
```

`Consistent` 不一定意味着系统必须提供强一致实现，但它给接口留下了表达空间。比如当前版本只支持快照查询，可以在 `Consistent=true` 时返回 `ErrUnsupportedConsistency`，而不是以后破坏接口签名。

生产接口要把一致性语义写清楚：

- `AddScore` 返回的是写入后的实时分数，还是排队后的受理结果？
- `TopN` 是实时榜单，还是最近一次快照？
- `Rank` 找不到用户时返回 `ErrNotFound`，还是返回空排名？
- 分页过程中榜单刷新，是否保证同一页视图一致？

这些问题不提前定义，调用方会根据自己的理解使用接口，后续很难兼容。

---

