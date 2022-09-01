## Log 

该 Log 结合 uber 开源的高性能日志库 [zap](https://github.com/uber-go/zap) 和用以自动按时间切割日志的 [file-rotatelogs](https://github.com/lestrrat-go/file-rotatelogs) 开发而成。



## Feature

- 生成自定义日志文件
- 每日自动切割日志文件
- 控制台和文件均输出日志信息
- ERROR 日志触发监控系统记录



## Getting Started

### Required

 [zap](https://github.com/uber-go/zap) 

 [file-rotatelogs](https://github.com/lestrrat-go/file-rotatelogs) 



### Usage

Log 库提供了  `Debug`、`Info`、`Warn`、`Error`、`DPanic`、`Panic`、`Fatal` 等结构化的日志输出接口。

```go
import "git.huoys.com/middle-end/kratos/v2/log"

log.Info("args")
log.Warn("args")
log.Error("args")
```

也提供了  `Debugf`、`Infof`、`Warnf`、`Errorf`、`DPanicf`、`Panicf`、`Fatalf` 等 `printf` 风格的日志输出接口。

```go
log.Infof("template %s","args")
log.Warnf("template %s","args")
log.Errorf("template %s","args")
```

另外还提供了给业务项目使用的 `Custom` 特有接口，可自定义日志名称记录日志信息。

```go
log.Custom("Money","template %s","args")
```



将在执行处的当前位置生成的 `logs` 文件夹中生成不同级别的文件，记录对应的日志信息。



















