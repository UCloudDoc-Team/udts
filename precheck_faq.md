# 预检查错误信息及解决方法

## 1 MySQL

### 1.1
**错误信息：** 

`log_bin is xxx, and should be ON`

`binlog_format is xxx, and should be ROW`

`binlog_row_image is xxx, and should be FULL`

`log_bin: xxx，需要修改为 ON`

`binlog_format: xxx，需要修改为 ROW` 

`binlog_row_image: xxx，需要修改为 FULL`

**解决方法：** 

如果涉及增量同步，包括 `增量任务`、`全量+增量任务`、`双向同步`以及`全量任务`之后再进行`增量任务`的场景，要求源数据库开启 binlog 功能，且

- `binlog_format` 为 `ROW`
- `binlog_row_image` 为 `FULL`

备注：
	如果只进行全量迁移，可以忽略这个问题

#### 1.1.1 源库未开启过 binlog
修改配置文件（默认为 my.cnf ），重启 MySQL
```
[mysqld]
...
log-bin = /data/mysql/logs
binlog_format = ROW
binlog_row_image = FULL
...
```

如果您使用的是云厂商的数据库服务，需要修改对应的配置文件，使用修改后的配置重新启动数据库。

备注： 
	MySQL 5.5 没有 binlog_row_image 这个变量，不需要设置

#### 1.1.2 源库开启过 binlog，但是 binlog_format 或 binlog_row_image 值不对

需要特别注意的是，如果通过 MySQL 命令设置 binlog_format，当 MySQL 存在连接往数据库中写入数据时，写入的 binlog_format 还是原值，需要将连接断开后才会生效。

```sql
FLUSH TABLES WITH READ LOCK;
FLUSH LOGS;
SET GLOBAL binlog_format = 'ROW';
-- 同样的，MySQL 5.5 不需要设置 binlog_row_image
SET GLOBAL binlog_row_image = 'FULL';
FLUSH LOGS;
UNLOCK TABLES;
```

变更完成后，可以通过以下命令断开现有的连接

```sql
-- 查看当前所有连接
> show processlist;
+-----+------+-----------------+--------+---------+------+----------+------------------+
| Id  | User | Host            | db     | Command | Time | State    | Info             |
+-----+------+-----------------+--------+---------+------+----------+------------------+
| 495 | root | 10.20.5.1:56820 | <null> | Query   | 0    | starting | show processlist |
| 497 | root | 10.20.5.1:56828 | <null> | Sleep   | 3    |          | <null>           |
+-----+------+-----------------+--------+---------+------+----------+------------------+

-- 通过 kill 断开所有 session，如果能确认哪些连接有写操作，可以只 kill 这些连接
> kill 497
-- kill 掉原来的 session 之后，flush logs 保证新的 binlog 格式为 ROW
> FLUSH LOGS;
```


如果你使用的是 master-slave 模式，需要执行以下命令

在 slave 节点上执行 

```
stop slave;
```

在 master 上执行

```
FLUSH TABLES WITH READ LOCK;
FLUSH LOGS;
SET GLOBAL binlog_format = 'ROW';
FLUSH LOGS;
UNLOCK TABLES;
```

在 slave 上执行

```
start slave;
```

备注：  
	通过 `SET GLOBAL binlog_format = 'ROW';` 设置参数，  
	再次通过 `show global variables like 'binlog_format';` 查询到的值还是原值，需要断开连接后重新连接才会显示变更后的值。

### 1.2

#### 1.2.1
**错误信息：** 

`The tables that need to be migrated from the source database include MyISAM engine tables, while the target database has GTID enabled, which may result in migration task failure.`

`源库需要迁移的表中包括 MyISAM 引擎表，同时目标库开启了 GTID ，可能导致迁移失败`

**解决方法：** 

如果源库需要迁移的表中包括 MyISAM 引擎表，同时目标库开启了 GTID ，可能导致 MySQL 1785 错误，报错信息如下：
```
When @@GLOBAL.ENFORCE_GTID_CONSISTENCY = 1, updates to non-transactional tables can only be done in either autocommitted statements or single-statement transactions, and never in the same statement as updates to transactional tables
```
建议用户将 MyISAM 引擎表转换为 InnoDB 引擎表，或者关闭目标库的 GTID 模式。

