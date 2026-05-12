# Go 高级开发教程：从并发原理到高性能系统设计

这是一份写给中高级 Go 开发者的教程。它不追求把每个 API 都罗列一遍，而是希望回答几个更根本的问题：

- goroutine 为什么轻量？轻量到什么程度？什么时候又会变重？
- channel、锁、`context` 分别解决什么问题？为什么不能混着乱用？
- Heap、Skiplist、分片 Map 这些数据结构为什么能提升性能？
- Top K、增量聚合、排行榜这些问题，本质上是在优化什么？
- 高并发系统为什么不能只靠“多开 goroutine”解决？
- 当系统处理不过来时，应该排队、丢弃、限流，还是降级？

很多人学习 Go 并发时，会先记住一句话：“不要通过共享内存来通信，而要通过通信来共享内存。”这句话很好，但如果只记住口号，反而容易误解。Go 的并发工具各有边界：channel 适合表达流程和同步，锁适合保护共享状态，`context` 适合控制生命周期，worker pool 适合控制系统容量。

真正成熟的工程师，不是把所有问题都写成 channel，也不是把所有共享数据都加一把大锁，而是知道在不同场景下做出合适的取舍。

这些问题的简短答案如下，后文会逐步展开。

| 问题 | 最佳答案 | 解读 |
| --- | --- | --- |
| goroutine 为什么轻量？ | 因为它由 Go runtime 在用户态调度，初始栈小，创建和切换成本低于 OS 线程。 | 轻量不等于免费。goroutine 过多仍会增加内存、调度和 GC 成本。 |
| channel、锁、`context` 分别解决什么？ | channel 表达通信和同步，锁保护共享状态，`context` 控制生命周期。 | 不要为了“更 Go”而滥用 channel。保护一个 Map 时，锁通常更直接。 |
| 数据结构为什么能提升性能？ | 好的数据结构减少了不必要的扫描、排序、锁竞争和内存访问。 | Heap 避免全量排序，Skiplist 支持动态有序，分片 Map 降低热点锁。 |
| Top K 和增量聚合本质上优化什么？ | 它们都在避免重复处理无关数据。 | Top K 不排序所有人，增量聚合不每次扫描原始事件。 |
| 高并发为什么不能只靠 goroutine？ | 因为真正稀缺的是 CPU、内存、队列容量、下游吞吐和锁资源。 | goroutine 只能表达并发，不能创造系统容量。 |
| 系统处理不过来怎么办？ | 优先限流和背压，再考虑排队、降级和补偿。 | 在线请求通常快速失败更好；关键数据写入要有补偿机制。 |

一个重要原则是：高并发系统不是让所有请求都成功，而是在超出容量时，让系统以可预期的方式失败。可预期，比表面上的“全都接收”更重要。

---

## 目录

- [学习路线](docs/learning-route.md)
- [第一部分：Go 并发基础](docs/part-01-go-concurrency.md)
  - [1. goroutine：不是免费的线程](docs/01-goroutine.md)
  - [2. channel：带同步语义的队列](docs/02-channel.md)
  - [3. Mutex 与 RWMutex：保护不变量](docs/03-mutex-rwmutex.md)
  - [4. sync.Map：特殊场景下的并发 Map](docs/04-sync-map.md)
  - [5. context：让并发任务知道什么时候该停](docs/05-context.md)
- [第二部分：高性能数据结构](docs/part-02-data-structures.md)
  - [6. Heap：用局部有序换取效率](docs/06-heap.md)
  - [7. 优先队列：让重要任务先执行](docs/07-priority-queue.md)
  - [8. TreeMap 与 Skiplist：为范围查询而生](docs/08-treemap-skiplist.md)
- [第三部分：并发数据结构设计](docs/part-03-concurrent-data-structures.md)
  - [9. 线程安全的本质](docs/09-thread-safety.md)
