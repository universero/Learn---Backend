> [[goroutine | goroutine基础知识]]
> [参考博客](https://strikefreedom.top/archives/high-performance-implementation-of-goroutine-pool#%E5%8F%82%E8%80%83)
> ___

goroutine pool适用于大规模平凡创建和销毁goroutine的场景

# 大规模 Goroutine 的瓶颈

1. 首先，即便每个 goroutine 只分配 2KB 的内存，但如果是恐怖如斯的数量，聚少成多，内存暴涨，就会对 GC 造成极大的负担，写过 Java 的同学应该知道 jvm GC 那万恶的 STW（Stop The World）机制，也就是 GC 的时候会挂起用户程序直到垃圾回收完，虽然 Go1.8 之后的 GC 已经去掉了 STW 以及优化成了并行 GC，性能上有了不小的提升，但是，如果太过于频繁地进行 GC，依然会有性能瓶颈；
2. 其次，还记得前面我们说的 runtime 和 GC 也都是 goroutine 吗？是的，如果 goroutine 规模太大，内存吃紧，runtime 调度和垃圾回收同样会出问题，虽然 G-M-P 模型足够优秀，韩信点兵，多多益善，但你不能不给士兵发口粮（内存）吧？没有内存，Go 调度器就会阻塞 goroutine，结果就是 P 的 Local 队列积压，又导致内存溢出，这就是个死循环...，甚至极有可能程序直接 Crash 掉，本来是想享受 golang 并发带来的效益，结果却得不偿失。

## http标准库的缺陷

net/http库的内部实现，从入口函数ListenAndServer开始查看源码
```go
func (srv *Server) ListenAndServe() error {
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
```
Serve方法是实际处理请求的逻辑
```go
func (srv *Server) Serve(l net.Listener) error {
	defer l.Close()
	...
    // 不断循环取出TCP连接
	for {
        // 看我看我！！！
		rw, e := l.Accept()
        ...
        // 再看我再看我！！！
		go c.serve(ctx)
	}
}
```
**首先，这个方法的参数`(l net.Listener)` ，是一个 TCP 监听的封装，负责监听网络端口， `rw, e := l.Accept()` 则是一个阻塞操作，从网络端口取出一个新的 TCP 连接进行处理，最后 `go c.serve(ctx)` 就是最后真正去处理这个 http 请求的逻辑了，这里启动了一个新的 goroutine 去执行处理逻辑，而且这是在一个无限循环体里面，所以意味着，每来一个请求它就会开一个 goroutine 去处理，不过有 Go 调度器背书，一般来说也没啥压力，然而，如果突然一大波请求涌进来了，这时候，就很成问题了，10w 个请求就要开 10w 个 goroutine，100w 个就要开 100w 个，线程调度压力陡升，内存爆满**

所以这里可以实现一个goroutine pool减少资源的消耗

# 实现一个Goroutine Pool

通过复用goroutine减轻runtime的调度压力和内存压力。

## 设计思路

启动服务之时先初始化一个 Goroutine Pool 池，这个 Pool 维护了一个类似栈的 LIFO 队列 ，里面存放负责处理任务的 Worker，然后在 client 端提交 task 到 Pool 中之后，在 Pool 内部，接收 task 之后的核心操作是：

1. 检查当前 Worker 队列中是否有可用的 Worker，如果有，取出执行当前的 task；
2. 没有可用的 Worker，判断当前在运行的 Worker 是否已超过该 Pool 的容量：{是 —> 再判断工作池是否为非阻塞模式：[是 ——> 直接返回 nil，否 ——> 阻塞等待直至有 Worker 被放回 Pool]，否 —> 新开一个 Worker（goroutine）处理}；
3. 每个 Worker 执行完任务之后，放回 Pool 的队列中等待。
![[GoroutinePool逻辑.png]]