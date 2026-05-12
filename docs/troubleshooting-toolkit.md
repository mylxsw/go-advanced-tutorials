[返回首页](../README.md) / [返回上级](failure-modes.md)

# 排障工具组合

线上排障不要只依赖单个工具。单个工具通常只能回答一个局部问题：指标告诉你“系统不对劲”，trace 告诉你“一次请求慢在哪里”，pprof 告诉你“代码把 CPU、内存或等待时间花在哪里”，benchmark 告诉你“某个局部实现是否真的更好”，压测告诉你“整个系统在真实并发下会不会崩”。

成熟的排障不是拿到一个 profile 就开始改代码，而是建立证据链：

1. 先确认现象是不是存在。
2. 再确认影响范围和触发条件。
3. 然后把系统耗时拆成排队、运行、等待下游、等待锁、等待 GC 等部分。
4. 最后定位到代码、配置或依赖，并用压测验证修复。

## 现象到工具的快速索引

| 现象 | 第一手工具 | 深入工具 | 最后验证 |
| --- | --- | --- | --- |
| p99 升高 | 指标、trace | CPU/heap/mutex/block profile | 压测复现 |
| CPU 高 | CPU profile | trace、日志采样 | benchmark、压测 |
| 内存上涨 | heap profile、GC 指标 | goroutine profile、缓存指标 | 长时间压测 |
| goroutine 上涨 | goroutine profile | block profile、trace | 停流量观察回落 |
| 锁竞争 | mutex profile、trace | benchmark 对比锁方案 | 不同并发数压测 |
| 下游慢 | trace、下游指标 | 日志、连接池指标 | 限流/熔断压测 |
| 数据错误 | race test、日志 | 单元测试、审查共享状态 | 回归测试 |

这个表不是让你机械套用，而是帮助你第一时间选对入口。真正排障时，通常会从指标开始，经过 trace 或 profile 缩小范围，再回到代码和压测。

## 指标：发现异常和判断影响面

指标是线上排障的第一层证据。它不适合解释每一行代码为什么慢，但适合回答三个问题：

- 异常什么时候开始？
- 哪个接口、任务、实例、下游或租户最异常？
- 异常和哪些系统信号同时变化？

高并发服务至少应该有这些指标：

```text
http_requests_total{path,method,status}
http_request_duration_seconds_bucket{path,method}
worker_queue_length{pool}
worker_submit_total{pool,result="ok|busy|timeout|dropped"}
worker_task_duration_seconds_bucket{pool}
runtime_goroutines
runtime_heap_alloc_bytes
runtime_gc_pause_seconds_bucket
dependency_request_duration_seconds_bucket{name,operation}
dependency_errors_total{name,operation,reason}
```

看指标时不要只看平均值。平均延迟很容易掩盖尾延迟：99 个请求 10ms，1 个请求 2s，平均值仍然可能看起来不吓人，但用户已经能感受到卡顿。高并发系统更关心 `p90`、`p99`、错误率、队列长度和等待时间。

排障时常见的指标组合判断：

| 指标变化 | 常见含义 |
| --- | --- |
| QPS 不变，p99 上涨，CPU 上涨 | 代码热路径变重、锁竞争、GC 压力或日志过多 |
| QPS 上涨，队列长度上涨，p99 上涨 | 系统处理能力到达上限，排队开始主导延迟 |
| QPS 不变，下游耗时上涨，本服务 p99 上涨 | 本服务可能只是被下游拖慢 |
| goroutine 数持续上涨，停止流量后不回落 | 大概率有泄漏、阻塞或没有取消 |
| heap 上涨，GC 频率上升，p99 同步上涨 | 分配压力或对象长期存活影响尾延迟 |
| 限流数上涨，但错误率稳定 | 系统正在保护自己，不一定是坏事 |

指标的原理很简单：服务在关键边界记录计数器、直方图和仪表值，监控系统按时间聚合。它的强项是趋势，不是细节。因此指标发现问题后，不要在仪表盘上猜代码，要继续抓 trace 或 profile。

## 日志：补充离散事件和业务上下文

日志适合回答“发生了什么事件”，尤其是指标和 profile 看不见的业务信息，例如用户 ID、订单 ID、限流原因、降级分支、重试次数、下游错误码。

高并发服务的日志要结构化：

