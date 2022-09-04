# [译]Go Concurrency Patterns: Pipelines and cancellation



英文原文：https://blog.golang.org/pipelines

作者：*Sameer Ajmani*

译者：Kevin_涛



[TOC]



#### 简介

Go 语言的并发原语使得开发者构建流式数据 `pipeline(管道)`变得容易，这些管道可以有效的利用 I/O 和多 CPU。本文介绍了这类 pipeline 的示例，重点介绍了操作失败时出现的细微差别，和如何清晰地处理错误的技术。



#### 什么是 pipeline ？

Go 语言中对`pipeline`没有标准的定义；`pipeline`只是多种并发编程中的一种。通常而言，`pipeline`是由`channel（信道）`连接的一系列阶段，其中每个阶段是一组运行相同方法的`goroutine`。在每个阶段中，`goroutine`会完成

- 通过入界 channels 从上游接收值
- 对数据进行一些处理，通常会产生新的值
- 通过出界 channels 向下游发送值

除第一个阶段和最后一个阶段外，所有的阶段中都可以有任意数量的`入界channels`和`出界channels`，它们仅有`出界channels`或`入界channels`（译者注：第一个阶段为生产者，仅有出界；最后一阶段为消费者，仅有入界）。第一个阶段通常也被称为`生产者`或`源`，最后一个阶段则被称为`消费者`或`接收器`。

我们将从一个简单的`pipeline`例子来解释这些思想和技术。后面，我们会提供更多实际的例子。



#### 平方数

思考一个有三个阶段的管道。

