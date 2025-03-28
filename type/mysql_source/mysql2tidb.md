
# MySQL 迁移到 TiDB

UDTS支持MySQL作为数据传输源传输到TiDB，支持版本有 MySQL(包含Percona版)5.5/5.6/5.7/8.0; MariaDB 10.1.2 及以上，以及PolarDB(MySQL兼容版本)。


## 迁移内容

全量和增量迁移以下内容
- Database、Table 结构及数据

在全量迁移时，为了保证数据的一致性，防止数据冲突，会在迁移前清理目标库中的数据，清理的内容为本次迁移对应的 Database 和 Table，具体如下

| `数据库名`设置 | `表名`设置           | 清理内容                                                |
|----------------|----------------------|-----------------------------------------------------|
| *              |                      | 清理源库中除内置库之外的数据库(参考备注说明 1)          |
| db1,db2,db3    |                      | 清理任务指定的多个 DB(参考备注说明 2)                   |
| db1            |                      | 清理任务指定的单个 DB                                   |
| db1            | table1,table2,table3 | 清理任务指定的单个DB 下指定的多个 Table(参考备注说明 3) |
| db1            | table1               | 清理任务指定的单个DB 下指定的单个 Table                 |


备注说明：

- 1、 假设在源库上执行 SHOW DATABASES 的内容如下

```
> SHOW databases;

INFORMATION_SCHEMA
PERFORMANCE_SCHEMA
mysql
sys
mydb1
mydb2
```

当前的内置库为 `mysql`、`sys`、`INFORMATION_SCHEMA`、`PERFORMANCE_SCHEMA`，排除这些 DB 后，剩余 `mydb1`和`mydb2`，则在迁移任务运行时，会在目标数据库中执行

```sql
DROP DATABASE IF EXISTS `mydb1`;
DROP DATABASE IF EXISTS `mydb2`;
```

- 2、 迁移任务配置的 `数据库名`为 db1,db2,db3，则在目标数据库中会执行


```sql
DROP DATABASE IF EXISTS `db1`;
DROP DATABASE IF EXISTS `db2`;
DROP DATABASE IF EXISTS `db3`;
```

- 3、 迁移任务配置的 `数据库名`为 db1, 迁移的 `表名` 为 table1,table2,table3，则在目标数据库中会执行

```sql
DROP TABLE IF EXISTS `db1`.`table1`;
DROP TABLE IF EXISTS `db1`.`table2`;
DROP TABLE IF EXISTS `db1`.`table3`;
```

## 前提条件

UDTS 在迁移时，可以通过`预检查`完成对需要条件的检查，包括以下内容

- 连接性检查，包括主机地址、端口、用户名和密码等信息的正确性。
- 权限检查，检查迁移时需要的权限。
- 配置检查，如 sql_mode、 binlog 格式、唯一键检查等。

### 连接性检查
连接性检查会对填写的 `主机地址`,`端口`，`用户名`，`密码`进行检查

- 主机连通性检查：如果填写了错误的`主机地址`或者`端口`，会出现连通性检查失败： `Project OR IP:10.19.37.212 OR Vpc:uvnet-u0ecvp, Subnet:subnet-2slodr error, please check again`, 
请确认当前的 `主机地址`，`端口`，选择的 `VPC ID`,`所属子网`，以及`项目`是否正确。

- 用户名密码检查：如果填写了错误的用户名或者密码，会出现连通性检查失败: `Error 1045: Access denied for user 'root1'@'10.42.255.213' (using password: YES)`, 请确认用户名和密码正确。

- 防火墙和白名单：如果主机地址，端口，用户名和密码均正确
  * 如果 MySQL 为主机自建，需要排查主机的 iptables 规则
  * 如果设置了白名单，请联系技术支持获取白名单地址

### 权限检查

默认配置迁移需要源数据库账号拥有 SUPER 权限， 如果没有 SUPER 权限，请选择 "NoLocks" 模式。  
如果用户需要在全量阶段开启 NoBinlog 模式，那么在目标库还需要有 SUPER 权限。

