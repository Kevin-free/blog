在许多 Go 语言项目中，我们需要一个好的日志记录器能够提供下面这些功能：

- 能够将事件记录到文件中，而不是应用程序控制台。
- 日志切割-能够根据文件大小、时间或间隔等来切割日志文件。
- 支持不同的日志级别。例如 INFO，DEBUG，ERROR 等。
- 能够打印基本信息，如调用文件/函数名和行号，日志时间等。



[Zap ](https://github.com/uber-go/zap)是非常快的、结构化的，分日志级别的Go日志库。

### 为什么选择Uber-go zap

- 它同时提供了结构化日志记录和 printf 风格的日志记录
- 它非常的快

根据 Uber-go Zap 的文档，它的性能比类似的结构化日志包更好——也比标准库更快。 以下是 Zap 发布的基准测试信息

记录一条消息和10个字段:

|     Package     |    Time     | Time % to zap | Objects Allocated |
| :-------------: | :---------: | :-----------: | :---------------: |
|      ⚡️ zap      |  862 ns/op  |      +0%      |    5 allocs/op    |
| ⚡️ zap (sugared) | 1250 ns/op  |     +45%      |   11 allocs/op    |
|     zerolog     | 4021 ns/op  |     +366%     |   76 allocs/op    |
|     go-kit      | 4542 ns/op  |     +427%     |   105 allocs/op   |
|    apex/log     | 26785 ns/op |    +3007%     |   115 allocs/op   |
|     logrus      | 29501 ns/op |    +3322%     |   125 allocs/op   |
|      log15      | 29906 ns/op |    +3369%     |   122 allocs/op   |

记录一个静态字符串，没有任何上下文或printf风格的模板：

|     Package      |    Time    | Time % to zap | Objects Allocated |
| :--------------: | :--------: | :-----------: | :---------------: |
|      ⚡️ zap       | 118 ns/op  |      +0%      |    0 allocs/op    |
| ⚡️ zap (sugared)  | 191 ns/op  |     +62%      |    2 allocs/op    |
|     zerolog      |  93 ns/op  |     -21%      |    0 allocs/op    |
|      go-kit      | 280 ns/op  |     +137%     |   11 allocs/op    |
| standard library | 499 ns/op  |     +323%     |    2 allocs/op    |
|     apex/log     | 1990 ns/op |    +1586%     |   10 allocs/op    |
|      logrus      | 3129 ns/op |    +2552%     |   24 allocs/op    |
|      log15       | 3887 ns/op |    +3194%     |   23 allocs/op    |

### 安装

运行下面的命令安装zap

```bash
go get -u go.uber.org/zap
```

### 配置Zap Logger

Zap提供了两种类型的日志记录器—`Sugared Logger`和`Logger`。

在性能很好但不是很关键的上下文中，使用`SugaredLogger`。它比其他结构化日志记录包快4-10倍，并且支持结构化和printf风格的日志记录。

在每一微秒和每一次内存分配都很重要的上下文中，使用`Logger`。它甚至比`SugaredLogger`更快，内存分配次数也更少，但它只支持强类型的结构化日志记录。















### Reference

[在Go语言项目中使用Zap日志库](https://www.liwenzhou.com/posts/Go/zap/)

[Go 每日一库之 zap](https://segmentfault.com/a/1190000022461706)

[golang高性能日志库zap的使用](https://studygolang.com/articles/32205)

[zap+日志分级分文件+按时间切割日志整合demo](https://www.cnblogs.com/Me1onRind/p/10918863.html)

[一文告诉你如何用好uber开源的zap日志库](https://tonybai.com/2021/07/14/uber-zap-advanced-usage/)









