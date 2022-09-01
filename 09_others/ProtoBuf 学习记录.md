## ProtoBuf 学习记录

### 指定字段规则

你指定的 message 字段可以是下面几种情况之一：

- **required**：格式良好的 message 必须包含该字段一次。
- **optional**：格式良好的 message 可以包含该字段零次或一次（不超过一次）。
- **repeated**：该字段可以在格式良好的消息中重复任意多次（包括零）。其中重复值的顺序会被保留。

由于一些历史原因，标量数字类型的 repeated 字段不能尽可能高效地编码。新代码应使用特殊选项 [packed = true] 来获得更高效的编码。例如：

```protobuf
repeated int32 samples = 4 [packed=true];
```



### 标量值类型

| .proto Type | Notes                                                        | C++ Type | Java Type  | Python Type[2]                       | Go Type  |
| :---------- | :----------------------------------------------------------- | :------- | :--------- | :----------------------------------- | :------- |
| double      |                                                              | double   | double     | float                                | *float64 |
| float       |                                                              | float    | float      | float                                | *float32 |
| int32       | 使用可变长度编码。编码负数的效率低 - 如果你的字段可能有负值，请改用 sint32 | int32    | int        | int                                  | *int32   |
| int64       | 使用可变长度编码。编码负数的效率低 - 如果你的字段可能有负值，请改用 sint64 | int64    | long       | int/long[3]                          | *int64   |
| uint32      | 使用可变长度编码                                             | uint32   | int[1]     | int/long[3]                          | *uint32  |
| uint64      | 使用可变长度编码                                             | uint64   | long[1]    | int/long[3]                          | *uint64  |
| sint32      | 使用可变长度编码。有符号的 int 值。这些比常规 int32 对负数能更有效地编码 | int32    | int        | int                                  | *int32   |
| sint64      | 使用可变长度编码。有符号的 int 值。这些比常规 int64 对负数能更有效地编码 | int64    | long       | int/long[3]                          | *int64   |
| fixed32     | 总是四个字节。如果值通常大于 228，则比 uint32 更有效。       | uint32   | int[1]     | int/long[3]                          | *uint32  |
| fixed64     | 总是八个字节。如果值通常大于 256，则比 uint64 更有效。       | uint64   | long[1]    | int/long[3]                          | *uint64  |
| sfixed32    | 总是四个字节                                                 | int32    | int        | int                                  | *int32   |
| sfixed64    | 总是四个字节                                                 | int64    | long       | int/long[3]                          | *int64   |
| bool        |                                                              | bool     | boolean    | bool                                 | *bool    |
| string      | 字符串必须始终包含 UTF-8 编码或 7 位 ASCII 文本              | string   | String     | unicode (Python 2) or str (Python 3) | *string  |
| bytes       | 可以包含任意字节序列                                         | string   | ByteString | bytes                                | []byte   |







## 使用

### 编写规则

![image-20200910100941351](/Users/zonst/Library/Application Support/typora-user-images/image-20200910100941351.png)



### .proto 生存 .go文件命令

`protoc --go_out=. texaspokersrv.proto    `







