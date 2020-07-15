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

增量迁移要求 MySQL(包括UDB MySQL) binlog_format 值为 ROW

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

```
FLUSH TABLES WITH READ LOCK;
FLUSH LOGS;
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL binlog_row_image = 'FULL';
FLUSH LOGS;
UNLOCK TABLES;
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

解决方法：
1. 用户在源库，修改表，增加自定义主键



#### 11 问: sync: Type:ExecSQL msg:"exec jobs failed,err:Error 1205:Lock wait timeout exceeded;try restarting transaction"

当增量任务的目标数据库有业务运行(对数据库有读写操作)，且业务对数据库有读写锁时，UDTS 服务向目标数据库中同步数据会出现超时的情况，我们在内部重试之后，依然还没有等到锁释放，就会出现这个错误，需要用户手动启动这个任务，任务会自动从上次同步完成的位置继续执行(即支持断点续传功能)

