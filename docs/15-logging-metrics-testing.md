[返回首页](../README.md) / [返回上级](part-05-engineering-design.md)

# 15. 日志、指标与测试

高并发系统不能靠猜。至少要观察：

| 指标 | 说明 |
| --- | --- |
| QPS | 当前吞吐 |
| p99 延迟 | 尾延迟 |
| 错误率 | 系统健康度 |
| 队列长度 | 是否积压 |
| goroutine 数 | 是否泄漏 |
| 内存和 GC | 是否有分配压力 |
| 限流数 | 是否过载 |

常用命令：

```bash
go test ./...
go test -race ./...
go test -bench=. -benchmem ./...
```

并发结构一定要跑 race test。性能优化一定要跑 benchmark。

## 15.1 指标要和系统边界对应

指标不是越多越好，而是要覆盖关键边界。以 worker pool 为例，至少要有：

```text
pool_submit_total{result="ok|busy|timeout"}
pool_queue_length
pool_task_duration_seconds
pool_worker_busy
pool_panic_total
```

这些指标对应系统的核心问题：

- `submit_total` 看入口是否被拒绝。
- `queue_length` 看是否积压。
- `task_duration` 看处理是否变慢。
- `worker_busy` 看 worker 是否打满。
- `panic_total` 看任务是否有未处理异常。

日志适合记录离散事件，指标适合观察趋势，trace 适合定位单次请求路径，pprof 适合分析 CPU、内存和 goroutine。不要试图用日志替代所有观测手段。

## 15.2 并发测试示例

并发数据结构要同时测正确性和 race。下面是分片计数器的测试思路：

```go
func TestShardedCounterConcurrentAdd(t *testing.T) {
	c := NewShardedCounter()

	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				c.Add("hot", 1)
			}
		}()
	}
	wg.Wait()

	if got := c.Value("hot"); got != 100000 {
		t.Fatalf("value=%d, want=100000", got)
	}
}
```

运行：

```bash
go test -race ./...
```

`-race` 不能证明没有并发 bug，但能抓住大量真实问题。对核心并发结构，race test 应该是 CI 的一部分。

## 15.3 benchmark 要回答具体问题

不要为了 benchmark 而 benchmark。好的 benchmark 应该回答一个选择题：`Mutex` 和 `RWMutex` 哪个更适合这个读写比例？分片数 32 和 128 哪个更好？快照查询是否真的比实时排序快？

```go
func BenchmarkShardedCounterAdd(b *testing.B) {
	c := NewShardedCounter()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			c.Add("user:123", 1)
		}
	})
}
```

如果所有 goroutine 都写同一个 key，这个 benchmark 测的是热点竞争；如果 key 随机分布，测的是分片扩展性。两者都重要，但含义完全不同。写 benchmark 时要让数据分布接近真实流量。

## 15.4 性能分析工具怎么用

性能优化不要从“感觉哪里慢”开始，而要从可重复的测量开始。Go 项目里最常用的工具组合是：

| 工具 | 解决的问题 | 重点看什么 |
| --- | --- | --- |
| `go test -bench` | 本地比较两种实现 | `ns/op`、`B/op`、`allocs/op` |
| `benchstat` | 多轮 benchmark 结果对比 | 变化百分比、是否稳定 |
| `pprof` CPU | CPU 时间花在哪里 | `flat`、`cum`、火焰图热函数 |
| `pprof` heap | 内存被谁分配和持有 | `alloc_space`、`inuse_space`、对象数量 |
| `pprof` goroutine | goroutine 是否泄漏或阻塞 | 大量重复堆栈、channel/IO 阻塞点 |
| `pprof` mutex/block | 是否有锁竞争和阻塞 | 等待时间最高的锁、阻塞位置 |
| `go tool trace` | 调度、阻塞、网络和 GC 时间线 | goroutine 延迟、STW、syscall、network wait |
| 压测工具 | 真实并发下系统表现 | QPS、p95/p99、错误率、吞吐拐点 |
| 指标系统 | 线上趋势和告警 | 延迟分位数、队列长度、GC、goroutine、下游耗时 |

本地微基准的典型流程：

```bash
go test -run='^$' -bench=BenchmarkShardedCounterAdd -benchmem -count=5 ./... > before.txt

# 修改实现后再次运行
go test -run='^$' -bench=BenchmarkShardedCounterAdd -benchmem -count=5 ./... > after.txt

go install golang.org/x/perf/cmd/benchstat@latest
benchstat before.txt after.txt
```

这里的参数含义：

- `-run='^$'`：`^$` 是一个不会匹配任何测试名的正则，用来跳过普通单元测试，只跑 benchmark。
- `-bench=BenchmarkShardedCounterAdd`：要跑的 benchmark 名字正则。
- `-benchmem`：同时输出内存分配数据（`B/op`、`allocs/op`）。
- `-count=5`：重复跑 5 轮，`benchstat` 需要多轮数据才能判断差异是否稳定。

