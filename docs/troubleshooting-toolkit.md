[返回首页](../README.md) / [返回上级](failure-modes.md)

# 排障工具组合

线上排障不要只依赖单个工具。单个工具容易给出局部答案，组合起来才能形成证据链。

| 现象 | 第一手工具 | 深入工具 | 最后验证 |
| --- | --- | --- | --- |
| p99 升高 | 指标、trace | CPU/heap/mutex/block profile | 压测复现 |
| CPU 高 | CPU profile | trace、日志采样 | benchmark、压测 |
| 内存上涨 | heap profile、GC 指标 | goroutine profile、缓存指标 | 长时间压测 |
| goroutine 上涨 | goroutine profile | block profile、trace | 停流量观察回落 |
| 锁竞争 | mutex profile、trace | benchmark 对比锁方案 | 不同并发数压测 |
| 下游慢 | trace、下游指标 | 日志、连接池指标 | 限流/熔断压测 |
| 数据错误 | race test、日志 | 单元测试、审查共享状态 | 回归测试 |

一个可执行的排障流程：

1. 先看仪表盘：QPS 是否变化，错误率是否升高，p99 是否和队列长度、GC、goroutine、下游延迟一起变化。
2. 再选一个具体接口或任务：不要混在全站平均值里分析，先锁定最异常的路径。
3. 抓 trace：确认慢在排队、运行、锁等待、GC、网络还是下游。
4. 抓对应 profile：运行慢看 CPU，分配多看 heap，阻塞看 goroutine/block，锁等待看 mutex。
5. 回到代码验证假设：找到热函数、共享锁、无界队列、重复计算或下游调用放大点。
6. 用 benchmark 验证局部改动：确认实现层面确实更快或分配更少。
7. 用压测验证整体效果：确认 p99、错误率、队列长度、GC 和下游指标都改善。

这套流程背后的核心原则是：指标负责发现异常，trace 负责拆路径，pprof 负责定位代码，benchmark 负责验证局部选择，压测负责验证系统行为。它们不是互相替代，而是回答不同层次的问题。

