[返回首页](../README.md) / [返回上级](appendix-a-concurrency-primitives.md)

# A.5 singleflight：合并重复请求

`golang.org/x/sync/singleflight` 把相同 key 的并发请求合并成一次。典型场景是缓存击穿：热点 key 过期的瞬间，不要让成百上千的请求同时打到数据库。

```go
import "golang.org/x/sync/singleflight"

var g singleflight.Group

func GetUser(ctx context.Context, id int64) (*User, error) {
	key := fmt.Sprintf("user:%d", id)

	v, err, _ := g.Do(key, func() (any, error) {
		// 同一时刻相同 key 只有一个 goroutine 进入这里
		return loadUserFromDB(ctx, id)
	})
	if err != nil {
		return nil, err
	}
	return v.(*User), nil
}
```

要点：

- 合并的是“同一 key 的回源”，不是整个接口的调用。
- 一个请求失败，所有在等待的请求也会收到同一个错误。如果希望失败不传染，可以在回调里捕获错误。
- 对于延迟很高的上游（比如外部 API），`DoChan` 配合 `select` 更灵活，调用方可以自己决定是否放弃等待。

