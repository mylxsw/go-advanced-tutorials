# 常见线上故障模式

| 现象 | 常见原因 | 处理方向 |
| --- | --- | --- |
| goroutine 持续上涨 | 阻塞、泄漏、未监听 context | 加超时、取消、边界 |
| p99 延迟升高 | 锁竞争、GC、下游慢、队列积压 | 看 trace、pprof、指标 |
| CPU 高但吞吐低 | 自旋、调度频繁、日志过多 | 降并发、批量、减少热路径开销 |
| 内存上涨 | 队列积压、缓存无淘汰 | 加上限、TTL、pprof |
| 数据偶发错误 | 数据竞争、快照被修改 | race test、不可变对象 |
| 数据库被打爆 | fan-out、重试风暴、缺少限流 | 限流、熔断、缓存 |

## 本部分目录

- [排障工具组合](troubleshooting-toolkit.md)
- [生产排障手册](production-troubleshooting.md)

[返回首页](../README.md)
