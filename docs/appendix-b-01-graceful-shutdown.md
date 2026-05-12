[返回首页](../README.md) / [返回上级](appendix-b-shutdown-circuit-breaker.md)

# B.1 优雅关闭

服务关闭不是“杀进程”。粗暴地退出会导致正在处理的请求失败、写入中的数据丢失、下游连接被强制断开。优雅关闭的目标是：收到停止信号后，先停止接收新请求，再等待已接收的请求完成，最后释放资源。

典型流程：

```text
1. 捕获 SIGINT / SIGTERM
2. 停止接收新请求（关闭监听、取消健康检查）
3. 通知后台 goroutine 退出
4. 等待在途请求处理完成（有最大等待时间）
5. flush 聚合状态、关闭数据库连接、上报下线
6. 进程退出
```

HTTP 服务的标准做法：

```go
func Run(ctx context.Context, srv *http.Server) error {
	errCh := make(chan error, 1)
	go func() {
		errCh <- srv.ListenAndServe()
	}()

	select {
	case <-ctx.Done():
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer cancel()
		return srv.Shutdown(shutdownCtx)
	case err := <-errCh:
		if err != nil && err != http.ErrServerClosed {
			return err
		}
		return nil
	}
}

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	srv := &http.Server{Addr: ":8080", Handler: router}
	if err := Run(ctx, srv); err != nil {
		log.Fatal(err)
	}
}
```

后台 worker pool 的优雅关闭分两步：先关闭入口（不再接收新任务），再等待队列 drain 完成或超时：

```go
func (p *Pool) Shutdown(ctx context.Context) error {
	p.closeOnce.Do(func() { close(p.closed) }) // 拒绝新任务
	done := make(chan struct{})
	go func() {
		p.wg.Wait() // 等待在途任务完成
		close(done)
	}()

	select {
	case <-done:
		return nil
	case <-ctx.Done():
		return ctx.Err() // 超时后强制退出，记录未完成任务数
	}
}
```

要点：

- 必须设置 shutdown 超时。永远等下去等于没有关闭。
- 关闭顺序要倒着来：先入口、再业务、最后基础设施（DB、MQ、缓存）。
- 关闭前把健康检查置为失败，让负载均衡尽快摘除自己。Kubernetes 环境下要配合 `preStop` hook 和 `terminationGracePeriodSeconds`。
- 异步任务要区分“可丢弃”和“必须完成”。必须完成的任务在关闭前应落盘或转移到持久队列。