查询方式：
```
# 在源数据库（db1）中查询是否存在 MyISAM 表
select table_schema, table_name
	from information_schema.tables
	where engine = 'MyISAM'
		and table_type = 'BASE TABLE'
		and table_schema in (db1);

# 在目标库中查询是否开启了 GTID
show global variables like 'gtid_mode';
```

设置方式：
```
# 方案一：修改源库
# 将 MyISAM 表 table1 的引擎修改为 InnoDB
alter table table1 ENGINE = InnoDB;

# 方案二：修改目标库
# 关闭目标库的 GTID 模式
set global gtid_mode = "ON_PERMISSIVE";
set global gtid_mode = "OFF_PERMISSIVE";
set global gtid_mode = "OFF";
```

#### 1.2.2
**错误信息：** 

`The source database contains a large number of tables, and the check for table engine timeout. Please check if there are any MyISAM engine tables in the tables to be migrated.`

`源库中表的数量太多，检查表引擎超时。请自行检查要迁移的库表中是否存在 MyISAM 引擎表。`

**解决方法：** 

参考 1.2.1 解决方法。


### 1.3
**错误信息：** 

`max_allowed_packet of the source is xxx, which is larger than max_allowed_packet of the target yyy`

`源库 max_allowed_packet 值 xxx 大于目标库 max_allowed_packet 值 yyy`

**解决方法：** 

源库的 `max_allowed_packet` 值大于目标库的 `max_allowed_packet` 时，可能导致目标库写入数据失败，建议调整目标数据库的 `max_allowed_packet` 值，使源目保持一致。

在源库中执行
```
> show global variables like "max_allowed_packet";
+--------------------+---------+
| Variable_name      | Value   |
+--------------------+---------+
| max_allowed_packet | 4194304 |
+--------------------+---------+
```
然后使用获取的 max_allowed_packet 值，在目标数据库中执行，比如
```
set global max_allowed_packet = 4194304;
```

### 1.4

#### 1.4.1
**错误信息：** 

`table xxx have no primary key or at least a unique key`

`表：xxx 需要主键或唯一键`

**解决方法：** 
如果迁移涉及到增量同步，包括 `增量任务`、`全量+增量任务`、`双向同步`以及`全量任务`之后再进行`增量任务`的场景，需要为每张表设置主键或唯一键，否则在增量同步阶段可能出现重复数据。如果只进行全量迁移，可以忽略这个问题。
```
alter table xxx add primary key(xxxx);
```

#### 1.4.2
**错误信息：** 

`The source database contains a large number of tables, and the check for table primary key timeout. Please check if there are any primary keys or unique keys in the tables to be migrated.`

`源库中表的数量太多，检查主键超时。请自行检查要迁移的库表中是否存在无主键和唯一键的表。`

**解决方法：** 

在源库中执行如下命令，检查无主键和唯一键的表。请拆分待迁移库表，分批执行查询语句以避免影响源库性能。
```sql
SELECT
  t1.table_schema,
  t1.table_name
FROM
  information_schema.TABLES t1
  LEFT OUTER JOIN information_schema.TABLE_CONSTRAINTS t2 ON t1.table_schema = t2.table_schema
  AND t1.table_name = t2.TABLE_NAME
  AND t2.CONSTRAINT_TYPE IN ('PRIMARY KEY', 'UNIQUE')
WHERE
  t2.table_name IS NULL
  AND t1.table_type = 'BASE TABLE'
  AND t1.table_schema in ("dbname1", "dbname2") --- 此处替换为待迁移的库名
  AND t1.table_name in ("tablename1", "tablename2") --- 指定表迁移时，此处替换为待迁移的表名。整库迁移时，删除这一行。
;
```
如果源库中存在无主键和唯一键的表，参考 1.4.1 解决方法。

### 1.5
**错误信息：** 

`sql_mode may cause error. Please check sql_mode NO_ZERO_DATE/NO_ZERO_IN_DATE in target db`

`sql_mode 可能导致报错，检查目标库的 sql_mode NO_ZERO_DATE/NO_ZERO_IN_DATE。`