```text
time=... level=warn trace_id=... user_id=... path=/api/feed
event=dependency_timeout dependency=profile timeout_ms=80 retry=1 cost_ms=83
```

排障时优先找这些字段：

- `trace_id`：把一次请求的入口、内部调用和下游调用串起来。
- `cost_ms`：关键阶段耗时，例如排队、执行、DB、RPC、序列化。
- `result`：成功、超时、限流、降级、取消、重试失败。
- `dependency`：哪个下游慢或错。
- `attempt` 或 `retry`：是否出现重试风暴。

日志的风险是量太大。高并发系统不要在热路径打印大量同步日志，也不要对每个请求都打印完整大对象。常用做法是错误日志全量，慢请求采样，正常请求按比例采样。日志用于补充证据，不应该替代指标和 trace。

## pprof CPU：定位 CPU 时间花在哪里

CPU profile 回答的问题是：程序真正运行在 CPU 上时，时间主要消耗在哪些函数和调用链上。

典型抓取方式：

```bash
curl -o cpu.pb.gz 'http://127.0.0.1:6060/debug/pprof/profile?seconds=30'
go tool pprof -http=:8080 ./your-server cpu.pb.gz
```

也可以进入命令行交互模式：

```bash
go tool pprof ./your-server cpu.pb.gz
```

常用命令：

```text
top
top -cum
list FunctionName
web
```

理解 CPU profile 要区分 `flat` 和 `cum`：

| 字段 | 含义 | 怎么读 |
| --- | --- | --- |
| `flat` | 函数自身消耗的 CPU | 高说明函数本身很重 |
| `cum` | 函数和它调用的子函数累计消耗 | 高说明这条调用链很重 |

如果 `encoding/json`、排序、正则、字符串拼接、日志格式化、加解密、压缩等函数很宽，通常说明热路径在重复做昂贵计算。解决方向可能是缓存中间结果、减少字段、改数据结构、批量处理，或者把同步工作挪出请求路径。

CPU profile 的常见误读是：看到 CPU 高就以为机器不够。CPU 高只是现象，真正的问题可能是每个请求做了太多无效工作，也可能是锁竞争导致 goroutine 频繁唤醒和抢占，还可能是重试风暴把流量放大了。

## pprof heap：区分“分配多”和“活得久”

heap profile 回答两个不同问题：

- 谁导致了大量分配？
- 谁持有了大量仍然存活的内存？

抓取方式：

```bash
curl -o heap.pb.gz 'http://127.0.0.1:6060/debug/pprof/heap'
go tool pprof -http=:8080 ./your-server heap.pb.gz

curl -o allocs.pb.gz 'http://127.0.0.1:6060/debug/pprof/allocs'
go tool pprof -http=:8080 ./your-server allocs.pb.gz
```

两个视角要分清：

| 视角 | 关注点 | 适合排查 |
| --- | --- | --- |
| `inuse_space` | 当前还活着的内存 | 缓存无淘汰、队列积压、大对象被引用 |
| `alloc_space` | 历史累计分配 | 热路径临时对象、频繁序列化、重复构造切片 |

内存问题不一定表现为 OOM。很多时候是分配太频繁，GC 被迫更频繁运行，最终表现为 p99 抖动。比如接口每次请求都构造大量临时 `[]byte`、`map[string]any` 或 JSON 对象，单次看不明显，在高 QPS 下会把 GC 压力放大。

判断方向：

- `alloc_space` 高但 `inuse_space` 不高：对象很快死亡，重点减少临时分配。
- `inuse_space` 持续上涨：对象被长期引用，重点查缓存、全局 map、队列、goroutine 栈。
- heap 上涨伴随队列长度上涨：可能不是内存泄漏，而是系统处理不过来导致积压。

## pprof goroutine：找泄漏和阻塞点

goroutine profile 回答的问题是：当前有哪些 goroutine，它们分别停在哪里。

抓纯文本栈：

```bash
curl 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=2'
```

抓 profile 文件：

```bash
curl -o goroutine.pb.gz 'http://127.0.0.1:6060/debug/pprof/goroutine'
go tool pprof -http=:8080 ./your-server goroutine.pb.gz
```

看 goroutine profile 不要逐条读，要先找重复最多的堆栈：

