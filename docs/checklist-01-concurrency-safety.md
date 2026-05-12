[返回首页](../README.md) / [返回上级](high-concurrency-checklist.md)

# 1. 并发安全

- 所有共享状态是否有明确保护？最佳实践是给每个结构体写清楚“哪些字段由哪把锁保护”，复杂对象可以在字段注释中标明。
- 是否跑过 `go test -race`？并发数据结构、缓存、worker pool、排行榜更新路径都应该覆盖 race test。
- 是否存在 goroutine 泄漏？压测前后观察 goroutine 数，停止流量后应回落到稳定水平。
- channel 关闭责任是否明确？通常发送方关闭；多个发送方时由协调者关闭。
- 锁内是否调用外部服务？生产代码中应避免锁内 RPC、DB、文件 IO 和同步日志。

