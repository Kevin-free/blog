# MySQL-ClickHouse 同步

## 1 问题背景

日志记录存储在数据源 MySQL 中，需要同步到 ClickHouse 中。为后续需求提供高效的查询操作。



## 2 解决方案

### 2.1 MySQL 表引擎

MySQL 引擎可以对存储在远程 MySQL 服务器上的数据执行 `SELECT` 查询。

调用格式：

```
MySQL('host:port', 'database', 'table', 'user', 'password'[, replace_query, 'on_duplicate_clause']);
```

**调用参数**

- `host:port` — MySQL 服务器地址。
- `database` — 数据库的名称。
- `table` — 表名称。
- `user` — 数据库用户。
- `password` — 用户密码。
- `replace_query` — 将 `INSERT INTO` 查询是否替换为 `REPLACE INTO` 的标志。如果 `replace_query=1`，则替换查询
- `'on_duplicate_clause'` — 将 `ON DUPLICATE KEY UPDATE 'on_duplicate_clause'` 表达式添加到 `INSERT` 查询语句中。例如：`impression = VALUES(impression) + impression`。如果需要指定 `'on_duplicate_clause'`，则需要设置 `replace_query=0`。如果同时设置 `replace_query = 1` 和 `'on_duplicate_clause'`，则会抛出异常。

此时，简单的 `WHERE` 子句（例如 `=, !=, >, >=, <, <=`）是在 MySQL 服务器上执行。

其余条件以及 `LIMIT` 采样约束语句仅在对MySQL的查询完成后才在ClickHouse中执行。

`MySQL` 引擎不支持 [可为空](https://clickhouse.com/docs/zh/engines/table-engines/integrations/mysql/) 数据类型，因此，当从MySQL表中读取数据时，`NULL` 将转换为指定列类型的默认值（通常为0或空字符串）。

#### 小结

MySQL Table Engine 的特性。

- Mapping to MySQL table
- Fetch table struct from MySQL
- Fetch data from MySQL when executing query

ClickHouse 最开始支持表级别同步 MySQL 数据，通过外部表引擎 MySQL Table Engine 来实现同 MySQL 表的映射。从 `information_schema` 表中获取对应表的结构，将其转换为 ClickHouse 支持的数据结构，此时在 ClickHouse 端，表结构建立成功。但是此时，并没有真正去同步数据。只有向 ClickHouse 中的该表发起请求时，才会主动的拉取要同步的 MySQL 表的数据。

MySQL Table Engine 使用起来非常简陋，但它是非常有意义的。因为这是第一次打通 ClickHouse 和 MySQL 的数据通道。但是，缺点异常明显：

i. 仅仅是对 MySQL 表关系的映射；

ii. 查询时传输 MySQL 数据到 ClickHouse，会给 MySQL 可能造成未知的网络压力和读压力，可能影响 MySQL 在生产中正常使用。

基于 MySQL Table Engine 只能映射 MySQL 表关系的缺点，QingCloud ClickHouse 团队实现了 MySQL Database Engine。

#### 实践

##### MySQL（此部分作为公用）

**新建数据库 `m_demo` 和表 `t_student` ：**

```sh
mysql> create database m_demo;

mysql> CREATE TABLE IF NOT EXISTS `t_student`(
    ->    `id` INT UNSIGNED AUTO_INCREMENT,
    ->    `name` VARCHAR(100) NOT NULL,
    ->    `age` INT(3) NOT NULL,
    ->    `create_date` DATE,
    ->    PRIMARY KEY ( `id` )
    -> )ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

插入数据：

```sh
mysql> INSERT INTO `m_demo`.`t_student`(`name`, `age`, `create_date`) VALUES ('Kevin', 18, NOW());
mysql> INSERT INTO `m_demo`.`t_student`(`name`, `age`, `create_date`) VALUES ('Tom', 16, NOW());
mysql> INSERT INTO `m_demo`.`t_student`(`name`, `age`, `create_date`) VALUES ('Jerry', 14, NOW());
```

##### ClickHouse

**新建数据库`c_mysql_table`和表引擎`t_student_slave`：**

```sh
kaiwen :)create c_mysql_table;

kaiwen :) CREATE TABLE t_student_slave(`id` UInt32,`name` String,`age` UInt32,`create_date` Nullable(Date32))ENGINE = MySQL('10.4.87.87:3306', 'm_demo', 't_student', 'root', 'tars2015');
```

查询数据：

```sh
kaiwen :) select * from t_student_slave;