`ns/op` 表示每次操作耗时，`B/op` 表示每次操作分配多少字节，`allocs/op` 表示每次操作分配次数。热路径优化时，`allocs/op` 经常比单次耗时更重要，因为大量小分配会放大 GC 压力，最后体现在 p99 延迟上。

如果暂时没搭 `net/http/pprof`，也可以直接让 benchmark 输出 profile，配合 `go tool pprof` 定位热点：

```bash
go test -run='^$' -bench=BenchmarkShardedCounterAdd -benchmem \
  -cpuprofile=cpu.pb.gz -memprofile=mem.pb.gz ./...

go tool pprof -http=:8080 cpu.pb.gz
go tool pprof -http=:8080 mem.pb.gz
```

这套组合适合分析单个函数的 CPU 热点和分配行为，门槛比接入线上 pprof 低很多。

CPU profile 的使用方式：

```bash
# 服务内需要引入 net/http/pprof，下面的 6060 端口只监听本机
# 注意：这个请求会阻塞 seconds 指定的秒数（此处 30 秒）才返回结果，不是卡死
curl -o cpu.pb.gz 'http://127.0.0.1:6060/debug/pprof/profile?seconds=30'

# -http=:8080 让 pprof 开一个 web UI 在本机 8080 端口，浏览器自动打开
# 写成 :0 的话会让 OS 随机分配一个空闲端口
go tool pprof -http=:8080 ./your-server cpu.pb.gz
```

带二进制参数（`./your-server`）是为了让 pprof 做完整的符号解析。如果本机找不到对应二进制，pprof 也能画图，但一些行级视图会受限。

在 `pprof` web UI 里优先看三类信息：

- `Top` 视图：哪些函数直接消耗 CPU，`flat` 高说明函数本身重。
- `Top` 按 `cum` 排序：哪些调用链累计消耗 CPU，`cum` 高说明它下面的子调用重。
- `VIEW → Flame Graph`：一眼看出最宽的调用路径，适合发现重复序列化、排序、正则、日志格式化。

如果更习惯命令行，可以直接用交互模式：

```bash
go tool pprof ./your-server cpu.pb.gz
```

进入后常用命令是 `top`、`top -cum`、`list FunctionName`、`web`（生成调用图 SVG）。

heap profile 的使用方式：

```bash
# heap 端点默认返回 inuse_space 视图
curl -o heap.pb.gz 'http://127.0.0.1:6060/debug/pprof/heap'
go tool pprof -http=:8080 ./your-server heap.pb.gz

# 如果要直接看“累计分配了多少”，可以抓 allocs 端点
curl -o allocs.pb.gz 'http://127.0.0.1:6060/debug/pprof/allocs'
go tool pprof -http=:8080 ./your-server allocs.pb.gz
```

heap 要区分两个视角：

- `inuse_space`：当前还活着的内存，适合排查缓存无淘汰、队列积压、对象被长期引用。
- `alloc_space`：历史累计分配，适合排查热路径频繁临时分配，即使这些对象很快被 GC。

在 pprof 交互模式里可以用 `sample_index=inuse_space` 或 `sample_index=alloc_space` 在同一份 profile 里切换视图，无需重新抓取。

goroutine profile 的使用方式：

```bash
# debug=2 会把每个 goroutine 的完整栈以纯文本输出，可以直接在浏览器或终端查看
# 不加 debug=2 拿到的是 protobuf 二进制格式，需要 go tool pprof 打开
curl 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=2'
```

它背后的原理是把当前所有 goroutine 的栈抓出来。排查时不要逐个看，而是找“重复最多的堆栈”。如果几千个 goroutine 都停在同一个 channel send，问题通常在消费者处理太慢、退出了，或者队列没有背压策略；如果大量停在网络读写，优先检查超时、连接池和下游延迟。

锁竞争和阻塞默认不会完整记录，需要先在**服务进程里**开启采样。下面这两行必须放在运行中的 Go 程序里（通常是 `main` 启动阶段，或者通过管理接口动态开启），写在 shell 里不会起作用：

```go
// 放在服务进程的 main 或初始化阶段
runtime.SetMutexProfileFraction(5) // 平均每 5 个争抢事件采样 1 个；传 0 关闭，传 1 采样全部
runtime.SetBlockProfileRate(1)     // 平均每多少纳秒的阻塞采样一次；传 1 采样全部（开销最大），传 0 关闭
```

这两个参数的语义容易被直觉误读：数值**越大采样越稀疏**，`1` 反而是“全量采样”，开销最大。生产环境排障时常用稍大的值（例如 `SetMutexProfileFraction(100)`、`SetBlockProfileRate(10000)`）来降低开销；排查结束后再调回 `0` 关闭。

开启后再抓取：

```bash
curl -o mutex.pb.gz 'http://127.0.0.1:6060/debug/pprof/mutex'
curl -o block.pb.gz 'http://127.0.0.1:6060/debug/pprof/block'
go tool pprof -http=:8080 ./your-server mutex.pb.gz
go tool pprof -http=:8080 ./your-server block.pb.gz
```

