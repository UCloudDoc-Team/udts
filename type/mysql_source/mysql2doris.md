# MySQL 迁移到 Doris
UDTS 支持 从 MySQL 迁移到 Doris。
MySQL 支持版本有 MySQL(包含Percona版) 5.5/5.6/5.7/8.x。
Doris 支持的版本有 3.x/4.x。


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

### 1.2 目标Doris限制
1. 迁移过程中会自动创建`syncer_meta_schema`的数据库，禁止手动删除（否则任务失败）。
2. 主键/唯一索引字段必须定义在其他字段之前，否则创建Doris表会失败。
3. 自动创建库表时会将源库的表类型转换成Doris表模型：
   - 源库表有主键或唯一索引时，迁移到目标库后成为主键表（UNIQUE KEY）；有唯一索引但没有主键时取第一个唯一索引的列，存在多个唯一索引时只用第一个。
   - 源库表既没有主键也没有唯一索引时，成为明细表（DUPLICATE KEY）。
4. 部分列类型在Doris中没有对应类型，会转换成 STRING（见数据类型映射），这类列不能作为主键/唯一索引列，否则建表会失败。
5. 主键/唯一索引列不能是 FLOAT 或 DOUBLE 类型，否则建表会失败。
6. 对同一张表连续执行多条DDL命令可能同步失败，Doris部分DDL是异步执行的，前一条DDL未完成时执行第二条会失败。建议合并DDL语句到一条。
7. 增量同步期间主键字段不能执行修改操作，否则任务会失败。
8. Doris 表的 `light_schema_change` 属性需为 true（默认开启），关闭时修改列相关DDL（如 RENAME COLUMN）会失败。

### 1.3 全量迁移
1. 全量迁移会迁移任务指定的源库中的库表结构和数据到目标库。
2. 全量迁移会清理指定库表在目标库中的数据，但不会删除表结构，如果需要重建表结构需要客户自己在目标库删除或重建。注意：在目标库Doris创建表时，源库有主键或唯一索引需要创建主键表，都没有则创建明细表，否则可能导致任务同步失败。

### 1.4 增量迁移
1. 增量迁移会实时解析源库binlog，将增量数据同步到目标库。
2. DML操作支持INSERT、UPDATE、DELETE。如果源端表既没有主键也没有唯一索引，那么UPDATE命令不会同步。
3. DDL操作支持:
   1. CREATE/DROP DATABASE
   2. CREATE/DROP/RENAME TABLE
   3. ALTER TABLE ADD/DROP/MODIFY COLUMN（ADD COLUMN 时不支持指定列位置，新增列统一追加到表末尾）
   4. ALTER TABLE ... RENAME COLUMN（必须单独执行，不支持与 ADD/DROP/MODIFY COLUMN 等其他子句混在同一条 ALTER 中）
   5. TRUNCATE TABLE
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
| 最大速率 | 内网的速率范围为 1-1024 MB/s                                                                                                                                                              |


### 传输目标表单
  
| 参数名         | 说明                                              |
| -------------- | ------------------------------------------------- |
| 地址类型       | 目标暂时只支持内网                                |
| FE节点查询地址 | Doris FE节点查询地址IP和MySQL协议端口（默认9030） |
| HTTP地址       | Doris HTTP协议地址IP和端口                        |
| 用户名         | Doris 连接用户名                                  |
| 密码           | Doris 数据库对应用户密码                          |


## 3. 权限要求
  
| 类别      | 权限说明                                                                   |
| --------- | -------------------------------------------------------------------------- |
| 源MySQL   | SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT      |
| 目标Doris | CREATE DATABASE, CREATE TABLE, DROP, SELECT, INSERT, DELETE, UPDATE, ALTER |


## 4. 数据类型映射

### 数值类型

| MySQL数据类型      | Doris数据类型 | 备注                    |
| ------------------ | ------------- | ----------------------- |
| TINYINT            | TINYINT       |                         |
| TINYINT UNSIGNED   | SMALLINT      |                         |
| SMALLINT           | SMALLINT      |                         |
| SMALLINT UNSIGNED  | INT           |                         |
| MEDIUMINT          | INT           |                         |
| MEDIUMINT UNSIGNED | INT           |                         |
| INT                | INT           |                         |
| INT UNSIGNED       | BIGINT        |                         |
| BIGINT             | BIGINT        |                         |
| BIGINT UNSIGNED    | LARGEINT      |                         |
| BIT(M)             | LARGEINT      |                         |
| DECIMAL            | DECIMAL       | 不支持 zerofill         |
| NUMERIC            | DECIMAL       |                         |
| FLOAT              | FLOAT         | 不能作为主键/唯一索引列 |
| DOUBLE             | DOUBLE        | 不能作为主键/唯一索引列 |
| BOOL / BOOLEAN     | TINYINT       |                         |

### 日期时间类型

| MySQL数据类型    | Doris数据类型   | 备注                      |
| ---------------- | --------------- | ------------------------- |
| DATE             | DATE            |                           |
| DATETIME[(fsp)]  | DATETIME[(fsp)] |                           |
| TIMESTAMP[(fsp)] | DATETIME[(fsp)] | Doris 没有 TIMESTAMP 类型 |
| TIME[(fsp)]      | VARCHAR(16)     |                           |
| YEAR[(4)]        | INT             |                           |

### 字符串类型

| MySQL数据类型                           | Doris数据类型 | 备注                                                                                                                                        |
| --------------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| CHAR / VARCHAR                          | VARCHAR       | CHAR 和 VARCHAR(n) 类型同步到 Doris 后会转换为 VARCHAR(4*n)；Doris VARCHAR 最大只支持 65533 字节，超过时转换为 STRING                       |
| BINARY / VARBINARY                      | STRING        |                                                                                                                  |
| TINYTEXT / TEXT / MEDIUMTEXT / LONGTEXT | STRING        |                                                                                                                  |
| TINYBLOB / BLOB / MEDIUMBLOB / LONGBLOB | STRING        |                                                                                                                  |
| ENUM                                    | STRING        |                                                                                                                  |
| SET                                     | STRING        |                                                                                                                  |
| JSON                                    | JSON          |                                                                                                                  |
