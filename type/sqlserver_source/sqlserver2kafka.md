# SQL Server 迁移到 Kafka
UDTS 支持 从 SQL Server 迁移到 Kafka。
 SQL Server 支持版本为 SQL Server 2008 R2及以上。
 Kafka 支持版本 2.x：包括 2.0、2.1、2.2、2.3、2.4、2.5、2.6、2.7、2.8 等。

## 1. 功能限制

### 1.1 源SQL Server限制
1. 源库和待迁移的表需要开启cdc功能。

```sql
-- 指定库名
use dbname
-- 开启cdc功能
exec sys.sp_cdc_enable_db
-- 查看 dbname 库是否开启cdc，返回值为 1 表示已开启
select is_cdc_enabled from sys.databases where name = "dbname";
-- 给 dbo.tablename 表开启cdc
exec sys.sp_cdc_enable_table @source_schema = 'dbo',@source_name = 'tablename',@role_name = null;
-- 查看 dbo.tablename 表是否开启cdc，返回值为 1 表示已开启
select is_tracked_by_cdc from sys.tables where name = 'tablename'
-- 修改 cdc 数据保留时间，至少修改为1440分钟（1天）或以上，建议14400分钟（7天）。
EXECUTE sys.sp_cdc_change_job @job_type = N'cleanup', @retention = 14400;
```

### 1.2 目标Kafka限制

1. 设置 `auto.create.topics.enable` 为true。 
2. 设置`delete.topic.enable` 为true


## 2. 迁移内容

| 迁移内容 | 说明                                                                |
| -------- | ------------------------------------------------------------------- |
| 迁移结构 | Database、Table 结构及数据                                          |
| 迁移范围 | 仅迁移创建任务时可以查到的库表， 任务运行中新增的表暂时不会自动迁移 |
| DDL      | CREATE、ALTER、DROP 等语句                                          |
| DML      | Snapshot/insert/update/delete                                       |


## 3. 表单填写

### 数据源表单

| 参数名   | 说明                                                                                                                            |
| -------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 地址     | 支持内网地址，专线地址两种方式。内网地址需要填写VPC和子网信息；专线地址既支持IP，也支持域名，如果使用域名需要用户网络有外网出口 |
| 端口     | SQL Server 连接端口                                                                                                              |
| 用户名   | SQL Server 连接用户名                                                                                                            |
| 密码     | SQL Server 数据库对应用户密码                                                                                                    |
| 数据库名 | SQL Server 数据库名，仅支持单库迁移                                                                                              |
| 表名     | SQL Server 传输表名，表名之间使用英文逗号隔开。示例：`dbo.tablename1,dbo.tablename2`，schema 为 dbo 时可以省略，只填写 tablename，示例：`tablename1,tablename2`                             |


###  传输目标表单

| 参数名       | 说明                                                                                                          |
| ------------ | ------------------------------------------------------------------------------------------------------------- |
| 内网地址     | Kafka集群的连接地址，示例：92.168.1.10:9092,192.168.1.11:9092,192.168.1.12:9093                               |
| 最大速率     | Kafka传输限速值，调整传输速率                                                                                 |
| Topic前缀    | UDTS迁移MySQL到Kafka会在目标Kafka上创建对应的Topic， 规则是每张表创建一个Topic，每个Topic都会以此参数作为前缀(格式为: Topic前缀_数据库名_表名) |
| 默认分区数量 | UDTS迁移MySQL到Kafka会在目标Kafka上创建对应Topic的默认分区数量                                                |


## 4. Kafka传输数据格式

UDTS 迁移到Kafka的内容，格式为 Debezium JSON 格式, 下游同步可使用Flink CDC ，数据格式指定为 debezium-json。

UDTS 会为每一张表创建一个Topic，没张表的数据会写入到其对应的Topic中，Topic的名称为: Topic前缀_数据库名_表名。