`mutex` profile 看“等锁耗时”，不是看“锁被调用次数”。`block` profile 看 goroutine 在 channel、select、cond、网络 poll 等位置阻塞的时间。它们有额外开销，不建议一直用高采样率在线上开启。

trace 更适合回答“CPU profile 看不出来，但请求为什么还是慢”：

```bash
# trace 文件体积很大，seconds 通常控制在 5~15 秒，不要在高峰期长时间抓
curl -o trace.out 'http://127.0.0.1:6060/debug/pprof/trace?seconds=10'

# 建议带上二进制，否则部分视图（Syscalls、User-defined tasks 等）会受限
go tool trace ./your-server trace.out
```

`trace` 的核心原理是记录 Go runtime 的调度事件。重点看 goroutine 是在运行、等待调度、等待网络、等待锁，还是被 GC 影响。CPU profile 看到的是“运行时花在哪里”，trace 看到的是“没运行时在等什么”。

GC 行为是另一类常见问题，不一定要抓 profile。最轻量的方式是用 `GODEBUG` 直接打开 GC 日志：

```bash
GODEBUG=gctrace=1 ./your-server
```

每次 GC 会打印一行，例如：

```text
gc 42 @3.210s 2%: 0.12+5.6+0.08 ms clock, 1.0+0.9/4.5/0.0+0.6 ms cpu, 128->132->64 MB, 130 MB goal, 8 P
```

关注三类信息：`2%` 是 GC 占 CPU 的比例，`ms clock` 里的数字是 STW 和标记耗时，`128->132->64 MB` 是 GC 前/峰值/后的堆大小。如果 GC 百分比持续高、堆大小来回剧烈波动，通常说明有高频临时分配，需要回到 heap/allocs profile 找源头。

如果需要在代码里把运行时状态暴露成指标（Prometheus、OpenTelemetry 等），推荐使用 `runtime/metrics` 包，它比老的 `runtime.MemStats` 更全，并且字段稳定：

```go
import "runtime/metrics"

samples := []metrics.Sample{
	{Name: "/sched/goroutines:goroutines"},
	{Name: "/memory/classes/heap/objects:bytes"},
	{Name: "/gc/pauses:seconds"},
}
metrics.Read(samples)
```

服务要暴露 pprof，最小接入方式如下：

```go
import (
	"log"
	"net/http"
	_ "net/http/pprof"
)

func startDebugServer() {
	go func() {
		// 只监听 127.0.0.1，不暴露到任何外部网卡
		if err := http.ListenAndServe("127.0.0.1:6060", nil); err != nil {
			log.Printf("debug server stopped: %v", err)
		}
	}()
}
```

生产环境不要把 pprof 直接暴露到公网。具体可落地的做法：

- **只监听本机**：绑到 `127.0.0.1`，通过 SSH 端口转发进去抓 profile，例如 `ssh -L 6060:127.0.0.1:6060 user@host`。
- **独立管理端口 + 鉴权**：把 pprof 放到单独的管理端口（和业务端口区分），前面挂一层要鉴权的反向代理或内网网关。
- **容器/K8s 场景**：不要把 pprof 端口写进 Service，只通过 `kubectl port-forward` 或 sidecar 访问。
- **临时开启**：如果实在只能偶尔排查，可以做成"默认关闭、通过管理命令动态开启一段时间后自动关闭"。

压测工具用于把系统推到拐点。常见选择：

```bash
hey -z 60s -c 200 http://127.0.0.1:8080/api/top
wrk -t4 -c200 -d60s http://127.0.0.1:8080/api/top
vegeta attack -duration=60s -rate=2000/s -targets=targets.txt | vegeta report
```

压测时不要只看平均延迟。至少同时记录：

- `p50/p90/p99`：尾延迟是否开始恶化。
- 错误率：是否出现超时、限流、熔断、5xx。
- 队列长度和入队等待时间：是否已经积压。
- goroutine 数：是否随压测持续上涨且停止后不回落。
- heap、GC pause、GC 次数：是否因为分配压力导致尾延迟。
- 下游连接池、慢查询、Redis 命中率：是否把压力转移到了依赖服务。

这些工具的配合顺序通常是：

1. 指标发现问题：确认是 CPU、内存、锁、队列、下游还是错误率异常。
2. 日志缩小范围：用 request id、trace id、接口名、错误码定位具体请求类型。
3. trace 拆请求路径：判断时间花在入队、业务计算、下游调用还是序列化。
4. pprof 定函数和堆栈：CPU 看热函数，heap 看分配和持有，goroutine 看阻塞点。
5. benchmark 验证修改：用小而稳定的 benchmark 证明改动真的改善了目标指标。
6. 压测复验系统：确认优化在真实并发和真实数据分布下仍然有效。

---

