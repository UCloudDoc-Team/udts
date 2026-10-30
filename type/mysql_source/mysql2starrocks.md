# MySQL 迁移到 StarRocks
UDTS 支持 从 MySQL 迁移到 StarRocks。
MySQL 支持版本有 MySQL(包含Percona版) 5.5/5.6/5.7/8.x。
StarRocks 支持的版本有3.3.19。
 

## 1. 功能限制

### 1.1 源MySQL限制

#### 1.1.1 增量/全+增迁移时，源库需要开启binlog，且格式设置为ROW, image设置为FULL。

```
查询方式:
show global variables like 'binlog_format';
show global variables like 'binlog_row_image';

设置方式：
set global binlog_format = "ROW" ;
set global binlog_row_image = "FULL" ;
```
#### 1.1.2 源库不能存在同名但大小写不一致的库或表，否则同步可能会异常，建议全部采用小写格式。
#### 1.1.3 源库不能有超过4G的binlog文件，建议大小是1G，否则同步会出错或失败。
#### 1.1.4 不支持 MySQL 8.0 的新特性 binlog 事务压缩 Transaction_payload_event。使用 binlog 事务压缩有导致上下游数据不一致的风险。
#### 1.1.5 源库表名或者库名不能包含特殊字符'.'，否则迁移失败。

### 1.2 目标StarRocks限制
1. 迁移过程中会自动创建`syncer_meta_schema`的数据库，禁止手动删除（否则任务失败）。
2. 主键类型不能是DECIMAL或者FLOAT或者DOUBLE。
3. 增量同步期间主键字段不能执行修改操作，否则任务会失败。
4. 对同一张表连续执行多条DDL命令会同步失败，StarRocks执行DDL是异步执行，第一条DDL执行未完全完成时执行第二条DDL会失败。建议合并DDL语句到一条。
5. 主键字段必须是表的第一个字段，否则创建StarRocks表会失败。
6. 自动创建库表时会将源库的表类型转换成StarRocks表类型，如果源库表有主键迁移到目标库后会成为主键表，如果没有主键（包括唯一索引的表）迁移到目标库后会成为明细表。

### 1.3 全量迁移
1. 全量迁移会迁移任务指定的源库中的库表结构和数据到目标库。
2. 全量迁移默认不会清理指定库表在目标库中的数据（数据源配置中「保留目标库数据」默认开启）。若关闭该选项，任务启动时会清理指定库表在目标库中的数据，但不会删除表结构，如果需要重建表结构需要客户自己在目标库删除或重建。注意：在目标库StarRocks创建表时，如果源库有主键需要创建主键表，没有主键（包括唯一索引的表）则创建明细表，否则可能导致任务同步失败。
3. 开启「保留目标库数据」（默认）时，目标库中已存在的表不会被重建，若表结构与源表不一致可能导致任务报错或数据不一致，目标库中相同主键的数据也可能导致任务报错，请客户自行确认或清理目标库数据。该选项与历史版本“默认清理目标库数据”的行为不同，请确认目标库配置符合预期。

### 1.4 增量迁移
1. 增量迁移会实时解析源库binlog，将增量数据同步到目标库。
2. DML操作支持INSERT、UPDATE、DELETE。如果表不含有主键，那么UPDATE命令不会同步。
3. DDL操作支持:
   1. CREATE/DROP DATABASE
   2. CREATE/DROP/RENAME TABLE
   3. ALTER TABLE ADD/DROP/MODIFY
4. 增量迁移会将源库的binlog位点信息记录在UDTS的任务中，下次增量迁移时会从上次的binlog位点开始解析。

## 2. 表单填写

### 数据源表单
  
