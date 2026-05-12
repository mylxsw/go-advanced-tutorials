[返回首页](../README.md) / [返回上级](part-04-algorithm-optimization.md)

# 11. 高频访问优化

高频路径优化的第一原则是：不要在每个请求里做重复且昂贵的事情。

常见成本包括：

- 锁竞争。
- 内存分配。
- 全量扫描。
- 全量排序。
- 数据库访问。
- 序列化和反序列化。
- 同步日志。

## 11.1 缓存的几个坑

| 问题 | 含义 | 解决方向 |
| --- | --- | --- |
| 缓存击穿 | 热点 key 过期，大量请求同时回源 | singleflight |
| 缓存穿透 | 查询不存在的数据 | 空值缓存 |
| 缓存雪崩 | 大量 key 同时过期 | TTL 加随机抖动 |
| 热点 key | 单个 key 被大量访问 | 多副本、分片、预热 |

缓存的目标不是让数据永远不变，而是在可接受的一致性范围内减少后端压力。

缓存问题的参考答案与生产解读：

- 缓存击穿的最佳做法是 singleflight。多个请求同时发现热点 key 过期时，只允许一个请求回源，其余请求等待结果或使用旧值。完整代码示例见[附录 A.5](appendix-a-05-singleflight.md)。
- 缓存穿透要缓存“不存在”的结果，但 TTL 要短，避免后来数据创建后仍长期返回不存在。
- 缓存雪崩要给 TTL 加随机抖动，例如基础 TTL 10 分钟，额外随机 0 到 60 秒，避免大量 key 同时失效。
- 热点 key 可以做本地缓存、多副本缓存、读写分离或提前预热。极端热点下，单个 Redis key 本身也会成为瓶颈。

生产实践：

- 缓存值要有版本或更新时间，方便排查“为什么用户看到旧数据”。
- 对核心数据使用 cache-aside 时，更新数据库后要删除缓存，而不是直接更新缓存。删除失败要有重试或消息补偿。
- 对读多写少的数据，可以使用本地内存缓存 + 远程缓存两级结构，但要注意本地缓存失效和容量上限。
- 热路径里不要同步打印大日志，不要每次都 JSON 序列化大对象，可以缓存序列化结果。
- 缓存不是一致性方案。涉及资金、库存、权限等关键数据时，缓存只能加速读取，最终判断仍应基于权威存储或强一致服务。

## 11.2 热路径优化示例：缓存序列化结果

很多接口的瓶颈不是数据库，而是每次请求都重复构造响应、排序、JSON 序列化。对于读多写少的数据，可以把序列化后的结果也放入快照。

```go
type TopSnapshot struct {
	Entries []Entry
	JSON    []byte
	At      time.Time
}

type TopCache struct {
	value atomic.Value // stores *TopSnapshot
}

func (c *TopCache) LoadJSON() ([]byte, bool) {
	v := c.value.Load()
	if v == nil {
		return nil, false
	}
	s := v.(*TopSnapshot)
	return append([]byte(nil), s.JSON...), true
}

func (c *TopCache) Store(entries []Entry) error {
	cp := append([]Entry(nil), entries...)
	data, err := json.Marshal(cp)
	if err != nil {
		return err
	}
	c.value.Store(&TopSnapshot{
		Entries: cp,
		JSON:    data,
		At:      time.Now(),
	})
	return nil
}
```

这里 `LoadJSON` 返回 `JSON` 的副本，是为了避免调用方修改内部字节切片。生产中如果响应写出后不会被修改，也可以通过约定减少拷贝，但前提是团队能严格遵守不可变规则。

这个优化背后的原理是“把重复计算从请求路径移到刷新路径”。如果榜单每秒刷新一次，而接口每秒被请求 5 万次，序列化一次和序列化 5 万次的成本差别非常大。

## 11.3 singleflight 的正确使用位置

`singleflight` 适合放在缓存回源处，而不是整个接口入口处。下面是一个典型 cache-aside 读取：

```go
type UserService struct {
	cache Cache
	group singleflight.Group
}

func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
	key := fmt.Sprintf("user:%d", id)

	if u, ok := s.cache.Get(key); ok {
		return u.(*User), nil
	}

	v, err, _ := s.group.Do(key, func() (any, error) {
		if u, ok := s.cache.Get(key); ok {
			return u.(*User), nil
		}

		u, err := loadUserFromDB(ctx, id)
		if err != nil {
			return nil, err
		}
		s.cache.Set(key, u, 5*time.Minute)
		return u, nil
	})
	if err != nil {
		return nil, err
	}
	return v.(*User), nil
}
```

回调里再次读缓存是必要的。因为当前 goroutine 等待进入 singleflight 回调期间，可能已经有别的请求把缓存填好了。这个二次检查可以减少不必要的 DB 访问。

生产注意点：

- singleflight 只合并同进程内的请求，多实例部署时还需要远程缓存或分布式锁配合。
- 不要对所有用户共用同一个 key，否则会把无关请求串行化。
- 回源必须有超时。否则热点 key 的所有等待者都会被一个慢查询拖住。
- 对不存在的数据也要缓存短 TTL 的空值，防止穿透。

---

