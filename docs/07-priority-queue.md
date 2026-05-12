[返回首页](../README.md) / [返回上级](part-02-data-structures.md)

# 7. 优先队列：让重要任务先执行

普通队列讲先来后到，优先队列讲谁更重要。

典型场景：

- 调度高优先级任务。
- 延迟队列取最早到期任务。
- 告警系统优先处理严重告警。
- 重试系统优先处理即将到期的任务。

## 7.1 生产级优先队列的难点

难点不在插入和弹出，而在任务会变化：

- 任务可能取消。
- 优先级可能调整。
- 执行时间可能推迟。
- 队列里可能残留无效任务。

因此常见实现会维护 `index` 字段，用于快速 `heap.Fix`。

```go
type Task struct {
	ID       string
	Priority int
	index    int
}
```

如果没有索引，更新一个任务需要扫描整个堆，复杂度会退化成 `O(n)`。

## 7.2 常见坑

### 坑 1：修改堆中元素后忘记 `heap.Fix`

为什么会出现：堆只在 `heap.Push`、`heap.Pop`、`heap.Fix` 等操作时维护堆序。你直接修改元素字段，堆并不知道顺序已经被破坏。

错误示例：

```go
task.Priority = 100 // 堆结构不会自动调整
```

修复方式：元素中保存 `index`，修改优先级后调用 `heap.Fix`。

```go
func (pq *PriorityQueue) Update(task *Task, priority int) {
	task.Priority = priority
	heap.Fix(pq, task.index)
}
```

生产建议：不要把堆中元素直接暴露给外部随意修改。提供 `Update` 方法统一维护不变量。

### 坑 2：`Less` 写反，最大堆和最小堆混淆

为什么会出现：Go 标准库 `container/heap` 默认根据 `Less` 构造最小堆。如果希望优先级大的先出队，`Less` 要反过来写。

```go
func (pq PriorityQueue) Less(i, j int) bool {
	return pq[i].Priority > pq[j].Priority // 最大堆
}
```

修复方式：给堆写单元测试，明确第一个 `Pop` 出来的是最高优先级还是最低优先级。

### 坑 3：任务取消后不删除，堆里积累大量无效任务

为什么会出现：延迟队列或任务调度中，任务可能在执行前被取消。如果只在任务弹出时发现它已取消，堆里会长期堆积无效节点。

解决方式：

- 如果能通过 `index` 找到任务，取消时调用 `heap.Remove`。
- 如果取消非常频繁，可以使用 lazy delete，但要监控无效节点比例，超过阈值后重建堆。

```go
func (pq *PriorityQueue) Cancel(task *Task) {
	if task.index >= 0 {
		heap.Remove(pq, task.index)
	}
}
```

### 坑 4：内存延迟队列没有持久化

为什么会出现：内存堆只存在于当前进程。进程重启、发布、宕机后，尚未执行的任务会丢失。

生产建议：订单超时关闭、支付补偿、消息重试这类关键任务，不要只放内存。可选方案包括：

- 数据库表保存任务状态，内存堆只做加速索引。
- Redis Sorted Set 保存执行时间。
- Kafka / RocketMQ / RabbitMQ 延迟消息。
- 服务启动时从持久化存储恢复未完成任务。

---

