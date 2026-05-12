[返回首页](../README.md) / [返回上级](failure-modes.md)

# 生产排障手册

遇到高并发问题时，不要先改代码。先判断系统属于哪一种问题。

## 1. goroutine 持续上涨

优先检查：

- 是否有 goroutine 阻塞在 channel send/receive。
- 是否有下游调用没有超时。
- 是否有请求结束后后台任务还在运行。
- worker 是否因为队列不关闭而无法退出。

排查方法：

```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

操作时建议连续抓两次，中间间隔 30 秒：

```bash
curl -o goroutine-1.txt 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=2'
sleep 30
curl -o goroutine-2.txt 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=2'
```

重点看三件事：

- 总数是否持续上涨，停止流量后是否回落。
- 重复最多的堆栈在哪里。
- 堆栈停在 channel、锁、网络 IO、定时器，还是业务循环。

如果大量 goroutine 卡在同一个 channel send，通常是消费者退出、处理太慢，或者队列没有背压策略。如果大量卡在网络读写，通常是下游超时没设置好。如果大量卡在 `time.After` 或定时器相关调用，要检查是否在循环里频繁创建 timer。

## 2. p99 延迟升高

优先检查：

- 锁等待是否增加。
- 队列长度是否上涨。
- 下游延迟是否上涨。
- GC 是否变频繁。
- 是否发生重试风暴。

最佳实践是把延迟拆成阶段：入队等待时间、处理时间、下游调用时间、序列化时间。只看总耗时很难定位问题。

具体操作：

```bash
curl -o trace.out 'http://127.0.0.1:6060/debug/pprof/trace?seconds=10'
go tool trace ./your-server trace.out

curl -o cpu.pb.gz 'http://127.0.0.1:6060/debug/pprof/profile?seconds=30'
go tool pprof -http=:8080 ./your-server cpu.pb.gz

curl -o heap.pb.gz 'http://127.0.0.1:6060/debug/pprof/heap'
go tool pprof -http=:8080 ./your-server heap.pb.gz
```

判断顺序：

- trace 显示大量 goroutine 等待调度：并发过高或 CPU 已打满。
- trace 显示大量 network wait：优先看下游耗时、连接池、超时和重试。
- pprof CPU 显示 JSON、排序、正则、日志函数很宽：先减少热路径重复计算。
- heap 显示 `alloc_space` 很高：频繁临时分配会放大 GC，继续看 `allocs/op`。
- 指标显示队列长度和入队等待时间一起上涨：处理能力不足，或者 worker 被慢任务占住。

## 3. CPU 高但吞吐不上去

常见原因：

- goroutine 太多，调度成本升高。
- 锁竞争导致自旋和频繁唤醒。
- 热路径频繁 JSON 编解码。
- 日志量过大。
- 正则、排序、加密等 CPU 重操作在每个请求中重复执行。

处理顺序：

1. 用 CPU profile 找最热函数。
2. 先减少重复计算和同步日志。
3. 再考虑缓存、批量、算法优化。
4. 最后才考虑调 `GOMAXPROCS` 或做底层微优化。

具体操作：

```bash
curl -o cpu.pb.gz 'http://127.0.0.1:6060/debug/pprof/profile?seconds=30'
go tool pprof ./your-server cpu.pb.gz
```

进入 `pprof` 后常用命令：

```text
top
top -cum
list FunctionName
web
```

`top` 看函数自身 CPU，`top -cum` 看调用链累计 CPU，`list` 看具体代码行。优化时不要只改最上层入口函数，真正要改的是火焰图里最宽、并且在请求热路径上重复出现的那段逻辑。

## 4. 数据库被打爆

数据库通常是整个系统最共享、最稀缺的资源。保护数据库比保护单个服务更重要。

生产处理建议：

- 给数据库访问设置连接池上限。
- 对昂贵查询做接口级限流。
- 使用缓存和 singleflight 合并热点读。
- fan-out 查询要限制并发。
- 重试必须有退避和最大次数。
- 对写入高峰使用消息队列削峰，但队列也必须有积压监控。

如果数据库已经过载，第一步通常不是扩容，而是先止血：限流、降级、关闭非核心查询、减少重试。

具体要看这些数据：

- 应用侧连接池：`OpenConnections`、`InUse`、`Idle`、`WaitCount`、`WaitDuration`。
- 数据库侧：慢查询、锁等待、活跃连接数、CPU、IO、buffer/cache 命中率。
- 接口侧：fan-out 次数、单请求 SQL 数、重试次数、超时数。
- 缓存侧：命中率、热点 key、回源 QPS、singleflight 合并次数。

Go 的 `database/sql` 可以直接拿连接池状态：

```go
stats := db.Stats()
log.Printf(
	"db open=%d in_use=%d idle=%d wait=%d wait_duration=%s",
	stats.OpenConnections,
	stats.InUse,
	stats.Idle,
	stats.WaitCount,
	stats.WaitDuration,
)
```

如果 `WaitCount` 和 `WaitDuration` 持续上涨，说明应用已经在等连接池，不一定是 Go 代码慢；可能是数据库慢、连接池太小、慢查询太多，或者上游 fan-out 把并发放大了。

---