| 堆栈位置 | 常见含义 |
| --- | --- |
| 停在 channel send | 消费者慢、消费者退出、队列无背压 |
| 停在 channel receive | 生产者退出、没有关闭 channel、任务等待无限期输入 |
| 停在 `select` 且没有 `ctx.Done()` | 取消信号无法传入，容易泄漏 |
| 停在网络读写 | 下游慢、没有超时、连接池耗尽 |
| 停在 `time.After` 或定时器 | 可能有大量未释放定时器或循环等待 |

排查 goroutine 上涨时，建议连续抓两次，中间间隔 30 秒。如果同一类堆栈数量持续上涨，停止流量后仍不回落，基本可以判断有泄漏或永久阻塞。

## mutex 和 block profile：看等待时间，不是看调用次数

CPU profile 只能看到 goroutine 在运行时的 CPU 消耗，很多线上慢请求其实是在等待。mutex 和 block profile 就是为了看等待。

开启采样：

```go
runtime.SetMutexProfileFraction(100)
runtime.SetBlockProfileRate(10000)
```

抓取：

```bash
curl -o mutex.pb.gz 'http://127.0.0.1:6060/debug/pprof/mutex'
curl -o block.pb.gz 'http://127.0.0.1:6060/debug/pprof/block'

go tool pprof -http=:8080 ./your-server mutex.pb.gz
go tool pprof -http=:8080 ./your-server block.pb.gz
```

`mutex` profile 看的是 goroutine 等锁花了多少时间。某把锁的等待时间高，说明它保护的临界区太热、太长，或者锁内做了外部 IO。

`block` profile 看的是 goroutine 在 channel、select、cond、网络 poll 等阻塞点等待了多久。它适合排查 worker pool 队列、channel 管道、等待下游响应等问题。

这两个 profile 有采样开销，不建议长期高采样开启。排障结束后要关闭：

```go
runtime.SetMutexProfileFraction(0)
runtime.SetBlockProfileRate(0)
```

常见优化方向：

- 缩小锁粒度，把无关状态拆开保护。
- 缩短锁持有时间，锁内只改内存状态，不做 RPC、DB、日志。
- 对热点 key 做分片，降低单点竞争。
- 对读多写少场景使用快照，而不是所有读都抢同一把锁。

## go tool trace：看请求没运行时在等什么

trace 回答的问题和 pprof 不一样。pprof 更像抽样统计，trace 更像时间线。它可以看到 goroutine 创建、运行、等待调度、系统调用、网络等待、GC、锁等待等事件。

抓取方式：

```bash
curl -o trace.out 'http://127.0.0.1:6060/debug/pprof/trace?seconds=10'
go tool trace trace.out
```

trace 特别适合这些问题：

- CPU profile 看不出热点，但请求还是慢。
- p99 偶发尖刺，平均值正常。
- 怀疑 goroutine 很多但真正运行不多。
- 怀疑网络等待、锁等待、GC 或调度延迟。
- fan-out 请求里某个慢下游拖住整体返回。

看 trace 时要把一次请求拆成几段：

| 时间类型 | 含义 | 常见原因 |
| --- | --- | --- |
| Running | goroutine 正在 CPU 上执行 | 计算重、序列化、排序、压缩 |
| Runnable | 可以运行但还没被调度 | CPU 打满、goroutine 太多、调度压力大 |
| Network wait | 等网络 IO | 下游慢、连接池问题、超时设置不合理 |
| Syscall | 等系统调用 | 文件 IO、DNS、系统资源等待 |
| GC assist / STW | 受 GC 影响 | 分配太多、堆太大 |
| Sync block | 等锁或 channel | 锁竞争、队列积压、消费者慢 |

trace 的核心价值是把“慢”拆开。只有知道慢在运行、排队、等网络还是等锁，后面的 profile 才能抓对方向。

## benchmark 和 benchstat：验证局部选择

benchmark 不适合证明整个系统能扛住线上流量，但非常适合比较两个局部实现，例如：

- `Mutex` 和 `RWMutex` 哪个更适合当前读写比例。
- 分片数 32、64、128 哪个更稳定。
- Top K 用全量排序还是小根堆。
- JSON 每次序列化还是缓存序列化结果。

运行方式：

```bash
go test -run='^$' -bench=BenchmarkShardedCounterAdd -benchmem -count=5 ./... > before.txt
go test -run='^$' -bench=BenchmarkShardedCounterAdd -benchmem -count=5 ./... > after.txt

go install golang.org/x/perf/cmd/benchstat@latest
benchstat before.txt after.txt
```

