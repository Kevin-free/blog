# go-kratos 搭建运营分析后台



## 准备工作

环境准备可参看前文[环境准备]

go 相关内容可查看[官方文档](https://go.dev/doc/)

kratos 相关内容可阅读[官方文档](https://go-kratos.dev/docs/)



## 创建项目

```sh
# 在 /home/kaiwen/gocode/src/platform/operation-backend 目录下创建项目服务
# 创建项目模板
❯ kratos new operation-server
🚀 Creating service operation-server, layout repo is https://github.com/go-kratos/kratos-layout.git, please wait a moment.

Cloning into '/home/kaiwen/.kratos/repo/github.com/go-kratos/kratos-layout@main'...

CREATED operation-server/.gitignore (528 bytes)
CREATED operation-server/Dockerfile (459 bytes)
CREATED operation-server/LICENSE (1066 bytes)
CREATED operation-server/Makefile (1837 bytes)
CREATED operation-server/README.md (1062 bytes)
CREATED operation-server/api/helloworld/v1/error_reason.pb.go (4991 bytes)
CREATED operation-server/api/helloworld/v1/error_reason.proto (295 bytes)
CREATED operation-server/api/helloworld/v1/greeter.pb.go (8074 bytes)
CREATED operation-server/api/helloworld/v1/greeter.proto (684 bytes)
CREATED operation-server/api/helloworld/v1/greeter_grpc.pb.go (3560 bytes)
CREATED operation-server/api/helloworld/v1/greeter_http.pb.go (2139 bytes)
CREATED operation-server/cmd/operation-server/main.go (1718 bytes)
CREATED operation-server/cmd/operation-server/wire.go (614 bytes)
CREATED operation-server/cmd/operation-server/wire_gen.go (1074 bytes)
CREATED operation-server/configs/config.yaml (266 bytes)
CREATED operation-server/go.mod (1024 bytes)
CREATED operation-server/go.sum (18354 bytes)
CREATED operation-server/internal/biz/README.md (6 bytes)
CREATED operation-server/internal/biz/biz.go (128 bytes)
CREATED operation-server/internal/biz/greeter.go (1241 bytes)
CREATED operation-server/internal/conf/conf.pb.go (20782 bytes)
CREATED operation-server/internal/conf/conf.proto (767 bytes)
CREATED operation-server/internal/data/README.md (7 bytes)
CREATED operation-server/internal/data/data.go (478 bytes)
CREATED operation-server/internal/data/greeter.go (840 bytes)
CREATED operation-server/internal/server/grpc.go (843 bytes)
CREATED operation-server/internal/server/http.go (847 bytes)
CREATED operation-server/internal/server/server.go (150 bytes)
CREATED operation-server/internal/service/README.md (10 bytes)
CREATED operation-server/internal/service/greeter.go (700 bytes)
CREATED operation-server/internal/service/service.go (136 bytes)
CREATED operation-server/openapi.yaml (1105 bytes)
CREATED operation-server/third_party/README.md (14 bytes)
CREATED operation-server/third_party/errors/errors.proto (411 bytes)
CREATED operation-server/third_party/google/api/annotations.proto (1051 bytes)
CREATED operation-server/third_party/google/api/client.proto (3395 bytes)
CREATED operation-server/third_party/google/api/field_behavior.proto (3011 bytes)
CREATED operation-server/third_party/google/api/http.proto (15140 bytes)
CREATED operation-server/third_party/google/api/httpbody.proto (2671 bytes)
CREATED operation-server/third_party/google/protobuf/descriptor.proto (38027 bytes)
CREATED operation-server/third_party/validate/README.md (81 bytes)
CREATED operation-server/third_party/validate/validate.proto (31270 bytes)

🍺 Project creation succeeded operation-server
💻 Use the following command to start the project 👇:

$ cd operation-server
$ go generate ./...
$ go build -o ./bin/ ./...
$ ./bin/operation-server -conf ./configs

                        🤝 Thanks for using Kratos
        📚 Tutorial: https://go-kratos.dev/docs/getting-started/start
```



## 项目结构

本项目使用  `kratos new` 新建项目，使用的 [kratos-layout](https://github.com/go-kratos/kratos-layout) 结构，其中包括了开发过程中所需的配套工具链( Makefile 等)，便于开发者更高效地维护整个项目。

可参考 [Kratos 构建微服务的工程化最佳实践](https://go-kratos.dev/blog/go-project-layout/)，项目布局如下图所示。

![ddd](D:\Kevin\my-study\开发记录\images\ddd.png)

生成项目的目录结构如下：

```
~/g/s/p/operation-backend/operation-server ❯ tree --dirsfirst
.
├── api 		// 下面维护了微服务使用的proto文件以及根据它们所生成的go文件
│   └── operation
│       └── v1
│           ├── operation_grpc.pb.go
│           ├── operation_http.pb.go
│           ├── operation.pb.go
│           ├── operation.proto
│           └── operation.swagger.json
├── bin 		// 存放可执行的二进制文件（自建）
│   └── cmd
├── cmd 		// 整个项目启动的入口文件
│   ├── main.go
│   ├── wire_gen.go
│   └── wire.go // 我们使用wire来维护依赖注入
├── configs  	// 这里通常维护一些本地调试用的样例配置文件
│   ├── clickhouse_write.yaml
│   ├── clickhouse.yaml
│   └── config.yaml
├── db 			// 存放数据库相关脚本（自建）
│   ├── c_ob_result.sql
│   ├── ov_log_2022-4-25.sql
│   └── ov_log.xml
├── internal	// 该服务所有不对外暴露的代码，通常的业务逻辑都在这下面，使用internal避免错误引用
│   ├── biz     // 业务逻辑的组装层，类似 DDD 的 domain 层，data 类似 DDD 的 repo，而 repo 接口在这里定义，使用依赖倒置的原则。
│   │   ├── biz.go
│   │   ├── operation.go
│   │   └── README.md
│   ├── conf 	// 内部使用的config的结构定义，使用proto格式生成
│   │   ├── conf.pb.go
│   │   └── conf.proto
│   ├── data    // 业务数据访问，包含 cache、db 等封装，实现了 biz 的 repo 接口。我们可能会把 data 与 dao 混淆在一起，data 偏重业务的含义，它所要做的是将领域对象重新拿出来，我们去掉了 DDD 的 infra层。
│   │   ├── ck_test.go
│   │   ├── data.go
│   │   ├── job_daily_result.go
│   │   ├── operation.go
│   │   ├── README.md
│   │   ├── sql_ob_result.go
│   │   └── sql_operation_backend.go
│   ├── server  // http和grpc实例的创建和配置
│   │   ├── grpc.go
│   │   ├── http.go
│   │   └── server.go
│   └── service  // 实现了 api 定义的服务层，类似 DDD 的 application 层，处理 DTO 到 biz 领域实体的转换(DTO -> DO)，同时协同各类 biz 交互，但是不应处理复杂逻辑
│       ├── job.go
│       ├── operation.go
│       ├── README.md
│       └── service.go
├── log			// 服务日志存放位置（自建）
│   └── zap.log
├── pkg			// 该服务可对外暴露的代码（自建）
│   └── logger
│       └── zap.go
├── third_party  // api 依赖的第三方proto
│   ├── errors
│   │   └── errors.proto
│   ├── google
│   │   ├── api
│   │   │   ├── annotations.proto
│   │   │   ├── client.proto
│   │   │   ├── field_behavior.proto
│   │   │   ├── httpbody.proto
│   │   │   └── http.proto
│   │   └── protobuf
│   │       └── descriptor.proto
│   ├── validate
│   │   ├── README.md
│   │   └── validate.proto
│   └── README.md
├── Dockerfile
├── go.mod
├── go.sum
├── LICENSE
├── Makefile
├── openapi.yaml
└── README.md
```





## 快速开始

### 1 定义 api

#### 1.1 编写 proto 文件

在 proto 文件中定义 api 接口

```protobuf
// api/operation/v1/operation.proto

syntax = "proto3";

package api.operation;

import "google/api/annotations.proto";

option go_package = "operation-server/api/operation;operation";
option java_multiple_files = true;
option java_package = "api.operation";

service Operation {
  // 查询新增玩家
  rpc GetRegisterCount (GetRegisterCountReq) returns (GetRegisterCountRsp) {
    option (google.api.http) = {
      get: "/v1/RegisterCount"
    };
  };
}

// 通用请求
message CommonReq {
  // 开始日期
  string startDate = 1;
  // 结束日期
  string endDate = 2;
  // 系统（预留字段）
  uint32 system = 3;
  // 渠道（预留字段）
  uint32 channel = 4;
  // 广告渠道（预留字段）
  uint32 adChannel = 5;
}
// 查询新增玩家-请求
message GetRegisterCountReq {
  // 通用请求
  CommonReq commonReq = 1;
  // 新增类型（0-账户、1-设备）
  uint32 registerType = 2;
}
// 查询新增玩家-返回
message GetRegisterCountRsp {
  // 结果日期数组
  repeated string resultDates = 1;
  // 新增数组
  repeated uint64 registerNums = 2;
}
```

#### 1.2 生成 proto 代码

```
# 可以直接通过 make 命令生成
make api

# 或使用 kratos cli 进行生成
kratos proto client api/operation/v1/operation.proto
```

会在proto文件同目录下生成:

```bash
api/operation/v1/operation.pb.go
api/operation/v1/operation_grpc.pb.go
# 注意 http 代码只会在 proto 文件中声明了 http 时才会生成
api/operation/v1/operation_http.pb.go
```

#### 1.3 生成 Service 代码

通过 proto文件，可以直接生成对应的 Service 实现代码：

使用 `-t` 指定生成目录

```bash
kratos proto server api/operation/v1/operation.proto -t internal/service
```

输出:
internal/service/operation.go

> 提示：如果 internal/service/operation.go 已经存在，则不会生成：
>
> Operation already exists: internal/service/operation.go
>
> 生成的代码只是方法框架，具体业务代码得自己实现。



### 2 实现 api 接口

#### 2.1 service 层

> 实现了 api 定义的服务层，类似 DDD 的 application 层，处理 DTO 到 biz 领域实体的转换(DTO -> DO)，同时协同各类 biz 交互，但是不应处理复杂逻辑

```go
// internal/service/operation.go

package service

import (
	"context"
	pb "operation-server/api/operation/v1"
	"operation-server/internal/biz"

	"github.com/go-kratos/kratos/v2/log"
	"github.com/robfig/cron/v3"
)

type OperationService struct {
	pb.UnimplementedOperationServer
	log  *log.Helper
	oc   *biz.OperationCase
	cron *cron.Cron
}

func NewOperationService(oc *biz.OperationCase, logger log.Logger) *OperationService {
	service := &OperationService{
		oc:  oc,
		log: log.NewHelper(logger),
	}
	service.cron = cron.New(cron.WithSeconds())
	// 开启定时任务
	//go service.JobStart()
	return service
}

// GetRegisterCount 查询新增玩家
func (s *OperationService) GetRegisterCount(ctx context.Context, req *pb.GetRegisterCountReq) (rsp *pb.GetRegisterCountRsp, err error) {
	s.log.Infof("GetRegisterCount req %v", req)
	rsp = &pb.GetRegisterCountRsp{}
	if req == nil || req.CommonReq == nil {
		return
	}
	rsp.ResultDates, rsp.RegisterNums, err = s.oc.GetRegisterCountCase(ctx, req)
	s.log.Infof("GetRegisterCount rsp: %v", rsp)
	return
}
```



#### 2.2 biz 层

> 业务逻辑的组装层，类似 DDD 的 domain 层，data 类似 DDD 的 repo，而 repo 接口在这里定义，使用依赖倒置的原则。



#### 2.3 data 层

> 业务数据访问，包含 cache、db 等封装，实现了 biz 的 repo 接口。我们可能会把 data 与 dao 混淆在一起，data 偏重业务的含义，它所要做的是将领域对象重新拿出来，我们去掉了 DDD 的 infra层。





### 3 自测联调





### 4 部署运行





## FAQ























