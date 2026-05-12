[返回首页](../README.md) / [返回上级](appendix-a-concurrency-primitives.md)

# A.3 sync.Pool：复用临时对象

`sync.Pool` 用来复用短期对象，降低 GC 压力。典型场景是 JSON 编解码缓冲、`bytes.Buffer`、临时切片。

```go
var bufPool = sync.Pool{
	New: func() any { return new(bytes.Buffer) },
}

func Format(v any) string {
	buf := bufPool.Get().(*bytes.Buffer)
	buf.Reset()
	defer bufPool.Put(buf)

	fmt.Fprintf(buf, "%v", v)
	return buf.String()
}
```

要点：

- Pool 里的对象随时可能被 GC 回收，不能用来做“缓存”。
- 放回前必须 Reset，避免污染下一个使用者。
- 不要把已经被别人引用的对象放回 Pool，否则会导致数据竞争。
- 只对热路径和大对象使用。小对象直接 new 通常更简单，GC 代价也不大。

