benchmark可以度量某个函数的性能，并针对性优化，但是这个前提是我们知道程序的瓶颈在哪。对于一个未知的程序，我们需要使用pprof来分析程序的性能

pprof包含两个部分
- 编译到程序中的 `runtime/pprof`包
- 性能剖析工具 go tool pprof

## 性能分析类型

### CPU性能分析

启动CPU分析时，运行时每隔10ms中断一次，记录此时正在运行的协程堆栈信息。程序运行结束后，可以分析记录的数据找到最热代码路径


```go
package main

import (
        "math/rand"
        "os"
        "runtime/pprof"
        "time"
)

func generate(n int) []int {
        rand.Seed(time.Now().UnixNano())
        nums := make([]int, 0)
        for i := 0; i < n; i++ {
                nums = append(nums, rand.Int())
        }
        return nums
}
func bubbleSort(nums []int) {
        for i := 0; i < len(nums); i++ {
                for j := 1; j < len(nums)-i; j++ {
                        if nums[j] < nums[j-1] {
                                nums[j], nums[j-1] = nums[j-1], nums[j]
                        }
                }
        }
}

func main() {
        f, _ := os.OpenFile("cpu.pprof",os.O_CREATE|os.O_RDWR, 0644)
        defer f.Close()
        pprof.StartCPUProfile(f)
        defer pprof.StopCPUProfile()
        n := 10
        for i := 0; i < 5; i++ {
                nums := generate(n)
                bubbleSort(nums)
                n *= 10
        }
}
```

按照上述样例即可开启CPU性能分析，运行完main.go后就会得到cpu.pprof，但是文件是无法直接阅读的，需要借助pprof分析
`go tool pprof -http=:9999 cpu.pprof`，需要先安装`apt install graphviz`
也可以通过命令行交互查看，去掉http参数即可
### 内存性能分析

记录内存分配时的堆栈信息，忽略栈内存分配信息。内存性能分析启用时，每1000次采样1次(比例可调)，由于内存性能分析是基于采样的，因此基于内存分析数据判断所有的内存使用情况是很困难的。

## 阻塞性能分析

阻塞性能分析是go特有的，用来记录一个协程等待一个共享资源花费的时间，在判断并发场景的瓶颈时很有效。阻塞场景包括：
- 在没有缓冲区的信道上发送或接收数据。
- 从空的信道上接收数据，或发送数据到满的信道上。
- 尝试获得一个已经被其他协程锁住的排它锁
一般情况下CPU和内存瓶颈解决后才会考虑这个

## 锁性能分析

类似于阻塞性能分析，但专注于锁竞争导致的等待或延时