SELECT *
FROM t_student_slave

Query id: 052fe5d2-d626-4c94-9015-a5a991418ab4

┌─id─┬─name──┬─age─┬─create_date─┐
│  1 │ Kevin │  18 │  2022-04-20 │
│  2 │ Tom   │  16 │  2022-04-20 │
│  3 │ Jerry │  14 │  2022-04-20 │
└────┴───────┴─────┴─────────────┘

3 rows in set. Elapsed: 0.010 sec. 
```

#### 分析

MySQL外表引擎，本身不存储数据，数据存储在MySQL中。在复制查询中，特别是有JOIN的情况下，访问外表是相当慢的，甚至不可能完成。

##### 优点：

- 可以只同步某张表
- 可以只同步某些字段

##### 缺点：

- 无法增量导入数据
- 效率低

#### 参考

[MySQL 表引擎](https://clickhouse.com/docs/zh/engines/table-engines/integrations/mysql/)

[ClickHouse 导入数据实战：MySQL篇](https://cloud.tencent.com/developer/article/1602662)

[HTAP | MySQL 到 ClickHouse 的高速公路](https://segmentfault.com/a/1190000040147564)



### 2.2 MySQL 库引擎

MySQL引擎用于将远程的MySQL服务器中的表映射到ClickHouse中，并允许您对表进行`INSERT`和`SELECT`查询，以方便您在ClickHouse与MySQL之间进行数据交换

`MySQL`数据库引擎会将对其的查询转换为MySQL语法并发送到MySQL服务器中，因此您可以执行诸如`SHOW TABLES`或`SHOW CREATE TABLE`之类的操作。

但您无法对其执行以下操作：

- `RENAME`
- `CREATE TABLE`
- `ALTER`

#### 创建数据库

```text
CREATE DATABASE [IF NOT EXISTS] db_name [ON CLUSTER cluster]
ENGINE = MySQL('host:port', ['database' | database], 'user', 'password')
```

**引擎参数**

- `host:port` — MySQL服务地址
- `database` — MySQL数据库名称
- `user` — MySQL用户名
- `password` — MySQL用户密码

#### 小结

MySQL Database Engine 的特性。

- Mapping to MySQL Database
- Fetch table list from MySQL
- Fetch table struct from MySQL
- Fetch data from MySQL when executing query

MySQL Database Engine 是库级别的映射，要从 `information_schema` 中拉取待同步库中包含的所有 MySQL 表的结构，解决了需要建立多表的问题。但仍然还有和 MySQL Table Engine 一样的缺点：查询时传输 MySQL 数据到 ClickHouse，给 MySQL 可能造成未知的网络压力和读压力，可能影响 MySQL 在生产中正常使用。

#### 实践

##### MySQL

使用公用库 `m_demo` 和表 `t_student` 

##### ClickHouse

**创建 MySQL 同步库**：

```sh
kaiwen :) CREATE DATABASE c_mysql_db ENGINE = MySQL('10.4.87.87:3306', 'm_demo', 'root', 'tars2015');

CREATE DATABASE c_mysql_db
ENGINE = MySQL('10.4.87.87:3306', 'm_demo', 'root', 'tars2015')

Query id: b1cdcdbc-0126-4961-ac0d-d6bf6e309666

Ok.

0 rows in set. Elapsed: 0.513 sec. 
```

查询验证：

```sh
kaiwen :) select * from t_student;

SELECT *
FROM t_student

Query id: 7a671744-46c1-47c4-8eee-28e951ad46bf

┌─id─┬─name──┬─age─┬─create_date─┐
│  1 │ Kevin │  18 │  2022-04-20 │
│  2 │ Tom   │  16 │  2022-04-20 │
│  3 │ Jerry │  14 │  2022-04-20 │
└────┴───────┴─────┴─────────────┘

