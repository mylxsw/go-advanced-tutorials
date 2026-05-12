[返回首页](../README.md) / [返回上级](appendix-a-concurrency-primitives.md)

# A.6 小结

| 原语 | 适用场景 | 不适合的场景 |
| --- | --- | --- |
| `atomic` | 计数器、标志位、快照指针替换 | 多字段不变量 |
| `sync.Once` | 懒加载单例、一次性初始化 | 需要重试或条件重置 |
| `sync.Pool` | 复用临时 buffer、临时对象 | 当成长期缓存 |
| `errgroup` | 并发 fan-out + 错误短路 | 每个子任务必须独立成功 |
| `singleflight` | 缓存击穿、合并重复回源 | 每个请求结果需要互不影响 |

---