| 类型/权限 | 源库权限(开启 NoLocks)                                                          | 源库权限(未开启 NoLocks)                                                                                 | 目标库权限                                                                                                                   |
|---------|-------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| 全量      | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW" , "PROCESS" | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW", "RELOAD", "LOCK TABLES" , "PROCESS" | "SELECT", "INSERT", "UPDATE", "CREATE", "DROP", "ALTER", "DELETE", "INDEX", "CREATE VIEW"              |
| 增量      | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW"             | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW"                                      | "SELECT", "INSERT", "UPDATE", "CREATE", "DROP", "ALTER", "DELETE", "INDEX", "CREATE VIEW" |
| 全+增     | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW" , "PROCESS" | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW", "RELOAD", "LOCK TABLES" , "PROCESS" | "SELECT", "INSERT", "UPDATE", "CREATE", "DROP", "ALTER", "DELETE", "INDEX", "CREATE VIEW"|

### sql_mode 检查

为了保证迁移能正确执行，最好保持 源和目标数据库的 sql_mode 一致，可以通过以下命令查询 sql_mode

```
select @@sql_mode;

--- 或者
show variables like "sql_mode";
```

如果目标数据库和源数据的 sql_mode 不一致，可以通过下面的命令进行修改

```
SET GLOBAL sql_mode='xxxx';
```

其中具体的值，可以通过连接源数据库查询。

### binlog 格式检查

如果迁移的任务类型为 `增量`、`全量+增量`, 或者全量任务后期需要进行增量迁移，需要源库开启 binlog，且格式设置为ROW, image设置为FULL

```
binlog_format    为 ROW
binlog_row_image 为 FULL
```

查询方式：
```
show global variables like 'binlog_format';
show global variables like 'binlog_row_image';
```

设置方式：
```
set global binlog_format = "ROW" ;
set global binlog_row_image = "FULL" ;
```

备注： 如果是 MySQL 5.5 ，没有 binlog_row_image 这个变量，不需要设置

### 从MySQL迁移到TiDB时，检查源库字符集

TiDB目前支持的字符集包括`ascii/latin1/binary/utf8/utf8mb4`。
从MySQL迁移到TiDB时，如果源库中需要迁移的表或表中某一字段采用的字符集不包含在上述字符集之中，则无法迁移。

查询方式：
```
show create table table1;
```

设置方式：
```
# 将表 table1 的字符集修改为 utf8
alter table task character set utf8;

# 将表 table1 中 column1 字段的字符集修改为 utf8
alter table table1 change column1 column1 varchar(200) character set utf8;
```

## 功能限制
### MyISAM 引擎表

UDTS 支持 MyISAM 引擎表的全量迁移及增量同步，但是有以下限制：
* 导出数据时MyISAM表允许读，不允许写，并且写SQL会阻塞之后的读SQL直到表转储完毕，可能会影响业务。
* 不支持一条事务中同时更新MyISAM引擎表和InnoDB引擎表。
* 对于损坏的表，在迁移之前要进行修复。

建议用户将MyISAM 引擎表转换为InnoDB引擎表以后再使用UDTS迁移。

### 数据量

建议单个全量迁移任务所迁移的数据量不超过200G， 最大支持500G。 如果所要迁移的数据量超过了500G，可以将任务拆分为多个任务进行迁移。 UDTS 提供按库、按表、按多库、按多表等多维度的迁移方式。 

如果迁移任务超过了500G， 且无法拆分为多个任务， 请联系技术支持。

### 其它限制
- 源库不能存在同名但大小写不一致的库或表，否则同步可能会异常， 建议全部采用小写格式。
- 不迁移除 test 以外的内置数据库。
- 在转储过程中，如果待迁移的数据库上有执行 DDL 语句，UDTS 任务会失败，用户可以选择一个不会执行 DDL 语句的时间段重启任务。
- 全量和增量阶段不支持 event, trigger, procedure, function。
- 增量同步开启之前，需要关闭 event.
        ```
        -- 停止所有 event

        SET GLOBAL event_scheduler = OFF;

        -- 如果按库迁移，停止指定库的 event 即可
        -- 查找对应的 event 并停止
        SHOW EVENTS;

        ALTER EVENT hello DISABLE;
        ```
- 源库不能有超过4G的binlog文件， 建议大小是1G, 否则同步会出错或失败。
- 不支持 MySQL 8.0 的新特性 binlog 事务压缩 Transaction_payload_event。使用 binlog 事务压缩有导致上下游数据不一致的风险。
- 如果源是阿里云 MySQL RDS, 所有表必须要有显式主键， 否则增量同步功能不支持。
- 数据库名中包含特殊字符'$'时，只支持单库迁移。
- 表名或者库名不能包含特殊字符'.'， 否则迁移失败。
- 增量迁移时，DDL 语句仅支持字符集`ascii/latin1/binary/utf8/utf8mb4`。