- [第四部分：算法优化](docs/part-04-algorithm-optimization.md)
  - [10. Top K：不要为无关数据排序](docs/10-top-k.md)
  - [11. 高频访问优化](docs/11-high-frequency-access.md)
  - [12. 增量聚合：把查询成本前移](docs/12-incremental-aggregation.md)
- [第五部分：工程化设计](docs/part-05-engineering-design.md)
  - [13. 接口设计](docs/13-api-design.md)
  - [14. 错误处理](docs/14-error-handling.md)
  - [15. 日志、指标与测试](docs/15-logging-metrics-testing.md)
- [第六部分：高并发系统设计](docs/part-06-high-concurrency-system-design.md)
  - [16. Worker Pool：给系统设置边界](docs/16-worker-pool.md)
  - [17. Fan-out / Fan-in：并行也会放大流量](docs/17-fan-out-fan-in.md)
  - [18. Backpressure：承认系统有容量上限](docs/18-backpressure.md)
  - [19. 限流：系统的安全阀](docs/19-rate-limiting.md)
- [第七部分：实战项目](docs/part-07-projects.md)
  - [20. Leaderboard 高级实现](docs/20-leaderboard.md)
  - [21. ActivityTracker 高级实现](docs/21-activity-tracker.md)
- [附录 A：补充并发原语](docs/appendix-a-concurrency-primitives.md)
  - [A.1 atomic：无锁计数与标志位](docs/appendix-a-01-atomic.md)
  - [A.2 sync.Once：只执行一次的初始化](docs/appendix-a-02-sync-once.md)
  - [A.3 sync.Pool：复用临时对象](docs/appendix-a-03-sync-pool.md)
  - [A.4 errgroup：并发等待 + 错误短路](docs/appendix-a-04-errgroup.md)
  - [A.5 singleflight：合并重复请求](docs/appendix-a-05-singleflight.md)
  - [A.6 小结](docs/appendix-a-06-summary.md)
- [附录 B：优雅关闭与熔断降级](docs/appendix-b-shutdown-circuit-breaker.md)
  - [B.1 优雅关闭](docs/appendix-b-01-graceful-shutdown.md)
  - [B.2 熔断（Circuit Breaker）](docs/appendix-b-02-circuit-breaker.md)
  - [B.3 降级（Fallback）](docs/appendix-b-03-fallback.md)
- [高并发设计检查清单](docs/high-concurrency-checklist.md)
- [常见线上故障模式](docs/failure-modes.md)
  - [排障工具组合](docs/troubleshooting-toolkit.md)
  - [生产排障手册](docs/production-troubleshooting.md)
- [结语](docs/conclusion.md)

---

## 学习路线

建议用 12 周完成这套训练。

| 周期 | 主题 | 要掌握的能力 |
| --- | --- | --- |
| 第 1-2 周 | Go 并发基础 | 理解 goroutine、channel、锁、`context` 的原理和边界 |
| 第 3-4 周 | 高性能数据结构 | 掌握 Heap、优先队列、TreeMap、Skiplist 的适用场景 |
| 第 5-6 周 | 并发数据结构 | 能设计线程安全 Map、计数器、缓存、排行榜 |
| 第 7-8 周 | 算法优化 | 能用 Top K、增量聚合、缓存减少系统负担 |
| 第 9-10 周 | 工程化设计 | 会设计接口、错误、日志、指标和测试 |
| 第 11-12 周 | 高并发系统 | 能设计 worker pool、fan-out、backpressure、限流系统 |

学习时不要只看代码。每一个主题都要从四个层次理解：

| 层次 | 应该问的问题 |
| --- | --- |
| 使用层 | API 怎么用？最小示例是什么？ |
| 原理层 | runtime 或数据结构内部大致做了什么？ |
| 工程层 | 生产环境下什么时候该用，什么时候不该用？ |
| 故障层 | 高并发时会怎样失败？如何保护系统？ |

---