**解决方法：** 
如果源库与目标库的 sql_mode 不一致，可能会导致部分数据无法迁移到目标库。
```
# 在源库中查询 sql_mode
show variables like "sql_mode";

# 在目标库中修改 sql_mode，使其与源库保持一致
SET GLOBAL sql_mode='xxxx';
```
### 1.6
**错误信息：** 

`log_slave_updates should be ON`

`log_slave_updates 需要修改为 ON`

**解决方法：** 
如果您使用从库作为迁移的源，那么要求从库开启 log_slave_updates，否则从库中没有 binlog 日志，导致无法迁移

修改配置文件（默认为 my.cnf ），重启 MySQL

```
[mysqld]
...
log_slave_updates = 1
...
```

### 1.7
**错误信息：** 

`please stop event: 'db1':event1`

`增量同步前，需要关闭 event: 'db1':event1`

**解决方法：** 

增量同步开启之前，需要关闭 event

```
# 停止所有 event
SET GLOBAL event_scheduler = OFF;

# 如果按库迁移，停止指定库的 event 即可
# 查找对应的 event 并停止
USE db1;
SHOW EVENTS;
ALTER EVENT event1 DISABLE;
```

### 1.8
**错误信息：** 

`The variable innodb_xxx has different values in source and target. Please modify the variables to ensure consistency.`

`源库与目标库的参数 XXX 取值不同，建议修改参数保持一致`

**解决方法：** 

源库与目标库的 innodb 相关参数不一致，可能导致数据迁移报错，建议修改目标库参数，与源库保持一致

```
# 在目标库中修改不一致的参数取值
set GLOBAL innodb_file_format = 'Barracuda';
set GLOBAL innodb_strict_mode = OFF;
```

### 1.9
**错误信息：** 

`The variable lower_case_table_names has different values in source and target. Please modify the variables to ensure consistency.`

`源库与目标库的参数 lower_case_table_names 取值不同，建议修改参数保持一致`

**解决方法：** 

源库与目标库的 lower_case_table_names 参数不一致，可能导致数据迁移报错，建议修改目标库参数，与源库保持一致

修改配置文件（默认为 my.cnf ），重启 MySQL
```
[mysqld]
...
lower_case_table_names = 0
...
```

如果您使用的是云厂商的数据库服务，需要修改对应的配置文件，使用修改后的配置重新启动数据库。

### 1.10

#### 1.10.1
**错误信息：** 

`The size of the source database xxxGB has exceeded the limit of service type.`

**解决方法：**

源库数据量超过资源类型限制。请参考 [实例类型](/udts/introduction/instancetype) 选择合适的资源类型。

#### 1.10.2
**错误信息：** 

`The size of the source database xxxGB has exceeded the limit of target database disk size. Please expand target database disk.`

**解决方法：**

目标库为 UDB 时，UDTS 会检查目标库硬盘容量。源库数据量超过目标库硬盘容量，请扩容目标库硬盘。

#### 1.10.3
**错误信息：** 

`The size of the source database xxxGB has exceeded half of target database disk size. Please use NoBinlog mode or expand target database disk.`

**解决方法：**

目标库为 UDB 时，UDTS 会检查目标库硬盘容量。源库数据量超过目标库硬盘容量的一半，请采用 “NoBinlog” 模式或扩容目标库硬盘。

#### 1.10.4
**错误信息：** 

`The source database contains a large number of tables, and the check for table data size timeout. Please check if the data size of the tables to be migrated exceeds the service type limit.`

`源库中表的数量太多，检查数据量超时。请自行检查要迁移的数据量是否超过资源类型限制。`

**解决方法：**

在源库中执行如下命令，检查源库中待迁移的库表数据量。请拆分待迁移库表，分批执行查询语句以避免影响源库性能。
```sql
SELECT
  sum(data_length + index_length) / 1024 / 1024 / 1024 as datasize
FROM
  information_schema.tables
WHERE
  table_schema IN ("dbname1", "dbname2") --- 此处替换为待迁移的库名
  AND table_name IN ("tablename1", "tablename2") --- 指定表迁移时，此处替换为待迁移的表名。整库迁移时，删除这一行。
;
```
请参考 [实例类型](/udts/introduction/instancetype) 选择合适的资源类型。

## 2 TiDB


### 2.1
**错误信息：** 

`tikv_gc_life_time is xxx, and should be great than 1h`

