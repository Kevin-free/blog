# [译]Data Race Detector



原文：https://golang.org/doc/articles/race_detector



[TOC]



### 介绍

数据竞争是并发系统中最常见和最难调试的错误类型之一。当两个 goroutine 并发地访问同一个变量并且至少有一个访问是写入时，就会发送数据竞争。详细信息请参阅 [Go 内存模型](https://golang.org/ref/mem)。

下面是一个能导致崩溃和内存损坏的数据竞争的例子:

```go
func main() {
	c := make(chan bool)
	m := make(map[string]string)
	go func() {
		m["1"] = "a" // 第一个访问冲突
		c <- true
	}()
	m["2"] = "b" // 第二个访问冲突
	<-c
	for k, v := range m {
		fmt.Println(k, v)
	}
}
```



### 用法

为了帮助诊断这些错误，Go 包含了一个内置的数据竞争检测器。要使用它，在 go 命令中添加-race 标志:

```
$ go test -race mypkg    // to test the package
$ go run -race mysrc.go  // to run the source file
$ go build -race mycmd   // to build the command
$ go install -race mypkg // to install the package
```



### 报告格式

当竞争检测器在程序中发现一个数据竞争时，它会打印一个报告。报告包含冲突访问的堆栈跟踪，以及在其中创建相关 goroutine 的堆栈。下面是一个例子:

```
WARNING: DATA RACE
Read by goroutine 185:
  net.(*pollServer).AddFD()
      src/net/fd_unix.go:89 +0x398
  net.(*pollServer).WaitWrite()
      src/net/fd_unix.go:247 +0x45
  net.(*netFD).Write()
      src/net/fd_unix.go:540 +0x4d4
  net.(*conn).Write()
      src/net/net.go:129 +0x101
  net.func·060()
      src/net/timeout_test.go:603 +0xaf

Previous write by goroutine 184:
  net.setWriteDeadline()
      src/net/sockopt_posix.go:135 +0xdf
  net.setDeadline()
      src/net/sockopt_posix.go:144 +0x9c
  net.(*conn).SetDeadline()
      src/net/net.go:161 +0xe3
  net.func·061()
      src/net/timeout_test.go:616 +0x3ed

Goroutine 185 (running) created at:
  net.func·061()
      src/net/timeout_test.go:609 +0x288

Goroutine 184 (running) created at:
  net.TestProlongTimeout()
      src/net/timeout_test.go:618 +0x298
  testing.tRunner()
      src/testing/testing.go:301 +0xe8
```



### 可选项

GORACE 环境变量设置了竞争检测选项，格式如下:

```
GORACE="option1=val1 option2=val2"
```

可选项如下：

- `log_path` (default `stderr`): 竞争检测器将报告写到名为`log_path.pid`的文件。特殊名称 `stdout` 和 `stderr` 导致报告分别写入标准输出和标准错误。
- `exitcode` (default `66`): 当检测到竞争之后退出时使用的退出状态。
- `strip_path_prefix` (default `""`): 从所有报告的文件路径中去掉这个前缀，使报告更简洁。
- `history_size` (default `1`): 每个 goroutine 的内存访问历史是“32K * 2**history_size 个元素”。 增加此值可以避免报告中出现“无法恢复堆栈”错误，但代价是增加了内存使用量。
- `halt_on_error` (default `0`):控制程序在报告第一次数据竞争后是否退出。
- `atexit_sleep_ms` (default `1000`): 退出前在主 goroutine 中休眠的毫秒数。

示例：

```
$ GORACE="log_path=/tmp/race/report strip_path_prefix=/my/go/sources/" go test -race
```



### 排除测试

当您使用 -race 标志构建时，go 命令定义了额外的构建标记竞争。 在运行竞争检测器时，您可以使用该标签来排除一些代码和测试。 一些例子：

```go
// +build !race

package foo

// The test contains a data race. See issue 123.
func TestFoo(t *testing.T) {
	// ...
}

// The test fails under the race detector due to timeouts.
func TestBar(t *testing.T) {
	// ...
}

// The test takes too long under the race detector.
func TestBaz(t *testing.T) {
	// ...
}
```



### 如何使用

首先，使用竞争检测器运行您的测试（go test -race）。 竞争检测器只发现在运行时发生的竞争，因此它无法在未执行的代码路径中找到竞争。 如果您的测试覆盖不完整，您可能会通过在实际工作负载下运行使用 -race 构建的二进制文件来发现更多竞争。



### 典型的数据竞争

以下是一些典型的数据竞争。 所有这些都可以用竞争检测器检测到。

#### 在循环计数器上的竞争

```go
func main() {
	var wg sync.WaitGroup
	wg.Add(5)
	for i := 0; i < 5; i++ {
		go func() {
			fmt.Println(i) // Not the 'i' you are looking for.
			wg.Done()
		}()
	}
	wg.Wait()
}
```

函数字面量中的变量 i 与循环使用的变量相同，因此 goroutine 中的读取与循环增量竞争。 （该程序通常打印 55555，而不是 01234。）可以通过复制变量来修复该程序：

```go
func main() {
	var wg sync.WaitGroup
	wg.Add(5)
	for i := 0; i < 5; i++ {
		go func(j int) {
			fmt.Println(j) // Good. Read local copy of the loop counter.
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```

#### 意外的共享变量

```go
// ParallelWrite writes data to file1 and file2, returns the errors.
func ParallelWrite(data []byte) chan error {
	res := make(chan error, 2)
	f1, err := os.Create("file1")
	if err != nil {
		res <- err
	} else {
		go func() {
			// This err is shared with the main goroutine,
			// so the write races with the write below.
			_, err = f1.Write(data)
			res <- err
			f1.Close()
		}()
	}
	f2, err := os.Create("file2") // The second conflicting write to err.
	if err != nil {
		res <- err
	} else {
		go func() {
			_, err = f2.Write(data)
			res <- err
			f2.Close()
		}()
	}
	return res
}
```

解决方法是在 goroutines 中引入新变量（注意是使用 :=）：

```go
			...
			_, err := f1.Write(data)
			...
			_, err := f2.Write(data)
			...
```

#### 未受保护的全局变量 

如果从多个 goroutine 中调用以下代码，则会导致 `service` 中 map 的竞争。 同一个 map 的并发读写是不安全的：

```go
var service map[string]net.Addr

func RegisterService(name string, addr net.Addr) {
	service[name] = addr
}

func LookupService(name string) net.Addr {
	return service[name]
}
```

为了使代码安全，请使用互斥锁保护访问：

```go
var (
	service   map[string]net.Addr
	serviceMu sync.Mutex
)

func RegisterService(name string, addr net.Addr) {
	serviceMu.Lock()
	defer serviceMu.Unlock()
	service[name] = addr
}

func LookupService(name string) net.Addr {
	serviceMu.Lock()
	defer serviceMu.Unlock()
	return service[name]
}
```

#### 原始未保护变量

数据竞争也可能发生在原始类型（bool、int、int64 等）的变量上，如下例所示：

```go
type Watchdog struct{ last int64 }

func (w *Watchdog) KeepAlive() {
	w.last = time.Now().UnixNano() // First conflicting access.
}

func (w *Watchdog) Start() {
	go func() {
		for {
			time.Sleep(time.Second)
			// Second conflicting access.
			if w.last < time.Now().Add(-10*time.Second).UnixNano() {
				fmt.Println("No keepalives for 10 seconds. Dying.")
				os.Exit(1)
			}
		}
	}()
}
```

即使是这种“无害的”数据竞争也可能导致难以调试的问题，这些问题由内存访问的非原子性、编译器优化的干扰或访问处理器内存的重新排序问题引起。

此竞争的典型修复方法是使用通道或互斥锁。 为了保持无锁行为，还可以使用 [sync/atomic](https://golang.org/pkg/sync/atomic/) 包。

```go
type Watchdog struct{ last int64 }

func (w *Watchdog) KeepAlive() {
	atomic.StoreInt64(&w.last, time.Now().UnixNano())
}

func (w *Watchdog) Start() {
	go func() {
		for {
			time.Sleep(time.Second)
			if atomic.LoadInt64(&w.last) < time.Now().Add(-10*time.Second).UnixNano() {
				fmt.Println("No keepalives for 10 seconds. Dying.")
				os.Exit(1)
			}
		}
	}()
}
```

#### 不同步的发送和关闭操作

如本示例所示，同一通道上的非同步发送和关闭操作也可能是竞争条件：

```go
c := make(chan struct{}) // or buffered channel

// The race detector cannot derive the happens before relation
// for the following send and close operations. These two operations
// are unsynchronized and happen concurrently.
go func() { c <- struct{}{} }()
close(c)
```

根据 Go 内存模型，通道上的发送发生在来自该通道的相应接收完成之前。 要同步发送和关闭操作，请使用接收操作来保证在关闭之前完成发送：

```go
c := make(chan struct{}) // or buffered channel

go func() { c <- struct{}{} }()
<-c
close(c)
```



### 支持的系统

竞争检测器可以运行在 `linux/amd64`, `linux/ppc64le`, `linux/arm64`, `freebsd/amd64`, `netbsd/amd64`, `darwin/amd64`, `darwin/arm64`, and `windows/amd64`.



### 运行时开销

竞争检测的成本因程序而异，但对于典型程序，内存使用量可能会增加 5-10 倍，执行时间会增加 2-20 倍。

竞争检测器当前为每个 defer 和 recovery 语句分配额外的 8 个字节。 在 goroutine 退出之前，不会恢复这些额外的分配。 这意味着如果你有一个长期运行的 goroutine 定期发出延迟和恢复调用，程序内存使用量可能会无限增长。 这些内存分配不会显示在 runtime.ReadMemStats 或 runtime/pprof 的输出中。





