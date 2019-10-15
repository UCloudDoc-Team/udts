# FAQ

{{indexmenu_n>4}}


#### 问: 现在支持多少区域间的跨域迁移？

UDTS 跨域迁移利用到了 UDPN或者专线，现在部署的服务区域为北京，上海，新加坡。以此3个区域有UDPN/专线传输线路的区域皆可支持。

#### 问：MySQL 全量迁移需要满足哪些条件

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

#### 问： MySQL 增量迁移需要满足哪些条件

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

#### 问： 增量提示 only support ROW format binlog 该如何操作

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

#### 问：ERROR 1292 (22007): Incorrect date value: '0000-00-00' for column

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


#### 问： Error 1451: Cannot delete or update a parent row: a foreign key constraint fails

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

比如

```
-- 级联更新与删除
ALTER TABLE `user` 
    ADD CONSTRAINT `g_fk2` 
    FOREIGN KEY (`group_id`) 
    REFERENCES `group` (`id`) 
    ON DELETE CASCADE 
    ON UPDATE CASCADE;
```