`tikv_gc_life_time: xxx，需要不小于 1h`

**解决方法：** 

如果迁移的任务类型为 `全量任务`、`全量+增量任务`,  且迁移的数据量较大时，需要 `tikv_gc_life_time` 的值大于 `转存` 时间。
一般情况下 2T 的数据，大约需要 45min，我们建议将该值设置为 1h 以上

在 TiDB 中执行以下语句
```
update mysql.tidb set VARIABLE_VALUE="1h" where VARIABLE_NAME="tikv_gc_life_time";
```

### 2.2
#### 2.2.1
**错误信息：** 

`TiDB dose not support charset in table xxx. Please change charset to any one of 'ascii/latin1/binary/utf8/utf8mb4'.`

`TiDB 不支持此表采用的字符集 xxx ，请修改字符集为 'ascii/latin1/binary/utf8/utf8mb4' 中任意一种`

**解决方法：** 


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

#### 2.2.2
**错误信息：** 

`The source database contains a large number of tables, and the check for table charset timeout. Please check if the tables to be migrated are using charsets that are not supported by the target database.`

`源库中表的数量太多，检查表字符集超时。请自行检查要迁移的库表中是否使用了目标库不支持的字符集。`

**解决方法：**

在源库中执行如下命令，检查源库中待迁移的库表字符集。请拆分待迁移库表，分批执行查询语句以避免影响源库性能。
```sql
SELECT
  table_schema,
  table_name,
  table_collation
FROM
  information_schema.tables
WHERE
  table_type = "BASE TABLE"
  AND table_schema IN ("dbname1", "dbname2") --- 此处替换为待迁移的库名
  AND table_name IN ("tablename1", "tablename2") --- 指定表迁移时，此处替换为待迁移的表名。整库迁移时，删除这一行。
;

SELECT
  table_schema,
  table_name,
  column_name,
  character_set_name
FROM
  information_schema.columns
WHERE
  table_schema IN ("dbname1", "dbname2") --- 此处替换为待迁移的库名
  AND table_name IN ("tablename1", "tablename2") --- 指定表迁移时，此处替换为待迁移的表名。整库迁移时，删除这一行。
;
```
TiDB目前支持的字符集包括`ascii/latin1/binary/utf8/utf8mb4`。如果源库中存在目标库不支持的字符集，参考 2.2.1 解决方法。

## 3 MongoDB

### 3.1
**错误信息：** 

`Source and target version do not match, source verion is 3.0, and target version is 5.0`

`源库与目标库版本不匹配，源库： 3.6.23 ，目标库： 5.0.14`

**解决方法：** 

Mongo 暂不支持跨大版本迁移，从3.x迁移到5.x时，需要创建4.x版本的中转库，先从3.x迁移到4.x，再从4.x迁移到5.x。

## 4 Redis

### 4.1
**错误信息：** 

`Source and target version do not match, source verion is 4.0, and target version is 7.0`

`源库与目标库版本不匹配，源库： 4.0 ，目标库： 7.0`

**解决方法：** 

Redis 跨大版本迁移可能存在兼容性问题，建议通过中转库迁移，例如，从3.x/4.x迁移到7.x时，需要创建5.x/6.x版本的中转库，先从3.x/4.x迁移到5.x/6.x，再从5.x/6.x迁移到7.x。

### 4.2
**错误信息：** 

`The source database version is 7.0, and does not support rump mode`

`源库版本 7.0 不支持 rump 模式`

**解决方法：** 

Redis 7.0 版本不支持 rump 模式，建议源库开启 psync 权限再进行迁移。

### 4.3
**错误信息：** 

`In Rump mode, migration from versions above 7.0 to below 7.0 is not supported.`

`Rump 模式下不支持从 redis 7.0 以上版本到redis 7.0 以下版本`

**解决方法：** 

Rump 模式下不支持从 redis 7.0 以上版本到redis 7.0 以下版本，建议迁移到相同版本。

### 4.4
**错误信息：** 

`The notify-keyspace-events value in the source Redis configuration does not contain 'A' or 'E'.`

`Redis 全+增迁移在 Rump 模式下， 源库notify-keyspace-events配置型必须包含 'A' or 'E' `

**解决方法：** 

设置源库 notify-keyspace-events 配置项， 必须包含 'AE'。
