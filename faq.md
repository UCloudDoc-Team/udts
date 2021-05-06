# FAQ

#### 1 问: 现在支持多少区域间的跨域迁移？

UDTS 跨域迁移利用到了 UDPN或者专线，已支持区域且有UDPN/专线传输线路的路径皆可支持。已支持区域详见控制台。

#### 2 问：MySQL 全量迁移需要满足哪些条件

###### 1. 权限要求  
    MySQL(包括UDB MySQL) 全量迁移需要迁移的账号具有权限： SELECT, RELOAD, LOCK TABLES, REPLICATION CLIENT

假设你的迁移账号为 backup_user ，你可以通过以下命令查看权限，  
确保 Select_priv、Reload_priv、Lock_tables_priv、Repl_client_priv 的值为 Y

```
mysql > SELECT User,Host,Select_priv,Reload_priv,Lock_tables_priv,Repl_client_priv FROM mysql.user WHERE User = "backup_user"\G;

User             | backup_user
Host             | %
Select_priv      | Y
Reload_priv      | Y
Lock_tables_priv | Y
Repl_client_priv | Y
```

###### 2. sql_mode 一致  
为了保证迁移能正确执行，最好保持 源和目标数据库的 sql_mode 一致，可以通过一下命令查询 sql_mode

```
select @@sql_mode;

--- 或者
show variables like "sql_mode";
```

如果目标数据库和源数据的 sql_mode 不一致，可以通过下面的命令进行修改

```
SET @@GLOBAL.sql_mode='xxxx';
```

其中具体的值，可以通过连接源数据库查询。

###### 3. binlog 格式  
如果您在完成全量迁移之后，需要增加增量迁移，需要要求 binlog_format 为 ROW, binlog_row_image 为 FULL（如果有这个变量的话, MySQL 5.5 中没有这个变量）

###### 4. 视图(view) 权限  
当源数据库有视图时，需要要求迁移的账号拥有 super 权限，如果您使用的是 UDB-MySQL，可以通过下面的命令获取 super 权限

```
update mysql.user set super_priv = 'Y' where user = 'root';   
flush privileges;
```

#### 3 问： MySQL 增量迁移需要满足哪些条件

###### 1. 权限要求

MySQL(包括UDB MySQL) 增量迁移需要迁移的账号具有权限：SELECT, REPLICATION SLAVE, REPLICATION CLIENT 

假设你的迁移账号为 backup_user ，你可以通过以下命令查看权限，  

确保 Select_priv、Repl_slave_priv、Repl_client_priv 的值为 Y

```
mysql > SELECT User,Host,Select_priv,Repl_slave_priv,Repl_client_priv FROM mysql.user WHERE User = "backup_user"\G;

User                   | backup_user
Host                   | %
Select_priv            | Y
Repl_slave_priv        | Y
Repl_client_priv       | Y
```

###### 2. binlog 配置要求

增量迁移要求 MySQL(包括UDB MySQL) 的 binlog 格式为 ROW，且如果有 binlog_row_image 变量，其值需要为 FULL

可以通过下面的命令查看

```
show global variables like 'binlog_format';
show global variables like 'binlog_row_image';
```

如果增量任务启动前，binlog_format 值不为 ROW，需要再次执行全量任务，以保证 binlog 信息正确。

###### 3. 停止 event  

```
-- 停止所有 event

SET GLOBAL event_scheduler = OFF;

-- 如果按库迁移，停止指定库的 event 即可
-- 查找对应的 event 并停止
SHOW EVENTS;

ALTER EVENT hello DISABLE;
```

#### 4 问： 增量提示 only support ROW format binlog 该如何操作

增量迁移要求 MySQL(包括UDB MySQL) binlog_format 值为 ROW，可以根据情况选择下面的方式之一来完成 binlog_format 的变更; 变更完成后，需要重新执行一次 UDTS 的 `全量 + 增量`

###### 1. 修改配置文件（默认为 my.cnf ），重启 MySQL

```
[mysqld]
...
binlog_format = ROW
binlog_row_image = FULL
...
```

备注： 如果是 MySQL 5.5 ，没有 binlog_row_image 这个变量，不需要设置

###### 2. 通过 MySQL 命令设置

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
	通过 SET GLOBAL binlog_format = 'ROW'; 设置参数，  
	再次通过 show global variables like 'binlog_format'; 查询到的值还是旧的值，需要重新断开连接再次连接后才会显示变更后的值。

