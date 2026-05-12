[返回首页](../README.md) / [返回上级](appendix-a-concurrency-primitives.md)

# A.2 sync.Once：只执行一次的初始化

`sync.Once` 保证某段代码在进程生命周期内只执行一次，线程安全且开销很小。常用于懒加载单例、初始化连接池、注册 metrics。

```go
var (
	once   sync.Once
	client *http.Client
)

func HTTPClient() *http.Client {
	once.Do(func() {
		client = &http.Client{Timeout: 3 * time.Second}
	})
	return client
}
```

要点：

- 如果 `Do` 中的函数 panic，`sync.Once` 仍认为“已执行”，后续调用不会重试。需要时要在 `Do` 内自己 recover 并重置。
- Go 1.21 引入了 `sync.OnceFunc`、`sync.OnceValue`、`sync.OnceValues`，写法更简洁，推荐在新代码里使用。

