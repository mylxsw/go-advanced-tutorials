# 附录 A：补充并发原语

除了 goroutine、channel、锁和 `context`，Go 标准库和 `golang.org/x/sync` 还提供了几个常用的并发原语。它们解决的场景更具体，但在高并发系统里非常实用。

## 本部分目录

- [A.1 atomic：无锁计数与标志位](appendix-a-01-atomic.md)
- [A.2 sync.Once：只执行一次的初始化](appendix-a-02-sync-once.md)
- [A.3 sync.Pool：复用临时对象](appendix-a-03-sync-pool.md)
- [A.4 errgroup：并发等待 + 错误短路](appendix-a-04-errgroup.md)
- [A.5 singleflight：合并重复请求](appendix-a-05-singleflight.md)
- [A.6 小结](appendix-a-06-summary.md)

[返回首页](../README.md)
