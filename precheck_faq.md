# FAQ

### 1 MySQL

#### 1.1
**错误信息：** 

`binlog_format is xxx, and should be ROW` 或
`binlog_row_image is %s, and should be FULL`

**解决方法：** 

如果迁移的任务类型为 `增量`、`全量+增量`, 或者全量任务结束后需要进行增量迁移，需要源库开启 binlog，并设置 `binlog_format` 为 `ROW`，`binlog_row_image` 为 `FULL`。可以根据情况选择下面的方式之一来完成 binlog 的变更

###### 1.1.1 修改配置文件（默认为 my.cnf ），重启 MySQL

```
[mysqld]
...
binlog_format = ROW
binlog_row_image = FULL
...
```

备注： 如果是 MySQL 5.5 ，没有 binlog_row_image 这个变量，不需要设置

###### 1.1.2 通过 MySQL 命令设置

需要特别注意的是，如果通过 MySQL 命令设置 binlog_format，当 MySQL 存在连接往数据库中写入数据时，写入的 binlog_format 还是老的值，需要将连接断开后才会生效。

```sql
FLUSH TABLES WITH READ LOCK;
FLUSH LOGS;
SET GLOBAL binlog_format = 'ROW';
-- 同样的，如果是 MySQL 5.5 ，不需要设置 binlog_row_image
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

FLUSH LOGS;
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
	再次通过 `show global variables like 'binlog_format';` 查询到的值还是旧的值，需要重新断开连接再次连接后才会显示变更后的值。

#### 1.2
**错误信息：** 

`MyISAM table in the source db and gtid_mode in the target db may conflict`

**解决方法：** 

如果源库需要迁移的表中包括 MyISAM 引擎表，同时目标库开启了 gtid ，可能导致 MySQL 1785 错误，报错信息如下：
```
When @@GLOBAL.ENFORCE_GTID_CONSISTENCY = 1, updates to non-transactional tables can only be done in either autocommitted statements or single-statement transactions, and never in the same statement as updates to transactional tables
```
建议用户将 MyISAM 引擎表转换为 InnoDB 引擎表，或者关闭目标库的 gtid 模式。
查询方式：
```
# 在源库中查询数据库db1中是否存在 MyISAM 表
select table_schema, table_name
	from information_schema.tables
	where engine = 'MyISAM'
		and table_type = 'BASE TABLE'
		and table_schema in (db1);

# 在目标库中查询是否开启了 gtid
show global variables like 'gtid_mode';
```

设置方式：
```
# 方案一：修改源库
# 将 MyISAM 表 table1 的引擎修改为 InnoDB
alter table table1 ENGINE = InnoDB;

# 方案二：修改目标库
# 关闭目标库的 gtid 模式
set global gtid_mode = "ON_PERMISSIVE";
set global gtid_mode = "OFF_PERMISSIVE";
set global gtid_mode = "OFF";
```



#### 1.3
**错误信息：** 

`max_allowed_packet of the source is xxx, which is larger than max_allowed_packet of the target xxx`

**解决方法：** 

源库的 `max_allowed_packet` 取值大于目标库的 `max_allowed_packet` 取值，可能导致目标库写入数据失败，建议调整`max_allowed_packet` 取值，使源目保持一致。

在目标数据库中执行语句 `set global max_allowed_packet = 2*1024*1024*10;`


### 2 TiDB


#### 2.1
**错误信息：** 

`tikv_gc_life_time is %s, and should be great than 1h`

**解决方法：** 

如果迁移的任务类型为 `全量`、`全量+增量`, 如果全量迁移的数据量较大，建议将 `tikv_gc_life_time` 值设置为 1h 以上

在数据库中执行语句 `update mysql.tidb set VARIABLE_VALUE="24h" where VARIABLE_NAME="tikv_gc_life_time";`

#### 2.2
**错误信息：** 

`TiDB dose not support charset in table %s. Please change charset to any one of '%s'.`

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