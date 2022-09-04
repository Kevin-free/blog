# [译] The Go Memory Model



原文：https://golang.org/ref/mem



[TOC]



### 介绍

Go 内存模型指定了在何种条件下可以保证在一个 goroutine 中读取变量时观察到在不同 goroutine 中写入相同变量所产生的值。



### 建议

修改多个 goroutine 同时访问的数据的程序必须序列化这种访问。

要序列化访问，请使用通道操作或其他同步原语（例如 `sync` 和 `sync/atomic` 包中的原语）保护数据。

如果您必须阅读本文档的其余部分才能了解您的程序的行为，那么您就太聪明了。

不要聪明。



### Happens Before

在单个 goroutine 中，读取和写入的行为必须像按照程序指定的顺序执行一样。也就是说，只有当重新排序不会改变语言规范定义的 goroutine 中的行为时，编译器和处理器才可以重新排序在单个 goroutine 中执行的读取和写入。由于这种重新排序，一个 goroutine 观察到的执行顺序可能与另一个 goroutine 感知的顺序不同。 例如，如果一个 goroutine 执行 a = 1; b = 2;，另一个可能会在 a 的更新值之前观察到 b 的更新值。

为了指定读和写的要求，我们定义 Go 程序中内存操作的偏序为 **happens before** 原则。 如果事件 e1 发生在事件 e2 之前，那么我们说 e2 发生在 e1 之后。 此外，如果 e1 没有发生在 e2 之前，也没有发生在 e2 之后，那么我们说 e1 和 e2 同时发生。

在单个 goroutine 中，happens-before 顺序是程序表达的顺序。

如果以下两项都成立，则允许对变量 v 的读取 r 观察到对 v 的写入 w：

1. r 不会在 w 之前发生。
2. 没有其他写 w' 到 v 发生在 w 之后但在 r 之前。

为了保证对变量 v 的读取 r 观察到对 v 的特定写入 w，请确保 w 是唯一允许观察的写入 r。 也就是说，如果以下两项都成立，则保证 r 观察 w：

1. w 发生在 r 之前。
2. 对共享变量 v 的任何其他写入要么发生在 w 之前，要么发生在 r 之后。

这对条件比第一对强； 它要求没有其他写入与 w 或 r 同时发生。

在单个 goroutine 内，没有并发，所以两个定义是等价的：一个读 r 观察最近一次写 w 写入 v 的值。当多个 goroutine 访问共享变量 v 时，它们必须使用同步事件来建立 happens-before 确保读取观察所需写入的条件。

使用 v 类型的零值初始化变量 v 的行为类似于内存模型中的写入。

对大于单个机器字的值的读取和写入表现为未指定顺序的多个机器字大小的操作。



### 同步

#### 初始化

程序初始化在单个 goroutine 中运行，但是 goroutine 可能会创建其他  goroutine ，并发运行。

如果包 p 导入包 q，则 q 的 `init` 函数在 p 的开始之前发生。

`main` 函数的开始实在所有 `init` 函数完成之后。



#### goroutine 的创建

`go` 关键词启动一个新的 goroutine 发生在 goroutine 开始执行之前。

例如，在这个程序中：

```go
var a string

func f() {
    print(a)
}

func hello() {
	a = "hello, world"
    go f()
}
```

在未来的某个时刻调用 `hello` 会打印 “hello，world“（也许在 `hello` 之后）



#### goroutine 的毁灭

goroutine  的退出不会保证发生在程序的任何事件之前。例如，下面这个程序：

```go
var a string
func hello() {
    go func(){ a = "hello" }()
    print(a)
}
```

对 a 的赋值之后不会出现任何同步事件，因此不能保证任何其他 goroutine 都能观察到它。实际上，激进的编译器可能会删除整个 go 语句。

如果一个 goroutine 的效果必须被另一个 goroutine 观察到，那么使用一个同步机制，如锁或通道通信来建立一个相对顺序。

#### 信道通信

信道通信是 goroutine 之间同步的主要方法。特定通道上的每个发送都与来自该通道的相应接收匹配，通常在不同的 goroutine 中。

*通道上的发送发生在通道完成对应的接收之前。*

这个程序：

```go
var c = make(chan int, 10)
var a string 

func f() {
	a = "hello,world"
    c <- 0
}

func main() {
    go f()
    <- c
    print(a)
}
```

保证打印 “hello，world”。对 a 的写操作发生在 对 c 的发送操作之前，这发生在 c 完成对应的接收之前，这发生在打印之前。

