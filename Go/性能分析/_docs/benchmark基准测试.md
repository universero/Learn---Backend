## 稳定的测试环境

当我们尝试去优化代码时，首先需要测试当前的性能。Go内置的testing测试框架提供了基准测试的能力，能让我们很容易的对一段代码进行测试。
为了保证测试的可重复性，需要保证测试环境的稳定性。
- 机器处于闲置状态，不执行其他任务，不和其他人共享资源
- 机器关闭节能模式
- 避免使用虚拟机和云主机，为了提高资源利用率，虚拟机的CPU和内存会超分配，机器性能不稳定
## 运行benchmark测试样例

```go
// fib.go  
package main  
  
func fib(n int) int {  
	if n == 0 || n == 1 {  
		return n  
	}  
	return fib(n-2) + fib(n-1)  
}
```

```go
// fib_test.go  
package main  
  
import "testing"  
  
func BenchmarkFib(b *testing.B) {  
	for n := 0; n < b.N; n++ {  
		fib(30) // run fib(30) b.N times  
	}  
}
```

创建如上两个样例文件。benchmark和普通的单元测试一样，位于_test.go文件中，以Benchmark开头，参数为 \*testing.B
go test 默认不允许benchmark样例，需要加上`-bench`参数， 可以传入一个正则表达式，让符合正则的样例运行
```shell
univero@univero:~/test$ go test -bench .
goos: linux
goarch: amd64
pkg: univero/test
cpu: Intel(R) Core(TM) Ultra 7 155H
BenchmarkFib-22              204           6937353 ns/op
PASS
ok      univero/test    3.127s
```
运行结果如上

## benchmarkd是如何工作的

benchmark用例的参数b \*tesing.B 的参数b.N表示这个用例需要运行的次数，且对于每个样例都不一样。b.N从1开始， 如果用例能在1s内完成，b.N的值便会增加，再次执行，越后面增加的越快。
观察上面的输出，BenchmarkFib-22中，-22即GOMAXPROCS，默认等于CPU核数，可通过-cpu参数修改，支持传入一个列表
-benchtime可以指定单次运行的时间，默认是1s，也可以用-benchtime=30x指定运行三十次
-count指定benchmarkd测试的轮次

## 内存分配情况

-benchmem参数可以度量内存分配的次数

```go
// generate_test.go  
package main  
  
import (  
	"math/rand"  
	"testing"  
	"time"  
)  
  
func generateWithCap(n int) []int {  
	rand.Seed(time.Now().UnixNano())  
	nums := make([]int, 0, n)  
	for i := 0; i < n; i++ {  
		nums = append(nums, rand.Int())  
	}  
	return nums  
}  
  
func generate(n int) []int {  
	rand.Seed(time.Now().UnixNano())  
	nums := make([]int, 0)  
	for i := 0; i < n; i++ {  
		nums = append(nums, rand.Int())  
	}  
	return nums  
}  
  
func BenchmarkGenerateWithCap(b *testing.B) {  
	for n := 0; n < b.N; n++ {  
		generateWithCap(1000000)  
	}  
}  
  
func BenchmarkGenerate(b *testing.B) {  
	for n := 0; n < b.N; n++ {  
		generate(1000000)  
	}  
}
```

```shell
univero@univero:~/test$ go test -bench="Generate" -benchmem .
goos: linux
goarch: amd64
pkg: univero/test
cpu: Intel(R) Core(TM) Ultra 7 155H
BenchmarkGenerateWithCap-22           69          22442601 ns/op         8003585 B/op          1 allocs/op
BenchmarkGenerate-22                  40          33843813 ns/op        41678491 B/op         39 allocs/op
PASS
ok      univero/test    2.964s
```

generateWithCap和generate的作用是一致的，生成一组长度为n的随机序列，但是generateWithCap创建切片时指定了容量，只需要分配一次内存，倒是效率更高

## 构造不同测试

```go
// generate_test.go  
package main  
  
import (  
	"math/rand"  
	"testing"  
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
func benchmarkGenerate(i int, b *testing.B) {  
	for n := 0; n < b.N; n++ {  
		generate(i)  
	}  
}  
  
func BenchmarkGenerate1000(b *testing.B)    { benchmarkGenerate(1000, b) }  
func BenchmarkGenerate10000(b *testing.B)   { benchmarkGenerate(10000, b) }  
func BenchmarkGenerate100000(b *testing.B)  { benchmarkGenerate(100000, b) }  
func BenchmarkGenerate1000000(b *testing.B) { benchmarkGenerate(1000000, b) }
```

```shell
univero@univero:~/test$ go test -bench="Generate" -benchmem .
goos: linux
goarch: amd64
pkg: univero/test
cpu: Intel(R) Core(TM) Ultra 7 155H
BenchmarkGenerate1000-22           41377             28658 ns/op           25208 B/op         12 allocs/op
BenchmarkGenerate10000-22           3296            324666 ns/op          357625 B/op         19 allocs/op
BenchmarkGenerate100000-22           280           4057367 ns/op         4101393 B/op         28 allocs/op
BenchmarkGenerate1000000-22           30          47276083 ns/op        41678208 B/op         39 allocs/op
PASS
ok      univero/test    6.643s
```
只需要简单的改造一下之前的样例就能实现不同输入。通过不同的输入构造可以测试代码的时间复杂度，如这个时间是线性变化的，就可以判断是O(n)

## 注意事项

如果在benchmark开始前需要一些准备工作且这些工作比较耗时，可以通过`time.Sleep(time.Second*3)`来模拟耗时
但是模拟耗时会影响到基准测试的准确性，可以用`b.ResetTimer()`重置计时器
同样，可能还会有些收尾的清理工作，所以可以用`b.StopTimer()`和`b.StartTimer()`控制计时器的行为