3 rows in set. Elapsed: 0.008 sec.
```

#### 分析

MySQL引擎将远程的MySQL服务器中的表映射到ClickHouse中，MySQL数据库引擎会将对其的查询转换为MySQL语法并发送到MySQL服务器中。

##### 优点：

- 支持全量和增量同步

##### 缺点：

- 转换为MySQL语法并发送到MySQL服务器中。这跟直接用mysql有什么区别？

#### 参考

[MySQL 数据库引擎](https://clickhouse.com/docs/zh/engines/database-engines/mysql/#creating-a-database)



### 2.3 MaterializedMySQL 库引擎

> 提示：ClickHouse 从20.8.\*才开始支持该引擎
>
> 另外：你可能看网上会发现有些写的是没有 d 的 MaterializeMySQL，据了解是 CK 版本不一致导致的。参见 [ISSUE](https://github.com/ClickHouse/ClickHouse/issues/31176)
>
> 实测 22.3.3 版本使用 MaterializedMySQL 都可以的。

这个引擎创建ClickHouse数据库，包含MySQL中所有的表，以及这些表中的所有数据。ClickHouse服务器作为MySQL副本工作。它读取binlog并执行DDL和DML查询。

> 注意：需要先开启 materialized 同步功能，否则会创建失败！
>
> ```sql
> # 开启materialized同步功能
> SET allow_experimental_database_materialized_mysql=1;
> ```

#### 创建数据库

```text
CREATE DATABASE [IF NOT EXISTS] db_name [ON CLUSTER cluster]
ENGINE = MaterializedMySQL('host:port', ['database' | database], 'user', 'password') [SETTINGS ...]
[TABLE OVERRIDE table1 (...), TABLE OVERRIDE table2 (...)]
```

**引擎参数**

- `host:port` — MySQL服务地址
- `database` — MySQL数据库名称
- `user` — MySQL用户名
- `password` — MySQL用户密码

**引擎配置**

- `max_rows_in_buffer` — 允许数据缓存到内存中的最大行数(对于单个表和无法查询的缓存数据)。当超过行数时，数据将被物化。默认值: `65505`。
- `max_bytes_in_buffer` — 允许在内存中缓存数据的最大字节数(对于单个表和无法查询的缓存数据)。当超过行数时，数据将被物化。默认值: `1048576`.
- `max_rows_in_buffers` — 允许数据缓存到内存中的最大行数(对于数据库和无法查询的缓存数据)。当超过行数时，数据将被物化。默认值: `65505`.
- `max_bytes_in_buffers` — 允许在内存中缓存数据的最大字节数(对于数据库和无法查询的缓存数据)。当超过行数时，数据将被物化。默认值: `1048576`.
- `max_flush_data_time` — 允许数据在内存中缓存的最大毫秒数(对于数据库和无法查询的缓存数据)。当超过这个时间时，数据将被物化。默认值: `1000`.
- `max_wait_time_when_mysql_unavailable` — 当MySQL不可用时重试间隔(毫秒)。负值禁止重试。默认值: `1000`.
- `allows_query_when_mysql_lost` — 当mysql丢失时，允许查询物化表。默认值: `0` (`false`).

eg：

```text
CREATE DATABASE mysql ENGINE = MaterializeMySQL('localhost:3306', 'db', 'user', '***') 
     SETTINGS 
        allows_query_when_mysql_lost=true,
        max_wait_time_when_mysql_unavailable=10000;
```

#### MySQL服务器端配置

为了`MaterializedMySQL`的正确工作，有一些必须设置的`MySQL`端配置设置:

- `default_authentication_plugin = mysql_native_password`，因为 `MaterializedMySQL` 只能授权使用该方法。
- `gtid_mode = on`，因为基于GTID的日志记录是提供正确的 `MaterializedMySQL`复制的强制要求。

#### 小结

MaterializeMySQL 的设计思路如下：

1. 首先检验源端 MySQL 参数是否符合规范;
2. 再将数据根据 GTID 分割为历史数据和增量数据;
3. 同步历史数据至 GTID 点;
4. 持续消费增量数据。

![3-1](https://segmentfault.com/img/bVcR6Uy)

如图 3-1 所示，MaterializeMySQL 函数的主体流程为：

CheckMySQLVars -> prepareSynchronized -> Synchronized。

MaterializedMySQL 库引擎的特性：

1.此引擎大大方便了mysql导入数据到clickhouse，但是官方提示还在实验中，不要用在生产环境

2.查询时要带上虚拟列_version ,否则会默认使用final，效率很低

3.使用集群会有很多的局限

MySQL 主从间数据同步时Slave节点将 BinLog Event 转换成相应的SQL语句，Slave 模拟 Master 写入。类似地，传统第三方插件沿用了MySQL主从模式的BinLog消费方案，即将 Event 解析后转换成 ClickHouse 兼容的 SQL 语句，然后在 ClickHouse 上执行（回放），但整个执行链路较长，通常性能损耗较大。不同的是，MaterializeMySQL 引擎提供的内部数据解析以及回写方案隐去了三方插件的复杂链路。回放时将 BinLog Event 转换成底层 Block 结构，然后直接写入底层存储引擎，接近于物理复制。此方案可以类比于将 BinLog Event 直接回放到 InnoDB 的 Page 中。

#### 实践

##### MySQL

使用公用库 `m_demo` 和表 `t_student` 

另外需配置 MySQL 支持同步！

```sh
# vim /etc/my.cnf  //添加如下配置 
  
