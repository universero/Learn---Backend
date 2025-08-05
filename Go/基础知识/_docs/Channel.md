___
## 概述

Channel是Go中的通信机制, 可以让一个goroutine向另一个goroutine发送值消息, 每个Channeld都有一个值的类型.
通过内置函数可以创建一个channel变量, `make(chan int)`, channel主要有两个操作, 通过channel变量和<-运算符位置的关系来区分
- 发送 `ch <- x`
- 接受 `x = <- ch`
和map一样, channel对应一个底层数据结构的引用, 零值是nil, 传递channel参数时传递的都是指针.两个同类型的channel可以用`==`进行比较, 仅当两个引用相同对象时结果为真
Channel还支持close操作, 用于关闭channel, 关闭后对channel的操作都将触发panic异常

### 无缓存的Channel

无缓存的Channel的发送操作会导致发送者goroutine的阻塞知道有消费者接收了这个channel. 反之, 如果消费者先接收了空的channel就会等待直至有缓存的channel.

### 带缓存的Channel

带缓存的Channel内部持有一个元素队列, 最大容量是make时指定的capcity参数, 在满时发送阻塞, 在空时获取阻塞

## Select

```go
select {
case <-ch1:
    // ...
case x := <-ch2:
    // ...use x...
case ch3 <- y:
    // ...
default:
    // ...
}
```
通过select代码块可以实现多路复用, 当存在可以执行的case时才会进入代码块执行.
本身只具备选择功能, 没法循环, 所以常和for配合使用