重点看：

| 字段 | 含义 |
| --- | --- |
| `ns/op` | 每次操作耗时 |
| `B/op` | 每次操作分配字节数 |
| `allocs/op` | 每次操作分配次数 |

写 benchmark 时，数据分布比代码本身更重要。同一个分片 map，如果所有 goroutine 都写同一个 key，测的是热点竞争；如果 key 随机分布，测的是分片扩展性。两者都真实，但回答的问题不同。

## race test：证明并发路径没有明显数据竞争

数据错误类问题不能只靠 pprof。pprof 看到的是性能，不会告诉你“同一个 map 被两个 goroutine 同时写了”。

运行方式：

```bash
go test -race ./...
```

race test 适合覆盖：

- 缓存读写。
- worker pool 状态变化。
- 排行榜更新和查询。
- copy-on-write 快照替换。
- channel 关闭和 goroutine 退出。

`-race` 不能证明没有并发 bug，但它能抓到大量真实的数据竞争。对核心并发结构，race test 应该进入 CI。排查偶发数据错误时，先构造高并发单元测试，再开 `-race`，通常比直接看线上日志更有效。

## 压测工具：验证系统容量和拐点

压测回答的是系统级问题：吞吐、延迟、错误率、队列长度、下游耗时、GC、goroutine 是否在某个并发点同时恶化。

常用工具：

```bash
hey -z 1m -c 100 http://127.0.0.1:8080/api/feed

wrk -t4 -c200 -d60s http://127.0.0.1:8080/api/feed

echo 'GET http://127.0.0.1:8080/api/feed' | vegeta attack -duration=60s -rate=500 | vegeta report
```

压测不要只跑一个并发数。更有价值的是找拐点：

1. 从低并发开始，确认基线。
2. 逐步提高 QPS 或并发数。
3. 每一档记录 p50、p90、p99、错误率、CPU、heap、GC、goroutine、队列长度、下游耗时。
4. 找到 p99 和错误率开始快速上升的位置。
5. 验证限流、背压、降级是否按预期触发。

如果 QPS 继续上升但吞吐不再增加，p99 和错误率开始恶化，说明系统已经越过容量上限。这个点比“最大 QPS”更重要，因为生产环境需要运行在拐点之前。

## 下游和连接池指标：很多慢不是 Go 代码本身慢

高并发系统经常被下游拖慢。只看本服务 CPU 和 heap，容易误判。

数据库至少要看连接池：

```go
stats := db.Stats()
fmt.Println(stats.OpenConnections)
fmt.Println(stats.InUse)
fmt.Println(stats.Idle)
fmt.Println(stats.WaitCount)
fmt.Println(stats.WaitDuration)
```

判断方式：

| 指标 | 含义 |
| --- | --- |
| `InUse` 长期接近上限 | 连接池可能打满 |
| `WaitCount` 持续上涨 | 请求在等连接 |
| `WaitDuration` 上涨 | 等连接时间正在影响延迟 |
| 下游 p99 上涨 | 本服务慢可能只是下游慢的传导 |

HTTP/RPC 下游也类似，要观察连接复用、超时、重试、错误码和每个 operation 的延迟。尤其要警惕重试风暴：一个下游慢了，本服务自动重试三次，实际流量被放大，最后把下游打得更慢。

## 一套可执行的排障流程

1. 看仪表盘，确认异常时间、范围和相关指标。
2. 锁定一个具体接口、任务或下游，不要在全站平均值里猜。
3. 抓 trace，把耗时拆成运行、等待调度、等待网络、等待锁、GC。
4. 根据 trace 选择 profile：运行慢看 CPU，分配多看 heap，阻塞看 goroutine/block，等锁看 mutex。
5. 回到代码验证假设，确认是否有全局锁、无界队列、重复计算、下游放大或缺少超时。
6. 用 benchmark 验证局部实现是否改善。
7. 用压测验证整体效果，确认 p99、错误率、队列长度、GC、goroutine 和下游指标都改善。

排障最忌讳“看到一个现象就改一处代码”。好的排障应该能说清楚：异常是什么，证据是什么，根因在哪里，修复为什么有效，修复后哪些指标证明它真的变好了。