log-bin=mysql-bin
server-id=1

gtid_mode=ON
enforce_gtid_consistency=1
binlog_format=ROW 

# service mysqld restart //改完重启  
  
mysql> show global variables like '%gtid%';  //查看配置  
select @@binlog_format;   //查看配置  
```

##### ClickHouse

**开启 materialized 同步功能**：

```sh
# 开启materialized同步功能
SET allow_experimental_database_materialized_mysql=1;

# 检查
kaiwen :) select value from system.settings where name = 'allow_experimental_database_materialized_mysql';

SELECT value
FROM system.settings
WHERE name = 'allow_experimental_database_materialized_mysql'

Query id: acf9a6dd-dc38-4b3e-b441-b2bf5fd39aab

┌─value─┐
│ 1     │
└───────┘

1 rows in set. Elapsed: 0.002 sec. 
```

> 如果未开启materialized同步功能，则会报错：

```sh
Received exception from server (version 22.3.3):
Code: 336. DB::Exception: Received from localhost:9000. DB::Exception: MaterializedMySQL is an experimental database engine. Enable allow_experimental_database_materialized_mysql to use it.. (UNKNOWN_DATABASE_ENGINE)
```

**创建 MaterializeMySQL 同步库**：

```sh
# MaterializedMySQL 参数分别是("mysqld服务地址", "待同步库名", "授权账户", "密码")
kaiwen :) CREATE DATABASE c_mt_mysql ENGINE = MaterializedMySQL('10.4.87.87:3306', 'm_demo', 'root', 'tars2015');

CREATE DATABASE c_mt_mysql
ENGINE = MaterializedMySQL('10.4.87.87:3306', 'm_demo', 'root', 'tars2015')

Query id: f3272508-a5ad-4c3e-bb72-d0b6287d89ed

Ok.

0 rows in set. Elapsed: 0.009 sec. 
```

此时可以看到 ClickHouse 中已经有从MySQL中同步的数据了：

```sh
kaiwen :) show tables;

SHOW TABLES

Query id: 3598b97a-1ca6-4c4b-8bb1-414f23840abd

┌─name──────┐
│ t_student │
└───────────┘

1 rows in set. Elapsed: 0.002 sec. 

kaiwen :) select * from t_student;

SELECT *
FROM t_student

Query id: 03dcb6aa-50bc-40be-8171-7033f038b23a

┌─id─┬─name──┬─age─┬─create_date─┐
│  1 │ Kevin │  18 │  2022-04-20 │
└────┴───────┴─────┴─────────────┘
┌─id─┬─name─┬─age─┬─create_date─┐
│  2 │ Tom  │  16 │  2022-04-20 │
└────┴──────┴─────┴─────────────┘
┌─id─┬─name─────┬─age─┬─create_date─┐
│  3 │ Jerry │  14 │  2022-04-20 │
└────┴──────────┴─────┴─────────────┘