## 注意事项
### 主从切换

如果源MySQL数据库是主从结构，当主从发生切换时，对UDTS产生的影响如下：
- 发生在全量迁移阶段，全量迁移失败，需要重启任务。
- 发生的增量同步阶段，如果使用GTID同步，则无影响；如果没有使用GTID同步， 则同步失败， 需要重新做全+增。


## SSL 安全连接
为了提高链路的安全性，支持使用 SSL 证书连接数据库。SSL 在传输层对数据进行加密，提升通信数据的安全性，但会增加一定的网络连接响应时间。

UDTS 当前支持 pem 格式的证书，如果您使用的是其它格式的证书，可以先转换为 pem 格式，`SSL 安全连接`可以在源或者目标中设置。

### 如何使用 
在源或者目标中，将 `SSL 安全连接` 选项打开，同时上传 ca 证书
![](http://udts-doc.cn-bj.ufileos.com/create-ssl.png)


## MySQL 填写表单

数据源表单

| 参数名   | 说明                                                                                                                                                              |
|--------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 地址类型 | 支持内网地址，外网地址，专线地址三种方式。内网地址需要填写VPC和子网信息；外网地址支持IP和域名两种方式；专线地址既支持IP，也支持域名，如果使用域名需要用户网络有外网出口。 |
| 端口     | MySQL连接端口                                                                                                                                                     |
| 用户名   | MySQL连接用户名                                                                                                                                                   |
| 密码     | MySQL数据库对应用户密码                                                                                                                                           |
| 数据库名 | MySQL数据库名称。 所有库传输请填写 *； 指定一个数据库传输，请填写数据库名；指定多个数据库传输，依次输入多个数据库名，库名之间使用英文逗号隔开。(如果数据库名称中包含空格则无法做增量迁移)|                                         |
| 表名     | MySQL传输表名。 只有当“数据库名”为指定一个数据库时有效。 若不填，默认为迁移指定库中的所有表； 指定一张表传输， 请填写表名； 指定多张表传输，依次输入多张表名，表名之间使用英文逗号隔开 |
| 最大速率    | 外网/专线的速率范围为 1-56 MB/s; 内网的速率范围为 1-1024 MB/s         |
| Nolocks     | 默认关闭，对于无法获取 SUPER 权限的友商RDS服务需要开启，UDB获取 SUPER 权限详见FAQ                  |
| SSL 安全连接  | 默认关闭，当您需要使用证书连接数据库时，可以打开该选项，具体可以[参考](#ssl-安全连接)             |

传输目标表单

| 参数名       | 说明                                                                                  |
|--------------|-------------------------------------------------------------------------------------|
| 地址类型     | 目标暂时只支持内网                                                                    |
| 端口         | TiDB 连接端口                                                                         |
| 用户名       | TiDB 连接用户名                                                                       |
| 密码         | TiDB 数据库对应用户密码                                                               |
| 最大速率     | 内网的速率范围为 1-1024 MB/s                                   |
| 兼容MySQL自增模式 | 默认关闭，TiDB默认自增ID和MySQl行为不一样， 开启此选项可以在大多数情况下兼容MySQL自增模式， 具体区别请参考: https://docs.pingcap.com/zh/tidb/stable/auto-increment/#mysql-%E5%85%BC%E5%AE%B9%E6%A8%A1%E5%BC%8F |


### 建议速率配置表

源限速配置参考值

| 数据库可用内存大小 (G) | 内网建议值 (MB/s) | 外网/专线建议值 (MB/s) |
| ------------------ | ---------- | --------------- |
| (0, 2)            | 10       | 10            |
| [2, 4)           | 15       | 15            |
| [4, 6)           | 20       | 20            |
| [6, 8)           | 30       | 30            |
| [8, 10)          | 40       | 40            |
| [10, 20)         | 50       | 50            |
| [20, 40)         | 60       | 56            |
| [40, 60)         | 70       | 56            |
| [60, 80)         | 80       | 56            |
| [80, 100)        | 90       | 56            |
| >=100             | 100      | 56            |


## MySQL与TiDB兼容性说明

TiDB 高度兼容MySQL协议、MySQL 常用的功能及语法。MySQL 生态中的系统工具（PHPMyAdmin、Navicat、MySQL Workbench、mysqldump、Mydumper/Myloader）、客户端等均适用于 TiDB。

> 但 TiDB 尚未支持一些 MySQL 功能，可能的原因如下：

有更好的解决方案，例如 JSON 取代 XML 函数。
目前对这些功能的需求度不高，例如存储过程和函数。
一些功能在分布式系统上的实现难度较大。

> 不支持的功能特性

https://docs.pingcap.com/zh/tidb/stable/mysql-compatibility/

- 存储过程与函数
- 触发器
- 事件
- 自定义函数
- 全文语法与索引 
- 空间类型的函数（即 GIS/GEOMETRY）、数据类型和索引 
- 非 ascii、latin1、binary、utf8、utf8mb4、gbk 的字符集
- SYS schema
- MySQL 追踪优化器
- XML 函数
- X-Protocol 
- 列级权限 
- XA 语法（TiDB 内部使用两阶段提交，但并没有通过 SQL 接口公开）
- CREATE TABLE tblName AS SELECT stmt 语法 
- CHECK TABLE 语法 
- CHECKSUM TABLE 语法 
- REPAIR TABLE 语法
- OPTIMIZE TABLE 语法
- HANDLER 语句
- CREATE TABLESPACE 语句
- "Session Tracker: 将 GTID 上下文信息添加到 OK 包中"
- JOIN 的 ON 子句的子查询 

> 不兼容的特性

- TiDB 的自增列可以保证唯一，但只能能保证在单个 TiDB server 中自增， 在使用UDTS迁移时打开`兼容MySQL自增模式`选项，能保证在多个 TiDB server 中自增，但不保证自动分配的值的连续性。
```
在后续TiDB使用时建表语句需要添加 AUTO_ID_CACHE 1， 如:

CREATE TABLE `test` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(20) NOT NULL,
  `age` int(11) NOT NULL,
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 AUTO_ID_CACHE 1;
```

- TiDB 可通过 tidb_allow_remove_auto_inc 系统变量开启或者关闭允许移除列的 AUTO_INCREMENT 属性。删除列属性的语法是：ALTER TABLE MODIFY 或 ALTER TABLE CHANGE。
- TiDB 不支持添加列的 AUTO_INCREMENT 属性，移除该属性后不可恢复。
- TiDB 中utf8与utf8mb4的默认排序规则分别为utf8_bin和utf8mb4_bin，与MySQL的默认排序规则utf8_general_ci和utf8mb4_general_ci不同，MySQL源为5.7时字符集为utf8排序规则为utf8_general_ci的表迁移到TiDB时，默认排序规则为utf8_bin，字符集为utf8mb4排序规则为utf8_general_ci的表迁移到TiDB时，默认排序规则为utf8mb4_bin， 如果业务强制要求大小写不敏感，请参照以下步骤修改:
```
1. mysql utf8mb4完全兼容utf8, 可以统一修改源库的表字符集为utf8mb4，此操作会锁表不影响读，会阻塞写操作：
alter table table_name convert to character set utf8mb4 collate utf8mb4_general_ci;

1. 源库如果单独指定字段的字符集为utf8， 也需要单独修改字段的字符集为utf8mb4排序规则为utf8mb4_general_ci：
alter table table_name modify column_name varchar(200) character set utf8mb4 collate utf8mb4_general_ci;

1. 设置目标TiDB default_collation_for_utf8mb4 参数值为 utf8mb4_general_ci;

TiDB sererless 版本执行 set global default_collation_for_utf8mb4="utf8mb4_general_ci";
TiDB 固定规格版本在参数管理中修改 default_collation_for_utf8mb4 参数值为 utf8mb4_general_ci;
```
- TiDB 默认单条SQL使用内存上限最大为1G，可以通过修改`tidb_mem_quota_query`调整，默认当SQL语句超过1G时，会报错：
```
INSERT INTO db1.tbl_test01 SELECT * FROM db2.tbl_test02;

数据量较小的情况下TiDB 可以执行， 但是当事务占用内存超过1G, TiDB会取消SQl执行并报错：
Your query has been cancelled due to exceeding the allowed memory limit for a single SQL query. Please try narrowing your query scope or increase the tidb_mem_quota_query limit and try again.

导入较大数据量时，请使用UDTS或者执行脚本分批次导入。
``` 