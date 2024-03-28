# SQLServer

UDTS 支持从 SQLServer 2008及以后版本到 MySQL 5.5及以后各版本的全量与全+增迁移任务。

## 功能限制
1. 仅支持单库单表迁移
2. 增量迁移不支持DDL
3. 全量与全+增迁移，源库均需要开启cdc功能。
```sql
-- 指定库名
use dbname
-- 开启cdc功能
exec sys.sp_cdc_enable_db
-- 查看 test 库是否开启cdc，返回值为 1 表示已开启
select is_cdc_enabled from sys.databases where name = "dbname";
```
4. 全+增迁移时，待迁移的表需要开启cdc功能。
```sql
-- 指定库名
use dbname
-- 给 dbo.tablename 表开启cdc
exec sys.sp_cdc_enable_table @source_schema = 'dbo',@source_name = 'tablename',@role_name = null;
-- 查看 dbo.tablename 表是否开启cdc，返回值为 1 表示已开启
select is_tracked_by_cdc from sys.tables where name = 'tablename'
```
5. 由于 SQLServer 与 MySQL 支持的数据类型有一定差异，部分数据类型需要转换，部分数据类型不支持迁移，具体情况如下：
| SQLServer 数据类型 | MySQL 数据类型      |
| ------------------ | ------------------- |
| bigint(i)          | bigint(i)           |
| smallint(i)        | smallint(i)         |
| int(i)             | int(i)              |
| tinyint(i)         | tinyint(i) unsigned |
| numeric            | decimal(i,j)        |
| decimal(i,j)       | decimal(i,j)        |
| smallmoney         | decimal(10,4)       |
| money              | decimal(19,4)       |
| bit                | boolean             |
| float              | double              |
| real               | double              |
| date               | date                |
| time(n)            | time(n)             |
| smalldatetime      | datetime            |
| datetime           | datetime            |
| datetimeoffset(n)  | datetime(n)         |
| datetime2(n)       | datetime(n)         |
| text               | text                |
| ntext              | text                |
| xml                | text                |
| char(n)            | char(n)             |
| nchar(n)           | char(n)             |
| varchar(n)         | varchar(n)          |
| nvarchar(n)        | varchar(n)          |
| binary(n)          | binary(n)           |
| varbinary(n)       | varbinary(n)        |
| image              | blob                |
| 其余数据类型       | 不支持迁移          |
6. 待迁移的表必须存在主键或唯一索引
7. 不支持迁移外键和 check 约束

## 填写表单

数据源表单

| 参数名   | 说明                                                                                                                            |
| -------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 地址     | 支持内网地址，专线地址两种方式。内网地址需要填写VPC和子网信息；专线地址既支持IP，也支持域名，如果使用域名需要用户网络有外网出口 |
| 端口     | SQLServer 连接端口                                                                                                              |
| 用户名   | SQLServer 连接用户名                                                                                                            |
| 密码     | SQLServer 数据库对应用户密码                                                                                                    |
| 数据库名 | SQLServer 数据库名，仅支持单库迁移                                                                                              |
| 表名     | SQLServer 传输表名，仅支持单表迁移，示例：dbo.tablename，schema 为 dbo 时可以省略，只填写 tablename                             |


传输目标表单

| 参数名   | 说明                     |
| -------- | ------------------------ |
| 地址类型 | 目标暂时只支持内网       |
| 端口     | MySQL 连接端口           |
| 用户名   | MySQL 连接用户名         |
| 密码     | MySQL 数据库对应用户密码 |