第一阶段，`gen`，这个函数将一个整数列表转化为一个 `channel`（用于发送列表中的整数）。`gen` 函数启动了一个 `goroutine` 用于发送整数到 `channel` 中并且当所有的值发送完之后关闭这个 `channel`。

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}
```

下图是译者便于理解所作：



![channel_01](http://kevinpub.ifree258.top/channel_01.png)



第二阶段，`sq`，从一个 channel 中接收整数并且返回一个 channel （用于发送接收值的平方）。在入界 channel 关闭之后，并且这个阶段所有的值都发送到下游后，出界 channel 将关闭。

```go
func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}
```

译者所作示意图：



![channel_02](http://kevinpub.ifree258.top/channel_02.png)



`main` 方法建立 `pipeline` 并且启动最后阶段：接收来自第二阶段的值并打印每一个，直到这个信道被关闭。

```go
func main() {
    // Set up the pipeline.
    c := gen(2, 3)
    out := sq(c)

    // Consume the output.
    fmt.Println(<-out) // 4
    fmt.Println(<-out) // 9
}
```



因为 `sq` 的入界信道和出界信道有相同的类型，我们可以对其组合任意次（译注：套娃操作）。我们也可以像其他阶段一样将  `main` 重写为 `for range` 的循环。

```go
func main() {
    // Set up the pipeline and consume the output.
    for n := range sq(sq(gen(2, 3))) {
        fmt.Println(n) // 16 then 81
    }
}
```



#### 扇出、扇入

在信道关闭之前，多个方法可以从同一个信道读取，这叫做**扇出**。这提供了一种在一组工人之间分发工作的方式，以并行使用 CPU 和 I/O。

一个方法可以从多个输入信道中读取，并合并为一个信道，当所有输入关闭时信道也被关闭。这叫做**扇入**。



![channel_03](http://kevinpub.ifree258.top/channel_03.png)



我们改变一下上面的管道，运行两个`sq`实例，每个实例均从同一个`channel`中读取。这里需要引入一个新的函数:`merge`，来实现`fan-in`打印出结果:

```go
func main() {
    in := gen(2, 3)

    // 通过两个都从 in 读取的 goroutine 分发 sq 任务
    c1 := sq(in)
    c2 := sq(in)

    // 消费合并自 c1 and c2 的输出
    for n := range merge(c1, c2) {
        fmt.Println(n) // 4 then 9, or 9 then 4
    }
}
```

`merge` 方法通过在每个入界信道中启动一个 `goroutine`复制值到单独的出界信道，将信道列表转为一个信道。一旦所有的`输出 goroutine` 启动了， `merge` 再开启一个 goroutine 来关闭这个出界信道在所有的发送结束之后。

**向一个关闭的信道发送会导致 `panic`，所以确保在调用 `close` 之前所有发送已完成是非常重要的。** [`sync.WaitGroup`](https://golang.org/pkg/sync/#WaitGroup) 提供了一种简单的方式来实现同步化：

```go
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // 为 cs 里的每个输入信道开启一个输出 goroutine 
    // 复制 c 中的值到 out 中，直到 c 关闭，然后调用 wg.Done
    output := func(c <-chan int) {
        for n := range c {
            out <- n
        }
        wg.Done()
    }
    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    // 启动一个 goroutine 负责在所有输出 goroutine 完成后关闭 out。
    // 这必须在 wg.Add 之后调用。
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```



#### 短停

对于 pipeline 有一些模式：

- 当阶段中的所有的发送操作结束时，关闭他们的出界信道。
- 阶段中保持接收来自入界信道的值，直到这些入界信道被关闭。

这种模式允许每个接收阶段写成 `range` 的循环，并确保一旦所有值都已成功发送到下游，所有的 `goroutine` 都会退出。

但是在实际的 `pipeline` 种，阶段并不总是能接收所有的入界值。有时是设计需求：为了优化只要接收值的子集。更多的情况是，一个阶段会较早的退出，因为在较早的阶段一个入界值代表了错误。在任何一种情况下，接收器都不必等待剩余的值都到达，并且我们希望较早阶段停止产生较晚阶段不需要的值。

在 `pipeline` 的例子中，如果一个阶段无法使用所有的入界值，尝试发送这些值的 `goroutine` 将无限期地阻塞。

```go
// 消费第一个来自 output 的值
out := merge(c1, c2)
fmt.Println(<-out) // 4 or 9
return
// 一旦我们没有接收到来自 out 的第二个值，
// 一个 output 的 goroutine 将被挂起尝试发送。
```

这是资源泄露：`goroutine` 消耗内存和运行时资源，`goroutine` 堆栈中的堆引用阻止数据被垃圾回收。`goroutine` 不会被垃圾回收，他们必须自己退出。

即使下游阶段无法接收所有入界值，我们也需要安排管道的上游阶段退出。 一种方法是将出界通道更改为具有缓冲区。 缓冲区可以容纳固定数量的值； 如果缓冲区中有空间，发送操作将立即完成：

```go
c := make(chan int, 2) // buffer size 2
c <- 1  // succeeds immediately
c <- 2  // succeeds immediately
c <- 3  // blocks until another goroutine does <-c and receives 1
```

当通道创建时已知要发送的值的数量时，缓冲区可以简化代码。 例如，我们可以重写`gen`以将整数列表复制到缓冲通道中，并避免创建新的`goroutine`：

```go
func gen(nums ...int) <-chan int {
    out := make(chan int, len(nums))
    for _, n := range nums {
        out <- n
    }
    close(out)
    return out
}
```

返回到管道中被阻塞的`goroutine`，我们可以考虑将缓冲区添加到`merge`返回的出界通道中：

```go
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int, 1) // enough space for the unread inputs
    // ... the rest is unchanged ...
```

虽然这可以修复该程序中被阻止的`goroutine`，但这是不好的代码。 此处缓冲区大小的选择1取决于知道合并将接收的值的数量以及下游阶段将消耗的值的数量。 这很脆弱：如果我们向`gen`传递一个附加值，或者如果下游阶段读取的值更少，那么我们将再次阻塞`goroutines`。

相反，我们需要为下游阶段提供一种方法，以向发送方指示他们将停止接受输入。



#### 明确地取消

当 `main` 决定退出不再接收来自 `out` 的所有值时，他必须告诉上游阶段的 `goroutine` 放弃他们尝试发送的值。他通常在被称为 `done` 的信道上发送值来实现。他发送两个值因为这里有两个可能被阻塞的发送器：

```go
func main() {
    in := gen(2, 3)

    // Distribute the sq work across two goroutines that both read from in.
    c1 := sq(in)
    c2 := sq(in)

    // Consume the first value from output.
    done := make(chan struct{}, 2)
    out := merge(done, c1, c2)
    fmt.Println(<-out) // 4 or 9

    // Tell the remaining senders we're leaving.
    done <- struct{}{}
    done <- struct{}{}
}
```

发送的 `goroutine` 会将其发送操作替换为 `select` 语句，该语句将在发送到 `out` 或从 `done` 接收到值时继续执行。`done` 的值类型是空结构因为其值无关紧要：接收事件表明对 `out` 的发送应该被丢弃。输出的 goroutine 继续在其入界信道 `c` 上循环，因此上游阶段不会阻塞。（稍后我们将讨论如果允许次循环尽早返回）

```go
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // Start an output goroutine for each input channel in cs.  output
    // copies values from c to out until c is closed or it receives a value
    // from done, then output calls wg.Done.
    output := func(c <-chan int) {
        for n := range c {
            select {
            case out <- n:
            case <-done:
            }
        }
        wg.Done()
    }
    // ... the rest is unchanged ...