###### 3. 云数据

如果你使用的是云数据，可以通过复制当前使用的`配置文件`，修改对应的 `binlog_format` 和 `binlog_row_image` 配置项，然后使用该配置重新应用数据库。


#### 5 问：ERROR 1292 (22007): Incorrect date value: '0000-00-00' for column

这是由于迁移的 MySQL(UDB-MySQL) 的 sql_mode 不一致导致的， 目标库的日期不允许 '0000-00-00' 这种无效的日期。

可以通过下面的命令查看 sql_mode

```
select @@sql_mode;

--- 或者
show variables like "sql_mode";
```

通过增加 ALLOW_INVALID_DATES 来允许插入这种无效的日期, 在查询出来的 sql_mode 值的后面，加上 ALLOW_INVALID_DATES

通过以下命令对 sql_mode 进行修改

```
SET @@GLOBAL.sql_mode='xxxx,ALLOW_INVALID_DATES';
```

这里的 xxxx 是指原有查询出来的 sql_mode 值


#### 6 问： Error 1451: Cannot delete or update a parent row: a foreign key constraint fails

假设有 group、user 两个 table 如下

```
CREATE TABLE `group` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `name` varchar(100) NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;

CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_name` varchar(100) COLLATE utf8_bin NOT NULL,
  `password` varchar(512) COLLATE utf8_bin NOT NULL,
  `group_id` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `gid` (`group_id`),
  CONSTRAINT `g_fk1` FOREIGN KEY (`group_id`) REFERENCES `group` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;

INSERT INTO `group` (`id`, `name`)
VALUES
	(1, 'group1');

INSERT INTO `user` (`id`, `user_name`, `password`, `group_id`)
VALUES
	(1, 'test_user', '123456', 1);

```

当我们执行删除时

```
DELETE FROM `group` WHERE `id` = 1;
```

MySQL 会出现 

```
Cannot delete or update a parent row: a foreign key constraint fails 
    (`db1`.`user`, CONSTRAINT `g_fk1` FOREIGN KEY (`group_id`) REFERENCES `group` (`id`))
```

原因是，我们删除 group 的时候，其关联的 user 对应的数据不知道如何操作。  
对于这种情况，我们需要设置级联删除/更新规则，参考规则如下

```
ALTER TABLE tbl_name
    ADD [CONSTRAINT [symbol]] FOREIGN KEY
    [index_name] (index_col_name, ...)
    REFERENCES tbl_name (index_col_name,...)
    [ON DELETE {RESTRICT | CASCADE | SET NULL | NO ACTION | SET DEFAULT}]
    [ON UPDATE {RESTRICT | CASCADE | SET NULL | NO ACTION | SET DEFAULT}]
```

修复步骤：

1. 删除外键约束 
```
ALTER TABLE `user` DROP FOREIGN KEY `g_fk1`;
```

2. 级联更新与删除
```
ALTER TABLE `user` 
    ADD CONSTRAINT `g_fk2` 
    FOREIGN KEY (`group_id`) 
    REFERENCES `group` (`id`) 
    ON DELETE CASCADE 
    ON UPDATE CASCADE;
```

3. 点击启动任务

#### 7 问： 如何判断MySQL-MySQL增量任务中，源目数据库已经同步完成

当增量任务中目标库的数据和源库达到一致时，增量任务的状态会由 同步中 变成 已同步。

已同步 状态说明：

指迁移任务中指定的 db 和 table 对应的目标数据库数据和源数据库达到一致。比如：当迁移任务为 * 时，已同步 是指目标数据库内置库(sys, test, INFORMATION_SCHEMA，PERFORMANCE_SCHEMA 等)之外的所有 db 达到和源库数据一致。当迁移任务为多 db 时，比如 db1,db2 ，是指目标数据的 db1,db2 和源数据库数据达到一致。当源数据库数据发生任何变化(即使发生变化的 db 不在迁移任务中)，已同步 状态也会再次变化为 同步中，直到数据再次达到一致。

#### 8 问： error="Error 1040: Too many connections" 错误处理

数据库服务器连接达到上限。

```
show variables like '%max_connections%';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 151   |
+-----------------+-------+
1 row in set (0.01 sec)
```

解决方法：

1. 增加 max_connections：  `set GLOBAL max_connections=1000;`



#### 9 问： row size too large (\u003e 8126). Changing some columns to TEXT or BLOB or using ROW_FORMAT=DYNAMIC or ROW_FORMAT=COMPRESSED may help. In current row format, BLOB prefix of 768 bytes is stored inline

解决方法：

在目标库执行

```
set GLOBAL innodb_file_format_max = 'Barracuda';
set GLOBAL innodb_file_format = 'Barracuda';
set GLOBAL  innodb_file_per_table = ON;
```


#### 10 问： Column count doesn't match value count

阿里云迁移到 UCloud 的增量任务出现类似的错误， Column count doesn't match value count: 2 (columns) vs 3 (values)，是因为源库中存在隐藏主键，导致数据的 columns 和 values 不匹配。

解决方法：查找没有主键的表，并增加主键
- 查找没有主键的表

```sql
SELECT
         table_schema,
         table_name
     FROM
         information_schema.tables
     WHERE (table_schema, table_name)
     NOT in( SELECT DISTINCT
             table_schema, table_name FROM information_schema.columns
         WHERE
             COLUMN_KEY = 'PRI')
         AND table_schema NOT in('sys', 'mysql', 'INFORMATION_SCHEMA', 'PERFORMANCE_SCHEMA');
