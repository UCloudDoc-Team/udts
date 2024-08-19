# SQL Server

UDTS 支持 SQL Server 2008及以后版本之间的互相迁移，支持 SQL Server 2008及以后版本到 MySQL 5.5及以后各版本的全量与全+增迁移任务。

## 功能限制
1. 支持单库迁移，可迁移整库或指定表，不支持迁移存储过程、触发器、视图等。
2. 全+增迁移不支持 DDL。
3. 全+增迁移时，源库和待迁移的表需要开启cdc功能。
```sql
-- 指定库名
use dbname
-- 开启cdc功能
exec sys.sp_cdc_enable_db
-- 查看 test 库是否开启cdc，返回值为 1 表示已开启
select is_cdc_enabled from sys.databases where name = "dbname";
-- 给 dbo.tablename 表开启cdc
exec sys.sp_cdc_enable_table @source_schema = 'dbo',@source_name = 'tablename',@role_name = null;
-- 查看 dbo.tablename 表是否开启cdc，返回值为 1 表示已开启
select is_tracked_by_cdc from sys.tables where name = 'tablename'
```

4. 待迁移的表必须存在主键或唯一索引，建议采用 int 类型作为主键。主键为 string 类型且数据量较大时，迁移速度会受到影响。
5. 待迁移的表中主键的自增属性无法迁移
6. 待迁移的表中 rowversion 字段无法迁移
7. 目标库为 MySQL 时，不迁移源库的 schema 名，如果源库中存在不同 schema 中的同名表，只会迁移其中一张表。


## 填写表单

数据源表单

| 参数名   | 说明                                                                                                                            |
| -------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 地址     | 支持内网地址，专线地址两种方式。内网地址需要填写VPC和子网信息；专线地址既支持IP，也支持域名，如果使用域名需要用户网络有外网出口 |
| 端口     | SQLServer 连接端口                                                                                                              |
| 用户名   | SQLServer 连接用户名                                                                                                            |
| 密码     | SQLServer 数据库对应用户密码                                                                                                    |
| 数据库名 | SQLServer 数据库名，仅支持单库迁移                                                                                              |
| 表名     | SQLServer 传输表名，表名之间使用英文逗号隔开。示例：`dbo.tablename1,dbo.tablename2`，schema 为 dbo 时可以省略，只填写 tablename，示例：`tablename1,tablename2`                             |


传输目标表单

目标库为 SQLServer 时

| 参数名   | 说明                     |
| -------- | ------------------------ |
| 地址类型 | 目标暂时只支持内网       |
| 端口     | SQLServer 连接端口           |
| 用户名   | SQLServer 连接用户名         |
| 密码     | SQLServer 数据库对应用户密码 |


目标库为 MySQL 时

| 参数名   | 说明                     |
| -------- | ------------------------ |
| 地址类型 | 目标暂时只支持内网       |
| 端口     | MySQL 连接端口           |
| 用户名   | MySQL 连接用户名         |
| 密码     | MySQL 数据库对应用户密码 |