3 rows in set. Elapsed: 0.004 sec. 
```

#### 分析

这个引擎创建ClickHouse数据库，包含MySQL中所有的表，以及这些表中的所有数据。ClickHouse服务器作为MySQL副本工作。它读取binlog并执行DDL和DML查询。

ClickHouse 内部实现一套 binlog 消费方案，然后根据 event 解析成 ClickHouse 内部的 block 结构，再直接回写到底层存储引擎，几乎是最高效的一种实现方式，实现与 MySQL 实时同步的能力，让分析更接近现实。

MySQL 主从间数据同步时Slave节点将 BinLog Event 转换成相应的SQL语句，Slave 模拟 Master 写入。类似地，传统第三方插件沿用了MySQL主从模式的BinLog消费方案，即将 Event 解析后转换成 ClickHouse 兼容的 SQL 语句，然后在 ClickHouse 上执行（回放），但整个执行链路较长，通常性能损耗较大。

不同的是，MaterializeMySQL 引擎提供的内部数据解析以及回写方案隐去了三方插件的复杂链路。回放时将 BinLog Event 转换成底层 Block 结构，然后直接写入底层存储引擎，接近于物理复制。此方案可以类比于将 BinLog Event 直接回放到 InnoDB 的 Page 中。

##### 优点：

- 支持全量和增量同步
- ClickHouse 官方实现，兼容高效

##### 缺点：

- 此引擎大大方便了mysql导入数据到clickhouse，但是官方提示还在实验中，不要用在生产环境
- 查询时要带上虚拟列_version ,否则会默认使用final，效率很低
- 使用集群会有很多的局限
- 支持mysql 库级别的数据同步，暂不支持表级别的
- MySQL中的每个表都应该包含PRIMARY KEY

#### 参考

[Roadmap 2021](https://github.com/ClickHouse/ClickHouse/issues/17623)

[MaterializedMySQL 引擎](https://clickhouse.com/docs/zh/engines/database-engines/materialized-mysql/)

[ClickHouse和他的朋友们（9）MySQL实时复制与实现](https://bohutang.me/2020/07/26/clickhouse-and-friends-mysql-replication/)

[MySQL到ClickHouse的高速公路-MaterializeMySQL引擎](https://bbs.huaweicloud.com/forum/thread-102438-1-1.html)



### 2.4 结合第三方软件

![img](https://segmentfault.com/img/bVcR6Uv)

还有可以采用第三方软件来同步数据，比如 Canal 或者 Kafka，通过解析 MySQL binlog，然后编写程序控制向 ClickHouse 写入。这样做有很大的优势，即同步流程自主可控。但是也带来了额外的问题：

i. 增加了数据同步的复杂度。

ii. 增加了第三方软件，使得运维难度指数级增加。

#### 分析

##### 优点：

- 定制化实现同步

##### 缺点：

- 开发成本，兼容效率

#### 参考

[腾讯大牛教你ClickHouse实时同步MySQL数据](https://cloud.tencent.com/developer/article/1737840?from=article.detail.1627940)

[clickhouse-mysql数据同步](https://zhuanlan.zhihu.com/p/462712274)



### 2.5 第三方产品

#### clickhouse-mysql-data-reader

Altinity 公司开源的一个 python 工具，用来从 mysql 迁移数据到 clickhouse(支持 binlog 增量更新和全量导入)，但是官方 readme 和代码脱节，根据 quick start 跑不通。

#### streamsets

streamsets 支持从 mysql 或者读 csv 全量导入，也支持订阅 binlog 增量插入，参考 [025-大数据 ETL 工具之 StreamSets 安装及订阅 mysql binlog](https://anjia0532.github.io/2019/06/10/cdh-streamsets/)。

#### Tapdata Cloud

[Tapdata Cloud](https://tapdata.net/tapdata-cloud.html)

#### mysql-clickhouse-replication

[mysql-clickhouse-replication](https://github.com/yymysql/mysql-clickhouse-replication)

#### Bifrost

[Bifrost](https://github.com/brokercap/Bifrost)

#### 分析

##### 优点：

- 开箱即用

##### 缺点：

- 数据安全性
- 定制化，性价比



### 2.6 总结

结合以上，目前考虑两种方案：2.3 MaterializedMySQL 库引擎 和 2.4 结合第三方软件

| 方案                     | 优点                                                        | 缺点                                                         |
| ------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------ |
| MaterializedMySQL 库引擎 | - 支持全量和增量同步  <br />- ClickHouse 官方实现，兼容高效 | - 此引擎大大方便了mysql导入数据到clickhouse，但是官方提示还在实验中，不要用在生产环境<br/>- 查询时要带上虚拟列_version ,否则会默认使用final，效率很低<br/>- 使用集群会有很多的局限<br/>- 支持mysql 库级别的数据同步，暂不支持表级别的<br/>- MySQL中的每个表都应该包含PRIMARY KEY |
| 结合第三方软件           | - 定制化实现同步                                            | - 开发成本，兼容效率                                         |



## 3. 我选择的方案

Q：

1. 是否可以全库全表同步？
2. 是否需要自主控制同步流程？