| 参数名   | 说明                                                                                                                                                                                      |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 地址类型 | 支持内网地址，外网地址，专线地址三种方式。内网地址需要填写VPC和子网信息；外网地址支持IP和域名两种方式；专线地址既支持IP，也支持域名，如果使用域名需要用户网络有外网出口。                 |
| 端口     | MySQL连接端口                                                                                                                                                                             |
| 用户名   | MySQL连接用户名                                                                                                                                                                           |
| 密码     | MySQL数据库对应用户密码                                                                                                                                                                   |
| 数据库名 | MySQL数据库名称。 所有库传输请填写 *； 指定一个数据库传输，请填写数据库名；指定多个数据库传输，依次输入多个数据库名，库名之间使用英文逗号隔开。(如果数据库名称中包含空格则无法做增量迁移) |
| 表名     | MySQL传输表名。 只有当“数据库名”为指定一个数据库时有效。 若不填，默认为迁移指定库中的所有表； 指定一张表传输， 请填写表名； 指定多张表传输，依次输入多张表名，表名之间使用英文逗号隔开    |
| 最大速率 | 内网的速率范围为 1-1024 MB/s                                                                                                                             |
| 保留目标库数据 | 默认开启。开启时任务启动不会清理目标库数据，可能导致任务报错或数据不一致，请自行清理目标库数据；关闭时任务启动将自动清理目标库数据，清理范围参考[全量迁移](#13-全量迁移) |


### 传输目标表单
  
| 参数名   | 说明                                                                                                                                                                                                                                                             |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 地址类型 | 目标暂时只支持内网                                                                                                                                                                                                                                               |
| FE节点查询地址  | StarRocks FE节点查询地址IP和端口                                                                                                                                                                                                                                                 |
| HTTP地址  | StarRocks HTTP协议地址IP和端口                                                                                                                                                                                                                                                   |
| 用户名   | StarRocks 连接用户名                                                                                                                                                                                                                                                 |
| 密码     | StarRocks 数据库对应用户密码                                                                                                                                                                                                                                         |
| 存储卷 | 对于存算分离的StarRocks来说，可以单独指定库表所使用的存储卷，如果不指定就创建库表到默认存储卷 |


## 3. 权限要求
  
| 类别      | 权限说明                                                              |
| --------- | --------------------------------------------------------------------- |
| 源MySQL   | SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT |
| 目标StarRocks | CREATE DATABASE, CREATE TABLE, SELECT, INSERT, DELETE, UPDATE, ALTER, 存储卷的USAGE权限 |


## 4. 数据类型映射

### 数值类型

| MySQL数据类型 | StarRocks数据类型 | 备注 |
|---|---|---|
| TINYINT | TINYINT | |
| TINYINT UNSIGNED | SMALLINT | |
| SMALLINT | SMALLINT | |
| SMALLINT UNSIGNED | INT | |
| MEDIUMINT | INT | |
| MEDIUMINT UNSIGNED | INT | |
| INT | INT | |
| INT UNSIGNED | BIGINT | |
| BIGINT | BIGINT | |
| BIGINT UNSIGNED | LARGEINT | |
| BIT(M) | BIGINT | |
| DECIMAL | DECIMAL | 不支持 zerofill |
| NUMERIC | DECIMAL | |
| FLOAT | FLOAT | |
| DOUBLE | DOUBLE | |
| BOOL / BOOLEAN | TINYINT | |

### 日期时间类型

| MySQL数据类型 | StarRocks数据类型 | 备注 |
|---|---|---|
| DATE | DATE | |
| DATETIME[(fsp)] | DATETIME | v3.3.5 及以上版本支持精确到微秒 |
| TIMESTAMP[(fsp)] | BIGINT | |
| TIME[(fsp)] | VARCHAR(16) | |
| YEAR[(4)] | INT | |

### 字符串类型

| MySQL数据类型 | StarRocks数据类型 | 备注 |
|---|---|---|
| CHAR / VARCHAR | VARCHAR | 为避免数据丢失，CHAR 和 VARCHAR(n) 类型同步到 StarRocks 后会转换为 VARCHAR(4*n)。长度最大不能超过 1MiB/4=256KiB |
| BINARY / VARBINARY | VARBINARY | |
| TINYTEXT / TEXT / MEDIUMTEXT / LONGTEXT | VARCHAR | StarRocks VARCHAR 最大只支持 1MiB，超过 1MiB 会写入失败 |
| TINYBLOB / BLOB / MEDIUMBLOB / LONGBLOB | VARCHAR | StarRocks VARCHAR 最大只支持 1MiB，超过 1MiB 会写入失败 |
| ENUM | VARCHAR | |
| SET | VARCHAR | |
| JSON | JSON | |