```

这种方法有一个问题：每个下游接收方都需要知道可能被阻塞的上游发送方的数量，并安排在早期返回时向这些发送方发信号。 跟踪这些计数是乏味且容易出错的。

我们需要一种方法来告知未知数量和无限数量的`goroutine`，以停止向下游发送其值。 在 Go 中，我们可以通过关闭通道来完成此操作，因为[在关闭的通道上的接收操作始终可以立即进行，从而产生元素类型的零值](https://golang.org/ref/spec#Receive_operator)。

这意味着 `main` 可以通过关闭 `done` 通道来解除所有的发送器。 此关闭实际上是向发送方的广播信号。 我们扩展了每个管道功能以接受`done`作为参数，并通过`defer`语句延迟发生，以便从`main`返回的所有返回路径都将通知管道阶段退出。

```go
func main() {
    // Set up a done channel that's shared by the whole pipeline,
    // and close that channel when this pipeline exits, as a signal
    // for all the goroutines we started to exit.
    done := make(chan struct{})
    defer close(done)

    in := gen(done, 2, 3)

    // Distribute the sq work across two goroutines that both read from in.
    c1 := sq(done, in)
    c2 := sq(done, in)

    // Consume the first value from output.
    out := merge(done, c1, c2)
    fmt.Println(<-out) // 4 or 9

    // done will be closed by the deferred call.
}
```

现在，一旦 `done` 关闭，我们的每个管道阶段都可以自由返回。 `merge` 中的 `output` 协程可以在不耗尽其入站通道的情况下返回，因为它知道上游发送器sq会在 `done` 关闭时停止尝试发送。 `output` 通过 `defer` 关键字确保在返回路径上调用 `wg.Done` ：

```go
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // Start an output goroutine for each input channel in cs.  output
    // copies values from c to out until c or done is closed, then calls
    // wg.Done.
    output := func(c <-chan int) {
        defer wg.Done()
        for n := range c {
            select {
            case out <- n:
            case <-done:
                return
            }
        }
    }
    // ... the rest is unchanged ...
```

同样地，`sq` 可以在 `done` 关闭时立即返回。`sq` 通过 `defer` 关键字确保 `out` 信道在所有的返回路径上被关闭。

```go
func sq(done <-chan struct{}, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-done:
                return
            }
        }
    }()
    return out
}
```

以下是构建 pipeline 的指导原则：

- 当阶段所有的发送操作完成时，关闭他们的出界信道。
- 阶段会一直从入界通道接收值，直到这些通道关闭或发送方被阻止为止。

`pipeline` 可以通过确保有足够的缓冲区来存储所有发送的值，或者在接收方可能放弃通道时显式发信号通知发送方，从而解除发送方的阻塞。



最后还有将 pipeline 应用于 MD5 的例子，笔者未译。



## Related articles

- [Contexts and structs](https://blog.golang.org/context-and-structs)
- [Go Concurrency Patterns: Context](https://blog.golang.org/context)
- [Introducing the Go Race Detector](https://blog.golang.org/race-detector)
- [Advanced Go Concurrency Patterns](https://blog.golang.org/io2013-talk-concurrency)
- [Concurrency is not parallelism](https://blog.golang.org/waza-talk)
- [Go videos from Google I/O 2012](https://blog.golang.org/io2012-videos)
- [Go Concurrency Patterns: Timing out, moving on](https://blog.golang.org/concurrency-timeouts)
- [Share Memory By Communicating](https://blog.golang.org/codelab-share)


