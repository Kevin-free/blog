# [译]Go Concurrency Patterns：Context



英文原文：https://blog.golang.org/context

作者：*Sameer Ajmani*

译者：Kevin_涛



[TOC]



### 简介

在Go语言的服务器程序中，每个接收到的请求都由一个单独的goroutine处理，处理请求时通常会再创建其他的goroutine访问后端服务，例如数据库或者RPC服务。因此，每个请求通常会有一个处理请求的goroutine集合，这些goroutine需要能够访问这个请求的参数，例如：终端用户的身份、验证令牌、请求超时时间等。当一个请求超时或者被取消时，这个请求相关的所有goroutine都需要马上退出，以便于系统能够马上回收资源。

在谷歌我们开发了context包，它简化了从请求到达的API入口向这个请求相关的所有goroutine传递当前请求的参数值、goroutine取消信号、请求超时时间等。context包目前发布在[这里](https://golang.org/pkg/context)，这篇blog会通过一个实际例子来描述如何使用它。



### Context

context包的核心是`Context`类型，下面介绍为了便于理解进行了一些简化，以[godoc](https://golang.org/pkg/context)为准。

```go
// 上下文携带了截止时间，取消信号和跨 API 边界的请求范围元数据。
// 上下文的方法被多个 goroutine 同时调用是安全的。
type Context interface {
    // Done 会返回一个 channel，这个 channel 会在当前工作的上下文取消后被关闭。
    Done() <-chan struct{}

    // Err 表示了在 Done 关闭后这个上下文被取消的原因。
    Err() error

    // Deadline 返回此上下文将被取消的时间（如果有）。
    Deadline() (deadline time.Time, ok bool)

    // Value 返回键关联的值，如果没有返回 nil 。
    Value(key interface{}) interface{}
}
```



Done 方法返回一个通道，该通道作为代表上下文运行的函数的取消信号：当通道关闭时，函数应该放弃它们的工作并返回。Err 方法返回一个错误，指示取消上下文的原因。 [Pipelines and Cancelation](https://blog.golang.org/pipelines) 文章更详细地讨论了 Done 通道。

Context 没有 Cancel 方法，原因与 Done 通道是只接收的原因相同：接收取消信号的函数通常不是发送信号的函数。 特别是，当父操作为子操作启动 goroutine 时，这些子操作应该无法取消父操作。 然而，WithCancel 函数（如下所述）提供了一种取消新 Context 值的方法。

Context 被多个 goroutines 同时使用是安全的。代码可以传递单个 Context 到任意数量的 goroutine 并且取消上下文以向所有发信号。

Deadline 方法允许函数决定它们是否应该开始工作； 如果剩下的时间太少，可能就不值得了。 代码还可以使用截止日期来设置 I/O 操作的超时时间。

Value 允许上下文携带请求范围的数据。 该数据必须是安全的，可供多个 goroutine 同时使用。



### 派生上下文

context 包提供了从现有值派生新上下文值的函数。 这些值形成一棵树：当一个 Context 被取消时，所有从它派生的 Context 也被取消。

`Background`是任何上下文树的根，它永远不会被取消：

```go
// Background returns an empty Context. It is never canceled, has no deadline,
// and has no values. Background is typically used in main, init, and tests,
// and as the top-level Context for incoming requests.
func Background() Context
```

WithCancel 和 WithTimeout 返回派生的 Context 值，这些值可以比父 Context 更早取消。 与传入请求关联的上下文通常在请求处理程序返回时被取消。 WithCancel 也可用于在使用多个副本时取消冗余请求。 WithTimeout 可用于设置后端服务器请求的截止日期：

```go
// WithCancel returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed or cancel is called.
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// A CancelFunc cancels a Context.
type CancelFunc func()

// WithTimeout returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed, cancel is called, or timeout elapses. The new
// Context's Deadline is the sooner of now+timeout and the parent's deadline, if
// any. If the timer is still running, the cancel function releases its
// resources.
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

WithValue 提供了一种将请求范围的值与上下文相关联的方法：

```go
// WithValue returns a copy of parent whose Value method returns val for key.
func WithValue(parent Context, key interface{}, val interface{}) Context
```

查看如何使用 context 包的最佳方法是通过一个工作示例。



### Example: Google Web Search

我们的示例是一个 HTTP 服务器，它通过将查询“golang”转发到 Google Web Search API 并呈现结果来处理 /search?q=golang&timeout=1s 等 URL。 timeout 参数告诉服务器在该持续时间过去后取消请求。

代码分为三个包：

- server 为 /search 提供 main 函数和处理程序。
- userip 提供了从请求中提取用户 IP 地址并将其与上下文关联的功能。
- google 提供了用于向 Google 发送查询的搜索功能。



### The server program

服务器程序通过为 golang 提供前几个谷歌搜索结果来处理像 /search?q=golang 这样的请求。 它注册 handleSearch 来处理 /search 端点。 处理程序创建一个名为 ctx 的初始上下文，并安排在处理程序返回时取消它。 如果请求包含超时 URL 参数，则在超时结束时自动取消 Context：

```go
func handleSearch(w http.ResponseWriter, req *http.Request) {
    // ctx is the Context for this handler. Calling cancel closes the
    // ctx.Done channel, which is the cancellation signal for requests
    // started by this handler.
    var (
        ctx    context.Context
        cancel context.CancelFunc
    )
    timeout, err := time.ParseDuration(req.FormValue("timeout"))
    if err == nil {
        // The request has a timeout, so create a context that is
        // canceled automatically when the timeout expires.
        ctx, cancel = context.WithTimeout(context.Background(), timeout)
    } else {
        ctx, cancel = context.WithCancel(context.Background())
    }
    defer cancel() // Cancel ctx as soon as handleSearch returns.
```

处理程序从请求中提取查询并通过调用 `userip` 包提取客户端的 IP 地址。 后端请求需要客户端的IP地址，因此`handleSearch`将其附加到`ctx`：

```go
    // Check the search query.
    query := req.FormValue("q")
    if query == "" {
        http.Error(w, "no query", http.StatusBadRequest)
        return
    }

    // Store the user IP in ctx for use by code in other packages.
    userIP, err := userip.FromRequest(req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    ctx = userip.NewContext(ctx, userIP)
```

处理程序使用 `ctx` 和 `query` 调用 `google.Search`：

```go
    // Run the Google search and print the results.
    start := time.Now()
    results, err := google.Search(ctx, query)
    elapsed := time.Since(start)
```

如果查询成功，处理程序渲染结果：

```go
    if err := resultsTemplate.Execute(w, struct {
        Results          google.Results
        Timeout, Elapsed time.Duration
    }{
        Results: results,
        Timeout: timeout,
        Elapsed: elapsed,
    }); err != nil {
        log.Print(err)
        return
    }
```



### Package userip

[userip](https://blog.golang.org/context/userip/userip.go)包提供了从请求中提取用户 IP 地址并将其与上下文关联的方法。`Cotext`提供了键和值都是 `interface{}`类型。键类型必须支持相等，并且值必须可以安全地被多个 goroutine 同时使用。像 userip 这样的包隐藏了这个映射的细节，并提供了对特定上下文值的强类型访问。

为避免键冲突，userip 定义了一个未导出的类型键并使用该类型的值作为上下文键：

```go
// The key type is unexported to prevent collisions with context keys defined in
// other packages.
type key int

// userIPkey is the context key for the user IP address.  Its value of zero is
// arbitrary.  If this package defined other context keys, they would have
// different integer values.
const userIPKey key = 0
```

FromRequest 从 http.Request 中提取 userIP 值：

```go
func FromRequest(req *http.Request) (net.IP, error) {
    ip, _, err := net.SplitHostPort(req.RemoteAddr)
    if err != nil {
        return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
    }
```

NewContext 返回一个带有提供的 userIP 值的新上下文：

```go
func NewContext(ctx context.Context, userIP net.IP) context.Context {
    return context.WithValue(ctx, userIPKey, userIP)
}
```

FromContext 从上下文中提取用户 IP：

```go
func FromContext(ctx context.Context) (net.IP, bool) {
    // ctx.Value returns nil if ctx has no value for the key;
    // the net.IP type assertion returns ok=false for nil.
    userIP, ok := ctx.Value(userIPKey).(net.IP)
    return userIP, ok
}
```





## Related articles

- [Contexts and structs](https://blog.golang.org/context-and-structs)
- [Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines)
- [Introducing the Go Race Detector](https://blog.golang.org/race-detector)
- [Advanced Go Concurrency Patterns](https://blog.golang.org/io2013-talk-concurrency)
- [Concurrency is not parallelism](https://blog.golang.org/waza-talk)
- [Go videos from Google I/O 2012](https://blog.golang.org/io2012-videos)
- [Go Concurrency Patterns: Timing out, moving on](https://blog.golang.org/concurrency-timeouts)
- [Share Memory By Communicating](https://blog.golang.org/codelab-share)