*通道的关闭发生在接收之前，接收返回零值，因为通道已关闭。*

在前面的示例中，用 close (c)替换 c <-0会生成具有相同保证行为的程序。

*从无缓冲通道的接收发生在该通道上的发送完成之前。*

这个程序（如上所述，但是与发送和接收语句交换并使用一个无缓冲的通道） :

```go
var c = make(chan int)
var a string

func f() {
	a = "hello, world"
	<-c
}

func main() {
	go f()
	c <- 0
	print(a)
}
```

也能保证打印“hello，world”。对 a 的写入发生在 c 上的接收之前，这发生在对应的 c 发送完成之前，这发生在打印之前。

如果是有缓冲通道（如，c = make(chan int, 1)），那么程序将不能保证打印“hello，world”。（它可能打印空字符串、崩溃或执行其他操作）

*容量为 C 的通道上的 kth 接收发送在从该通道发送的 k+Cth 完成之前。*

此规则将先前的规则推广到缓冲通道。 它允许通过缓冲通道对计数信号量进行建模：通道中的项目数对应于活动使用的数量，通道的容量对应于同时使用的最大数量，发送项目获取信号量，以及 接收一个项目释放信号量。 这是限制并发的常用习惯用法。

这个程序为工作列表的每个入口启动了一个 goroutine，但是 goroutine 使用有缓冲通道进行协调，以确保一次最多三个正在运行工作函数。

```go
var limit = make(chan int, 3)

func main() {
    for _, w := range work{
        go func(w func()){
            limit <- 1
            w()
            <- limit
        }(w)
    }
    select{}
}
```

#### 锁

`sync` 包实现了两个锁的数据类型：`sync.Mutex` 和 `sync.RWMtex`。

对于任何`sync.Mutex` 或`sync.RWMutex` 变量`l` 且n < m，`l.Unlock()` 的n 调用发生在`l.Lock()` 的调用m 返回之前。

这个程序：

```go
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```

保证打印“hello, world”。 第一次调用 l.Unlock() （在 f 中）发生在第二次调用 l.Lock() （在 main 中）返回之前，发生在打印之前。

对于在 `sync.RWMutex` 变量 l 上对 `l.RLock` 的任何调用，都有一个 n，使得 `l.RLock` 在调用 n 到 `l.Unlock` 之后发生（返回），并且匹配的 `l.RUnlock` 在调用 n+1 到 `l.Lock` 之前发生。

#### Once

`sync` 包通过使用 `Once` 类型为多个 goroutine 的初始化提供了一种安全机制。多个线程可以为特定的 `f` 执行 `once.Do(f)`，但是只有一个会运行 `f()`，并且其他调用会阻塞直到 `f()` 返回。

从 `once.Do(f)` 对 `f()` 的单个调用发生（返回）在任何对 `once.Do(f)` 的调用返回之前。

这个程序：

```go
var a string
var once sync.Once

func setup() {
	a = "hello, world"
}

func doprint() {
	once.Do(setup)
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

调用 `twoprint` 将只调用一次 `setup`。 `setup`函数将在任一`print`调用之前完成。 结果将是“hello, world”将被打印两次。



### 不正确的同步

请注意，读取 r 可能会观察到与 r 并发发生的写入 w 写入的值。 即使发生这种情况，也不意味着在 r 之后发生的读取会观察到发生在 w 之前的写入。

这个程序：

```go
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```

可能会发生 g 打印 2 然后打印 0。

这一事实使一些常见的习语无效。

双重检查锁定是一种避免同步开销的尝试。 例如， `twoprint` 程序可能被错误地写为：

```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

但不能保证，在 `doprint` 中，观察对 `done` 的写入意味着观察对 `a` 的写入。 这个版本可以（错误地）打印一个空字符串而不是“hello, world”。

另一个错误的习惯用法是忙于等待一个值，例如：

```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```

和以前一样，不能保证在 main 中，观察对 done 的写入意味着观察对 a 的写入，因此该程序也可以打印一个空字符串。 更糟糕的是，由于两个线程之间没有同步事件，因此无法保证 main 会观察到对 done 的写入。 main 中的循环不能保证完成。

这个主题有更微妙的变体，比如这个程序。

```go
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
```

即使 main 观察到 g != nil 并退出它的循环，也不能保证它会观察到 g.msg 的初始化值。

在所有这些示例中，解决方案都是相同的：使用明确的同步。







