[返回首页](../README.md) / [返回上级](part-05-engineering-design.md)

# 14. 错误处理

Go 的错误处理强调显式返回。生产系统里要区分错误类型。

```go
var ErrNotFound = errors.New("not found")
var ErrRateLimited = errors.New("rate limited")

func loadUser(id int64) error {
	return fmt.Errorf("load user %d: %w", id, ErrNotFound)
}
```

建议：

- 底层错误要 wrap 上下文。
- 边界层转换成稳定错误码。
- 不用字符串比较错误。
- 区分可重试和不可重试错误。
- 不要每一层都重复打日志。

## 14.1 给错误加上稳定语义

生产系统里，错误不仅给人看，也给程序判断。推荐定义稳定错误，再用 `errors.Is` 判断：

```go
var (
	ErrNotFound       = errors.New("not found")
	ErrBusy           = errors.New("busy")
	ErrInvalidRequest = errors.New("invalid request")
)

func validateTopN(n int) error {
	if n <= 0 || n > 1000 {
		return fmt.Errorf("%w: n must be in [1,1000]", ErrInvalidRequest)
	}
	return nil
}

func writeHTTPError(w http.ResponseWriter, err error) {
	switch {
	case errors.Is(err, ErrInvalidRequest):
		http.Error(w, err.Error(), http.StatusBadRequest)
	case errors.Is(err, ErrNotFound):
		http.Error(w, err.Error(), http.StatusNotFound)
	case errors.Is(err, ErrBusy):
		http.Error(w, err.Error(), http.StatusTooManyRequests)
	default:
		http.Error(w, "internal error", http.StatusInternalServerError)
	}
}
```

这里不要用字符串比较。字符串是给人看的，后续很容易变化；错误类型和错误码才是给程序判断的。

生产日志也要避免重复。通常在边界层打日志，例如 HTTP handler、消息消费入口、定时任务入口。底层函数只负责 wrap 错误上下文：

```go
func LoadBoard(ctx context.Context, id string) (*Board, error) {
	board, err := repo.FindBoard(ctx, id)
	if err != nil {
		return nil, fmt.Errorf("find board %s: %w", id, err)
	}
	return board, nil
}
```

如果每一层都打印一次同一个错误，线上日志会被放大，排障反而更困难。

---