```

- 对这些没有主键的表增加主键


#### 11 问: sync: Type:ExecSQL msg:"exec jobs failed,err:Error 1205:Lock wait timeout exceeded;try restarting transaction"

当目标数据库配置较低时可能会遇到这个问题， 用户重新启动任务即可。任务会自动从上次同步完成的位置继续执行(即支持断点续传功能)

#### 12 问：MySQL 全量任务运行中会有数据库锁吗？什么时候释放

默认模式下： 全量任务运行中，为了保证数据的一致性，会执行 FTWRL（`FLUSH TABLES WITH READ LOCK`），如果迁移的数据库中存在 MyISAM 等非事务表时，锁会在这些非事务表数据备份完成之后释放（锁释放的时间与表数据大小有关）；对于 InnoDB 引擎表，会立即释放锁，并通过 `START TRANSACTION WITH CONSISTENT SNAPSHOT` 开启读一致的事务。
- 对于 MyISAM表，正在转储中的表允许读，不允许写，直到表转储完毕
- 对于 InnoDB表，正在转储中的表允许读写。

Nolock 模式下： 不会对任何数据库及表加锁。

#### 13 问：Redis 迁移出现 ERR illegal address 

原因可能是用户开启了白名单设置，但是UDTS机器的 IP 不在白名单内。 如需要白名单地址， 请联系技术支持。

#### 14 问：Error 3140: Invalid JSON text

在MySQL 同步过程中出现 Error 3140: Invalid JSON text: \"The document is empty.\" at position 0 in value for column， 原因是源库校验不严格。数据库中的字段要求为 NOT NULL，但是数据中存在值为 NULL 的数据。

有两个解决方法，根据需要处理：
- 对源库中的数据进行修复，将所有值 NULL 的数据修正为正确的值 (这也符合业务逻辑需要)
- 或者对目标库中的表进行修改，将字段修改为允许为 NULL。 例如表为 period_progress，字段为 total

```
ALTER TABLE `period_progress` CHANGE `total` `total` JSON NULL;
```

#### 15 问： Error 1264: Out of range value for column 'xxx' at row 100

在 MySQL2TiDB 的数据导入过程中，可能会出现 Error 1264: Out of range value for column 'xxx' at row 100， 这里的原因是 数据在源数据库中没有被强制校验，即非正常的数据也可以正常存放，但导入到目标时目标数据库要求数据必须满足指定的约束。

比如：

```sql
CREATE TABLE `aaa` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `lot` float(10,7) NOT NULL DEFAULT '0.0000000',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

加入源数据库中存在以上一张结构如上的表，表中要求自然 lot 的数据类型为 float，长度为 10，小数点部分为 7 位，如果数据中存在一条数据为 -1000，则导入时可能会失败，而出现 Error 1264: Out of range value for column 'lot' 的情况。

解决方案有两个，需要对源数据库进行调整：
- 调整表结构，加大字段长度

    ```sql
    select max(`lot`) from `aaa`;
    ```

    获取最长的值，然后通过 alter 修改表结构，比如将上面的表 `aaa` 的 `lot` 修改为 float(11,7)

- 修正数据，使其符合表结构的限制

    ```sql
    select `id` where `lot` > 999 OR `lot` < -999;
    ```
    
    对查找出来的数据进行修正。
