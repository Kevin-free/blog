# 保姆级教程 | 在 Go 中结合 Zap 实现定制日志库



## 前言

在实际的 Go 项目中，我们经常需要一个好的日志记录器能够提供以下功能：

- 能够打印基本信息，如调用文件、函数名和行号，日志时间等。
- 能够将事件记录到文件中，而不只是应用程序控制台。
- 支持不同的日志级别。例如 INFO，DEBUG，ERROR 等。
- 日志切割-能够根据文件大小、时间或间隔等来切割日志文件。
- 可以生成自定义名称的日志文件。

多数项目都需要前 3 项功能，并且前 3 项功能只需结合 zap 即可实现。

然而还有一些需求需要根据文件大小或时间进行日志切割，以及根据业务指标生成自定义名称的日志文件，这除了使用 zap 还要依赖第三方包来实现。

为了循序渐进，这篇文章将分享以下几个方面：

1. Go 原生日志库的使用和优缺点
2. Zap 日志库的介绍和使用
3. 搭配第三方库实现日志切割



## 默认的 Go Logger

在我们开始定制自己的日志库之前，让我们先看看 Go 语言提供的基本日志功能。Go 提供的默认日志包是 https://golang.org/pkg/log/。

### 实现 Go Logger

实现一个 Go 语言中的日志记录器非常简单：创建一个新的日志文件，然后设置它为日志的输出位置。

#### 设置 Logger

我们可以像下面的代码一样设置日志记录器

```go
func SetupLogger() {
    logFileLocation, _ := os.OpenFile("/Users/kevin/test.log", os.O_CREATE|os.O_APPEND|os.O_RDWR, 0744)
    log.SetOutPut(logFileLoaction)
}
```

#### 使用 Logger

让我们来写一些虚拟的代码来使用这个日志记录器。

在下面的示例中，我们将建立一个到URL的HTTP连接，并将状态代码/错误记录到日志文件中。

```go
func simpleHttpGet(url string) {
	resp, err := http.Get(url)
	if err != nil {
		log.Printf("Error fetching url %s : %s", url, err.Error())
	} else {
		log.Printf("Status Code for %s : %s", url, resp.Status)
		resp.Body.Close()
	}
}
```

#### Logger的运行

现在让我们执行上面的代码并查看日志记录器的运行情况。

```go
func main() {
	SetupLogger()
	simpleHttpGet("www.google.com")
	simpleHttpGet("http://www.google.com")
}
```

当我们执行上面的代码，我们能看到一个`test.log`文件被创建，下面的内容会被添加到这个日志文件中。

```bash
2019/05/24 01:14:13 Error fetching url www.google.com : Get www.google.com: unsupported protocol scheme ""
2019/05/24 01:14:14 Status Code for http://www.google.com : 200 OK
```

### Go Logger 的优点和缺点

#### 优点

它最大的优点是使用非常简单。我们可以设置任何 `io.Writer`作为日志记录输出并向其发送要写入的日志。

#### 缺点

- 仅限基本的日志级别

  - 只有一个 `Print` 选项。不支持 `Info/Debug` 等多个级别

  > 对于错误日志，它有 `Fatal` 和 `Panic`
  >
  > - Fatal 日志通过调用 `os.Exit(1)`来结束程序
  > - Panic 日志是在写入日志消息之后抛出一个` panic`
  > - 但是它缺少一个 Error 日志级别，这个级别可以在不抛出 panic 或退出程序的情况下记录错误

- 缺乏日志格式化的能力——例如记录调用者的函数名和行号，格式化日期和时间，等等。
- 不提供日志切割的能力。



## Uber-go Zap

[Zap](https://github.com/uber-go/zap) 是 Uber 开源的非常快的、结构化的，分日志级别的 Go 日志库。

对于记录热路径日志的应用程序，基于反射的序列化和字符串格式化的成本高得令人望而却步——它们占用大量 CPU 资源并进行许多小分配。 换句话说，使用 json.Marshal 和 fmt.Fprintf 来记录大量接口 interface{}会使您的应用变慢。

Zap 采用了不同的方法。 它包括一个无反射、零分配的 JSON 编码器，并且基础 Logger 力求尽可能避免序列化开销和分配。 通过在此基础上构建高级 SugaredLogger，zap 允许用户选择何时需要计算每次分配以及何时更喜欢更熟悉的松散类型的 API。

### 选择 Logger

在性能不错但不重要的情况下，请使用 SugaredLogger。 它比其他结构化日志记录包快 4-10 倍，并支持结构化和 printf 风格的日志记录。 与 log15 和 go-kit 一样，SugaredLogger 的结构化日志 API 是松散类型的，并接受可变数量的键值对。 （对于更高级的用例，它们也接受强类型字段 - 有关详细信息，请参阅 SugaredLogger.With 文档。）

```go
sugar := zap.NewExample().Sugar()
defer sugar.Sync()
sugar.Infow("failed to fetch URL",
  "url", "http://example.com",
  "attempt", 3,
  "backoff", time.Second,
)
sugar.Infof("failed to fetch URL: %s", "http://example.com")
```

默认情况下，logger 是无缓冲的。 但是，由于 zap 的低级 API 允许缓冲，因此在让您的进程退出之前调用 Sync 是一个好习惯。

在每一微秒和每一次分配都很重要的罕见情况下，使用 Logger。 它甚至比 SugaredLogger 更快，分配的资源也少得多，但它只支持强类型、结构化的日志记录。

```go
logger := zap.NewExample()
defer logger.Sync()
logger.Info("failed to fetch URL",
  zap.String("url", "http://example.com"),
  zap.Int("attempt", 3),
  zap.Duration("backoff", time.Second),
)
```

在 Logger 和 SugaredLogger 之间进行选择不需要是一个应用程序范围的决定：两者之间的转换既简单又便宜。

```go
logger := zap.NewExample()
defer logger.Sync()
sugar := logger.Sugar()
plain := sugar.Desugar()
```

### 配置 Zap

构建 Logger 的最简单方法是使用 zap 的自带的预设：NewExample、NewProduction 和 NewDevelopment。 这些预设使用单个函数调用构建记录器：

```go
logger, err := zap.NewProduction()
if err != nil {
  log.Fatalf("can't initialize zap logger: %v", err)
}
defer logger.Sync()
```

预设适用于小型项目，但较大的项目和组织自然需要更多的自定义。 对于大多数用户来说，zap 的 Config 结构在灵活性和便利性之间取得了适当的平衡。 有关示例代码，请参阅包级 [BasicConfiguration](https://pkg.go.dev/go.uber.org/zap?utm_source=godoc#example-package-BasicConfiguration) 示例。

更多特殊的配置（在文件之间拆分输出，将日志发送到消息队列等）是可能的，但需要直接使用 go.uber.org/zap/zapcore。 有关示例代码，请参阅包级 [AdvancedConfiguration](https://pkg.go.dev/go.uber.org/zap?utm_source=godoc#example-package-AdvancedConfiguration) 示例。

### 扩展 Zap

zap 包本身是围绕 go.uber.org/zap/zapcore 中的接口的一个相对较薄的包装器。 扩展 zap 以支持新的编码（例如 BSON）、新的日志接收器（例如 Kafka）或其他更奇特的东西（可能是异常聚合服务，例如 Sentry 或 Rollbar）通常需要实现 zapcore.Encoder、zapcore.WriteSyncer ，或 zapcore.Core 接口。 有关详细信息，请参阅 [zapcore](https://pkg.go.dev/go.uber.org/zap) 文档。

同样，包作者可以使用 zapcore 包中的高性能 Encoder 和 Core 实现来构建自己的记录器。

#### 将日志写入文件而不是终端

我们要做的第一个更改是把日志写入文件，而不是打印到应用程序控制台。

我们将使用 `zap.New(...)`方法来手动传递所有配置，而不是使用像 `zap.NewProduction()`这样的预设方法来创建 logger。

```go
func New(core zapcore.Core, options ...Option) *Logger
```

`zapcore.Core` 需要三个配置——`Encoder`，`WriteSyncer`，`LogLevel`。

1. **Encoder** 是日志消息的编码器。

```go
// 我们先使用开箱即用的`NewJSONEncoder()`，并使用预先设置的`ProductionEncoderConfig()`。   
zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig())
```

2. **WriterSyncer** 是支持Sync方法的io.Writer，含义是日志输出的地方，我们可以很方便的通过zapcore.AddSync将一个io.Writer转换为支持Sync方法的WriteSyncer。

```go
file, _ := os.Create("./test.log")
writeSyncer := zapcore.AddSync(file)
// 若要文件和终端都输出日志，可如下写法
//writeSyncer := zapcore.NewMultiWriteSyncer(zapcore.AddSync(file), zapcore.AddSync(os.Stdout))
```

3. **LogLevel** 是日志级别相关的参数。

```go
// 我们可以记录 Debug 及以上级别的日志。
logLevel := zapcore.DebugLevel
// 等同于下写法
//logLevel := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool { // 因为 Debug 是最低级别，所以记录的是所有级别
//    return lvl >= zapcore.DebugLevel
//})

// 只记录某种级别可以这样写
//logLevel := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool { // 只记录 Debug 级别
//    return lvl == zapcore.DebugLevel
//})
```

然后我们用一个简单的 demo 示例如何使用手动创建的 logger：

```go
var sugarLogger *zap.SugaredLogger

func main() {
	InitLogger()
	defer sugarLogger.Sync()
	simpleHttpGet("www.google.com")
	simpleHttpGet("http://www.google.com")
}

func InitLogger() {
	writeSyncer := getLogWriter()
	encoder := getEncoder()
	core := zapcore.NewCore(encoder, writeSyncer, zapcore.DebugLevel)

	logger := zap.New(core)
	sugarLogger = logger.Sugar()
}

func getEncoder() zapcore.Encoder {
	return zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig())
}

func getLogWriter() zapcore.WriteSyncer {
	file, _ := os.Create("./test.log")
	return zapcore.AddSync(file)
}

func simpleHttpGet(url string) {
	sugarLogger.Debugf("Trying to hit GET request for %s", url)
	resp, err := http.Get(url)
	if err != nil {
		sugarLogger.Errorf("Error fetching URL %s : Error = %s", url, err)
	} else {
		sugarLogger.Infof("Success! statusCode = %s for URL %s", resp.Status, url)
		resp.Body.Close()
	}
}
```

当使用这些修改过的logger配置调用上述部分的`main()`函数时，以下输出将打印在文件——`test.log`中。

```bash
{"level":"debug","ts":1572160754.994731,"msg":"Trying to hit GET request for www.sogo.com"}
{"level":"error","ts":1572160754.994982,"msg":"Error fetching URL www.sogo.com : Error = Get www.sogo.com: unsupported protocol scheme \"\""}
{"level":"debug","ts":1572160754.994996,"msg":"Trying to hit GET request for http://www.sogo.com"}
{"level":"info","ts":1572160757.3755069,"msg":"Success! statusCode = 200 OK for URL http://www.sogo.com"}
```

#### 将JSON Encoder更改为普通的Log Encoder

现在，我们希望将编码器从 JSON Encoder 更改为普通 Encoder。为此，我们需要将`NewJSONEncoder()`更改为`NewConsoleEncoder()`。

```go
return zapcore.NewConsoleEncoder(zap.NewProductionEncoderConfig())
```

当使用这些修改过的logger配置调用上述部分的`main()`函数时，以下输出将打印在文件——`test.log`中。

```bash
1.572161051846623e+09	debug	Trying to hit GET request for www.sogo.com
1.572161051846828e+09	error	Error fetching URL www.sogo.com : Error = Get www.sogo.com: unsupported protocol scheme ""
1.5721610518468401e+09	debug	Trying to hit GET request for http://www.sogo.com
1.572161052068744e+09	info	Success! statusCode = 200 OK for URL http://www.sogo.com
```

#### 更改时间编码并添加调用者详细信息

鉴于我们对配置所做的更改，有下面两个问题：

- 时间是以非人类可读的方式展示，例如1.572161051846623e+09
- 调用方函数的详细信息没有显示在日志中

我们要做的第一件事是覆盖默认的`ProductionConfig()`，并进行以下更改:

- 修改时间编码器
- 在日志文件中使用大写字母记录日志级别

```go
func getEncoder() zapcore.Encoder {
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder //指定时间格式
    // 也可如下指定时间格式
    //encoderConfig.EncodeTime = func(t time.Time, enc zapcore.PrimitiveArrayEncoder) {
	//	enc.AppendString(t.Format("2006-01-02 15:04:05"))
	//}
	encoderConfig.EncodeLevel = zapcore.CapitalLevelEncoder //设置日志级别格式：大写
	return zapcore.NewConsoleEncoder(encoderConfig)
}
```

接下来，我们将修改zap logger代码，添加将调用函数信息记录到日志中的功能。为此，我们将在`zap.New(..)`函数中添加一个`Option`。

```go
logger := zap.New(core, zap.AddCaller())
```

当使用这些修改过的logger配置调用上述部分的`main()`函数时，以下输出将打印在文件——`test.log`中。

```bash
2019-10-27T15:33:29.855+0800	DEBUG	logic/temp2.go:47	Trying to hit GET request for www.sogo.com
2019-10-27T15:33:29.855+0800	ERROR	logic/temp2.go:50	Error fetching URL www.sogo.com : Error = Get www.sogo.com: unsupported protocol scheme ""
2019-10-27T15:33:29.856+0800	DEBUG	logic/temp2.go:47	Trying to hit GET request for http://www.sogo.com
2019-10-27T15:33:30.125+0800	INFO	logic/temp2.go:52	Success! statusCode = 200 OK for URL http://www.sogo.com
```



## 搭配第三方库进行日志切割归档

这个日志程序中唯一缺少的就是日志切割归档功能。

> *Zap本身不支持切割归档日志文件*

为了添加日志切割归档功能，我们可以搭配第三方库来实现，目前我了解的有以下两种方案：

- [Lumberjack](https://github.com/natefinch/lumberjack)：可以根据文件大小分割，虽然 star 数较多，但是不符合我要根据时间分割的需求。并且网上很多关于搭配 Lumberjack 的文章，这里我就不做介绍了。
- [file-rotatelogs](https://github.com/lestrrat-go/file-rotatelogs)：可以自动按时间分割。但是作者已经不再维护此项目，如果需要也可自行实现。

### 在 zap logger 中加入 file-rotatelogs

要在 zap 中加入  file-rotatelogs 支持，我们需要修改 `WriteSyncer` ，因为 `WriteSyncer` 是 `io.Writer`，所以我们实现了如下`getWriter()`函数：

```go
// getWriter 切割归档日志
func getWriter(filename string) io.Writer {
   // 生成rotatelogs的Logger 实际生成的文件名 demo.log.YYmmddHH
   // WithLinkName 是指向最新日志的软链接 （加了会报错，未解决）
   // 保存 WithMaxAge 天内的日志，每 WithRotationTime (整点)分割一次日志
   hook, err := rotatelogs.New(
      filename+".%Y-%m-%d", // 没有使用go风格反人类的format格式
      //rotatelogs.WithLinkName(filename),
      rotatelogs.WithMaxAge(time.Hour*24*7),  // 保存 7 天
      rotatelogs.WithRotationTime(time.Hour*24), // 每日切割(note:此单位决定了log文件的时间最小精度)
   )

   if err != nil {
      panic(err)
   }
   return hook
}
```

 代码示例：

```go
package main

import (
	"fmt"
	"github.com/lestrrat/go-file-rotatelogs"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"io"
	"net/http"
	"time"
)

var sugarLogger *zap.SugaredLogger

func main() {
	fmt.Println("begin main")
	InitLogger()
	defer sugarLogger.Sync()
	simpleHttpGet("www.cnblogs.com")
	simpleHttpGet("https://www.baidu.com")
}

//例子，http访问url,返回状态
func simpleHttpGet(url string) {
	fmt.Println("begin simpleHttpGet")
	sugarLogger.Debugf("Trying to hit GET request for %s", url)
	resp, err := http.Get(url)
	if err != nil {
		sugarLogger.Errorf("Error fetching URL %s : Error = %s", url, err)
	} else {
		sugarLogger.Infof("Success! statusCode = %s for URL %s", resp.Status, url)
		resp.Body.Close()
	}
}

func InitLogger() {
	encoder := getEncoder()

	//两个interface,判断日志等级
	//warnlevel以下归到info日志
	infoLevel := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool {
		return lvl < zapcore.WarnLevel
	})
	//warnlevel及以上归到warn日志
	warnLevel := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool {
		return lvl >= zapcore.WarnLevel
	})
	infoWriter := getWriter("./logs/demo_info.log")
	warnWriter := getWriter("./logs/demo_warn.log")

	//创建zap.Core,for logger
	core := zapcore.NewTee(
		zapcore.NewCore(encoder, zapcore.AddSync(infoWriter), infoLevel),
		zapcore.NewCore(encoder, zapcore.AddSync(warnWriter), warnLevel),
	)
	//生成Logger
	logger := zap.New(core, zap.AddCaller())
	sugarLogger = logger.Sugar()
}

//getEncoder
func getEncoder() zapcore.Encoder {
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
	encoderConfig.EncodeLevel = zapcore.CapitalLevelEncoder
	return zapcore.NewConsoleEncoder(encoderConfig)
}

//得到LogWriter
func getLogWriter(filePath string) zapcore.WriteSyncer {
	warnIoWriter := getWriter(filePath)
	return zapcore.AddSync(warnIoWriter)
}

//日志文件切割
func getWriter(filename string) io.Writer {
	// 保存日志30天，每24小时分割一次日志
	/*
	hook, err := rotatelogs.New(
		filename+"_%Y%m%d.log",
		rotatelogs.WithLinkName(filename),
		rotatelogs.WithMaxAge(time.Hour*24*30),
		rotatelogs.WithRotationTime(time.Hour*24),
	)
	*/
	//保存日志30天，每1分钟分割一次日志
	hook, err := rotatelogs.New(
		filename+"_%Y%m%d%H%M.log",
        // 加了下行会报错：failed to rotate: failed to create new symlink: symlink ... A required privilege is not held by the client.
 		//怀疑是 rotatelogs 不再维护的原因
		//rotatelogs.WithLinkName(filename), 
		rotatelogs.WithMaxAge(time.Hour*24*30),
		rotatelogs.WithRotationTime(time.Minute*1),
	)
	if err != nil {
		panic(err)
	}
	return hook
}

```

1. 运行代码，控制台输出:

```bash
begin main
begin simpleHttpGet:www.cnblogs.com
begin simpleHttpGet:https://www.baidu.com
```

两个url地址中,第一个缺少协议，所以会生成错误日志,写到warn日志中，
另一个地址正确，会写到info日志中

2. 查看日志文件，已根据时间间隔自动切割

--logs

----demo_info.log_202109012016.log

----demo_info.log_202109012017.log

----demo_error.log_202109012016.log

----demo_error.log_202109012017.log



## 生成自定义名称的日志文件

有时可能需要根据业务需求生成指定的日志文件（如对于金币业务的日志统一记录在 Money 开头的日志文件中），这时我们可以添加一个`newLoggerByName()`方法根据传入的名称生成新的 Logger。

代码如下：

```go
// newLoggerByName 根据日志名称创建一个新的logger
func newLoggerByName(name string) *zap.SugaredLogger {
   // 设置一些日志格式
   encoderConfig := zap.NewProductionEncoderConfig()
   encoderConfig.EncodeTime = func(t time.Time, enc zapcore.PrimitiveArrayEncoder) { //指定时间格式
      enc.AppendString(t.Format("2006-01-02 15:04:05"))
   }
   encoderConfig.EncodeLevel = zapcore.CapitalLevelEncoder //设置日志级别格式：大写
   //获取编码器 NewConsoleEncoder() 输出普通文本格式
   encoder := zapcore.NewConsoleEncoder(encoderConfig)
   //日志级别
   logLevel := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool { // 所有级别
      return lvl >= zapcore.DebugLevel
   })

   // 获取 info、error日志文件的io.Writer 抽象 getWriter() 在下方实现
   logWriter := getWriter("./logs/" + name + ".log")

   // 最后创建具体的Logger
   core := zapcore.NewTee(
      // Encoder:编码器(如何写入日志)， WriterSyncer ：指定日志将写到哪里去， Log Level：哪种级别的日志将被写入
      zapcore.NewCore(encoder, zapcore.NewMultiWriteSyncer(zapcore.AddSync(logWriter), zapcore.AddSync(os.Stdout)), logLevel),
   )

   log := zap.New(core, zap.AddCaller()) // 需要传入 zap.AddCaller() 才会显示打日志点的文件名和行数, 有点小坑
   newSugaredLogger := log.Sugar()
   return newSugaredLogger
}
```



## 封装

为了其他项目调用，我们将日志库进行封装。

完整代码如下：

```go
package logger

import (
	"github.com/lestrrat-go/file-rotatelogs"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"io"
	"os"
	"runtime"
	"strconv"
	"time"
	"zaplog_demo/metric"
)

// SugaredLogger 性能是 logging 的 4-10 倍，可以使用结构化和 printf 风格的 API
// Logger 性能最优，但是只能使用结构化的 API
var sugaredLogger *zap.SugaredLogger

// init init a default logger
func init() {
	// 设置一些日志格式
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.EncodeTime = func(t time.Time, enc zapcore.PrimitiveArrayEncoder) { //指定时间格式
		enc.AppendString(t.Format("2006-01-02 15:04:05"))
	}
	encoderConfig.EncodeLevel = zapcore.CapitalLevelEncoder //设置日志级别格式：大写
	//获取编码器 NewConsoleEncoder() 输出普通文本格式
	encoder := zapcore.NewConsoleEncoder(encoderConfig)

	//日志级别
	infoLevel := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool { // INFO 级别
		return lvl == zapcore.InfoLevel
	})
	warnLevel := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool { // WARN 级别
		return lvl == zapcore.WarnLevel
	})
	errorLevel := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool { // ERROR 及以上级别
		return lvl >= zapcore.ErrorLevel
	})

	// 获取 info、error日志文件的io.Writer 抽象 getWriter() 在下方实现
	infoWriter := getWriter("./logs/demo_info.log")
	warnWriter := getWriter("./logs/demo_warn.log")
	errorWriter := getWriter("./logs/demo_error.log")

	// 最后创建具体的Logger
	core := zapcore.NewTee(
		// Encoder:编码器(如何写入日志)， WriterSyncer ：指定日志将写到哪里去， Log Level：哪种级别的日志将被写入
		zapcore.NewCore(encoder, zapcore.NewMultiWriteSyncer(zapcore.AddSync(infoWriter), zapcore.AddSync(os.Stdout)), infoLevel),
		zapcore.NewCore(encoder, zapcore.NewMultiWriteSyncer(zapcore.AddSync(warnWriter), zapcore.AddSync(os.Stdout)), warnLevel),
		zapcore.NewCore(encoder, zapcore.NewMultiWriteSyncer(zapcore.AddSync(errorWriter), zapcore.AddSync(os.Stdout)), errorLevel),
	)

	log := zap.New(core, zap.AddCaller()) // 需要传入 zap.AddCaller() 才会显示打日志点的文件名和行数, 有点小坑
	sugaredLogger = log.Sugar()
}

// newLoggerByName 根据日志名称创建一个新的logger
func newLoggerByName(name string) *zap.SugaredLogger {
	// 设置一些日志格式
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.EncodeTime = func(t time.Time, enc zapcore.PrimitiveArrayEncoder) { //指定时间格式
		enc.AppendString(t.Format("2006-01-02 15:04:05"))
	}
	encoderConfig.EncodeLevel = zapcore.CapitalLevelEncoder //设置日志级别格式：大写
	//获取编码器 NewConsoleEncoder() 输出普通文本格式
	encoder := zapcore.NewConsoleEncoder(encoderConfig)
	//日志级别
	logLevel := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool { // 所有级别
		return lvl >= zapcore.DebugLevel
	})

	// 获取 info、error日志文件的io.Writer 抽象 getWriter() 在下方实现
	logWriter := getWriter("./logs/demo_" + name + ".log")

	// 最后创建具体的Logger
	core := zapcore.NewTee(
		// Encoder:编码器(如何写入日志)， WriterSyncer ：指定日志将写到哪里去， Log Level：哪种级别的日志将被写入
		zapcore.NewCore(encoder, zapcore.NewMultiWriteSyncer(zapcore.AddSync(logWriter), zapcore.AddSync(os.Stdout)), logLevel),
	)

	log := zap.New(core, zap.AddCaller()) // 需要传入 zap.AddCaller() 才会显示打日志点的文件名和行数, 有点小坑
	newSugaredLogger := log.Sugar()
	return newSugaredLogger
}

// getWriter 切割归档日志 By rotatelogs
func getWriter(filename string) io.Writer {
	// 生成rotatelogs的Logger 实际生成的文件名 demo.log.YYmmddHH
	// WithLinkName 是指向最新日志的软链接 （加了会报错，未解决）
	// 保存 WithMaxAge 天内的日志，每 WithRotationTime (整点)分割一次日志
	hook, err := rotatelogs.New(
		filename+".%Y-%m-%d", // 没有使用go风格反人类的format格式
		//rotatelogs.WithLinkName(filename),
		rotatelogs.WithMaxAge(time.Hour*24*7),  // 保存 7 天
		rotatelogs.WithRotationTime(time.Hour*24), // 每日切割(note:此单位决定了log文件的时间最小精度)
	)

	if err != nil {
		panic(err)
	}
	return hook
}

func Debug(args ...interface{}) {
	sugaredLogger.Debug(args...)
}

func Debugf(template string, args ...interface{}) {
	sugaredLogger.Debugf(template, args...)
}

func Info(args ...interface{}) {
	sugaredLogger.Info(args...)
}

func Infof(template string, args ...interface{}) {
	sugaredLogger.Infof(template, args...)
}

func Warn(args ...interface{}) {
	sugaredLogger.Warn(args...)
}

func Warnf(template string, args ...interface{}) {
	sugaredLogger.Warnf(template, args...)
}

func Error(args ...interface{}) {
	sugaredLogger.Error(args...)
}

func Errorf(template string, args ...interface{}) {
	sugaredLogger.Errorf(template, args...)
}

func DPanic(args ...interface{}) {
	sugaredLogger.DPanic(args...)
}

func DPanicf(template string, args ...interface{}) {
	sugaredLogger.DPanicf(template, args...)
}

func Panic(args ...interface{}) {
	sugaredLogger.Panic(args...)
}

func Panicf(template string, args ...interface{}) {
	sugaredLogger.Panicf(template, args...)
}

func Fatal(args ...interface{}) {
	sugaredLogger.Fatal(args...)
}

func Fatalf(template string, args ...interface{}) {
	sugaredLogger.Fatalf(template, args...)
}

// Custom 业务通用日志输出
func Custom(name string, template string, args ...interface{}) {
	newLoggerByName(name).Debugf(template, args...)
}

```

测试代码：

```go
package main

import (
   "zaplog_demo/logger"
)

func main() {
   logger.Custom("Money", "logger custom: %v", 666)

   logger.Info("logger info: %v")
   logger.Infof("logger infof: %v", 2)
   logger.Error("logger error")
}
```



## Reference

https://github.com/uber-go/zap

https://github.com/lestrrat-go/file-rotatelogs

[zap documentation](https://pkg.go.dev/go.uber.org/zap)

[在Go语言项目中使用Zap日志库](https://www.liwenzhou.com/posts/Go/zap/#autoid-1-2-2)

[zap+日志分级分文件+按时间切割日志整合demo](https://www.cnblogs.com/Me1onRind/p/10918863.html)

[go:用zap和go-file-rotatelogs实现日志的记录和日志按时间分割](https://www.cxyzjd.com/article/weixin_43881017/110200176)



[一文告诉你如何用好uber开源的zap日志库](https://tonybai.com/2021/07/14/uber-zap-advanced-usage/)

[深度 | 从Go高性能日志库zap看如何实现高性能Go组件](https://mp.weixin.qq.com/s/i0bMh_gLLrdnhAEWlF-xDw)

