# FAQ

## 1 问: 现在支持多少区域间的跨域迁移？

UDTS 跨域迁移利用到了 UDPN或者专线，已支持区域且有UDPN/专线传输线路的路径皆可支持。已支持区域详见控制台。

## 2 问：MySQL 全量迁移需要满足哪些条件
[请参考](udts/type/mysqlsource.md)

## 3 问： MySQL 增量迁移需要满足哪些条件
[请参考](udts/type/mysqlsource.md)

## 4 问： 增量提示 only support ROW format binlog 该如何操作

增量迁移要求 MySQL(包括UDB MySQL) binlog_format 值为 ROW，可以根据情况选择下面的方式之一来完成 binlog_format 的变更; 变更完成后，需要重新执行一次 UDTS 的 `全量 + 增量`

### 1. 修改配置文件（默认为 my.cnf ），重启 MySQL

```
[mysqld]
...
binlog_format = ROW
binlog_row_image = FULL
...
```

备注： 如果是 MySQL 5.5 ，没有 binlog_row_image 这个变量，不需要设置

### 2. 通过 MySQL 命令设置

需要特别注意的是，如果通过 MySQL 命令设置 binlog_format，当 MySQL 存在连接往数据库中写入数据时，写入的 binlog_format 还是原值，需要将连接断开后才会生效。

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
	再次通过 `show global variables like 'binlog_format';` 查询到的值还是原值，需要断开连接后重新连接才会显示变更后的值。

### 3. 云数据

如果你使用的是云数据，可以通过复制当前使用的`配置文件`，修改对应的 `binlog_format` 和 `binlog_row_image` 配置项，然后使用该配置重新应用数据库。


## 5 问：ERROR 1292 (22007): Incorrect date value: '0000-00-00' for column

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

## 6 问： 如何判断MySQL-MySQL增量任务中，源目数据库已经同步完成

当增量任务中目标库的数据和源库达到一致时，增量任务的状态会由 同步中 变成 已同步。

已同步 状态说明：

指迁移任务中指定的 db 和 table 对应的目标数据库数据和源数据库达到一致。比如：当迁移任务为 * 时，已同步 是指目标数据库内置库(sys, test, INFORMATION_SCHEMA，PERFORMANCE_SCHEMA 等)之外的所有 db 达到和源库数据一致。当迁移任务为多 db 时，比如 db1,db2 ，是指目标数据的 db1,db2 和源数据库数据达到一致。当源数据库数据发生任何变化(即使发生变化的 db 不在迁移任务中)，已同步 状态也会再次变化为 同步中，直到数据再次达到一致。

## 7 问： error="Error 1040: Too many connections" 错误处理

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

## 8 问： row size too large (\u003e 8126). Changing some columns to TEXT or BLOB or using ROW_FORMAT=DYNAMIC or ROW_FORMAT=COMPRESSED may help. In current row format, BLOB prefix of 768 bytes is stored inline

解决方法：

在目标库执行

```
set GLOBAL innodb_file_format_max = 'Barracuda';
set GLOBAL innodb_file_format = 'Barracuda';
set GLOBAL  innodb_file_per_table = ON;
set GLOBAL innodb_strict_mode = OFF;
```


## 9 问： Column count doesn't match value count

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


## 10 问: sync: Type:ExecSQL msg:"exec jobs failed,err:Error 1205:Lock wait timeout exceeded;try restarting transaction"

当目标数据库配置较低时可能会遇到这个问题， 用户重新启动任务即可。任务会自动从上次同步完成的位置继续执行(即支持断点续传功能)

## 11 问：MySQL 全量任务运行中会有数据库锁吗？什么时候释放

默认模式下： 全量任务运行中，为了保证数据的一致性，会执行 FTWRL（`FLUSH TABLES WITH READ LOCK`），如果迁移的数据库中存在 MyISAM 等非事务表时，锁会在这些非事务表数据备份完成之后释放（锁释放的时间与表数据大小有关）；对于 InnoDB 引擎表，会立即释放锁，并通过 `START TRANSACTION WITH CONSISTENT SNAPSHOT` 开启读一致的事务。
- 对于 MyISAM表，正在转储中的表允许读，不允许写，直到表转储完毕
- 对于 InnoDB表，正在转储中的表允许读写。

Nolock 模式下： 不会对任何数据库及表加锁。

## 12 问：Redis 迁移出现 ERR illegal address 

原因可能是用户开启了白名单设置，但是UDTS机器的 IP 不在白名单内。 如需要白名单地址， 请联系技术支持。

## 13 问：Error 3140: Invalid JSON text

在MySQL 同步过程中出现 Error 3140: Invalid JSON text: \"The document is empty.\" at position 0 in value for column， 原因是源库校验不严格。数据库中的字段要求为 NOT NULL，但是数据中存在值为 NULL 的数据。

有两个解决方法，根据需要处理：
- 对源库中的数据进行修复，将所有值 NULL 的数据修正为正确的值 (这也符合业务逻辑需要)
- 或者对目标库中的表进行修改，将字段修改为允许为 NULL。 例如表为 period_progress，字段为 total

```
ALTER TABLE `period_progress` CHANGE `total` `total` JSON NULL;
```

## 14 问：Mongodb 迁移出现 error reading collection: cursor id 5707195885304103447 not found

mongodb 的cursor 默认有效时间是10分钟， 如果数据无法在10分钟内处理完成，cursor将会过期，导致无法获取剩下的数据
解决方法
- 登录mongodb 的shell 切换至admin库 将cursor超时时间设置成1天
```
db.runCommand( { setParameter:1 , "cursorTimeoutMillis":86400000}  )
```

## 15 问： Error 1264: Out of range value for column 'xxx' at row 100

在 MySQL2TiDB 的数据导入过程中，可能会出现 Error 1264: Out of range value for column 'xxx' at row 100， 这里的原因是 数据在源数据库中没有被强制校验，即非正常的数据也可以正常存放，但导入到目标时目标数据库要求数据必须满足指定的约束。

比如：

```sql
CREATE TABLE `aaa` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `lot` float(10,7) NOT NULL DEFAULT '0.0000000',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

假如源数据库中存在一张结构如上的表，表中要求字段 `lot` 的数据类型为 `float`，长度为 10，小数点部分为 7 位，如果数据中存在一条数据为 -1000 的数据，则导入时可能会失败，而出现 Error 1264: Out of range value for column 'lot' 的情况。

解决方案有两个，需要对源数据库进行调整：
- 调整表结构，加大字段长度

    ```sql
    select max(`lot`) from `aaa`;
    select min(`lot`) from `aaa`;
    ```

    获得最大值和最小值，然后通过 alter 修改表结构，使其符合具体的条件。比如将上面的表 `aaa` 的 `lot` 修改为 float(11,7)

    ```sql
    ALTER TABLE `aaa` MODIFY `lot` float(11,7) NOT NULL DEFAULT '0.0000000',
    ```

- 修正数据，使其符合表结构的限制

    ```sql
    select `id` where `lot` > 999 OR `lot` < -999;
    ```
    
    对查找出来的数据进行修正。

## 16 问：目标库为TiDB时，出现 Error 1071: Specified key was too long; max key length is 3072 bytes

需要增大目标库TiDB文件中的配置项`max-index-length`

## 17 问：目标库为TiDB时，出现 Error: incorrect utf8 value f09f8c80 for column xxx

需要设置目标库tidb_skip_utf8_check
```sql
set global tidb_skip_utf8_check=1;
```

## 18 问：目标库为TiDB时，出现 Error 1071: Specified key was too long; max key length is 3072 bytes

可以通过将TiDB配置文件中 `max-index-length` 项的值改成 12288 来解决

## 19 问： Cannot add foreign key constraint

在 MySQL 迁移到 MySQL/TiDB 时，如果出现 Cannot add foreign key constraint，需要检查表结构的 CHARSET，当前表结构及其依赖的外键的表结构 CHARSET 要一致。

以下为示例，当 group 和 user 表的 CHARSET 不同时，可能会出现错误： Cannot add foreign key constraint

```sql
CREATE TABLE `group` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(255) NOT NULL,
  `password` varchar(255) DEFAULT NULL,
  `g_id` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  FOREIGN KEY (`g_id`) REFERENCES `group`(`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

出现该问题时如何处理：
1. 在源数据库中，执行以下 SQL，修改表的 CHARSET，推荐使用兼容的编码替换老的编码(使用 utf8mb4 替换 utf8)，使 `user` 和 `group` 的 CHARSET 一致。

```sql
ALTER TABLE `user` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
```

2. 重新运行 UDTS 任务

## 20 问：源库是MySQL，增量同步时出现 ERROR 1236 (HY000): The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires

这是因为增量同步需要的binlog同步点，无法在源库中找到。可能的原因有以下四种：
- 1 用户是全+增任务，全量的时间超过了binlog的最大过期时间，导致全量结束时，记录的binlog同步点被过期清除了。这种情况需要调大源库binlog的过期时间。
- 2 用户源库的binlog被手动清除。
- 3 用户的任务中途暂停或者失败过，然后间隔一段时间后再启动的任务，在此期间binlog过期被清除了。
- 4 用户增量同步没有使用GTID进行同步，并且发生主从切换。

解决方法：以上情况都需要重新执行一次 UDTS 的 全量 + 增量，直接重启任务无法解决该问题。
步骤：
- 对于全量 + 增量任务，在任务详情页点击编辑任务，填入相关任务信息，然后选择保存任务。当前任务的进度会被清除，启动任务后会重新从全量开始。
- 对于直接创建增量任务的，需要重新做一次全量迁移，然后重新填写正确的`BinlogName`、`BinlogPos`、`GTID`，启动任务之后会从指定的 binlog 位置开始同步。

关于如何修改binlog日志过期时间有以下两种方式:
- 1  临时生效，重启数据库后失效<br/><br/>  
  
  查看默认设置的过期时间
    ```sql
    show variables like "%expire_logs%";
    ```
  设置保留 15 天
    ```sql
    set global expire_logs_days=15
    ```
  刷新日志
    ```sql
    flush logs；
    ```
  查看新生成的 binlog 日志
    ```sql
    show master status\G:
    ```
  注意：以上命令在数据库执行会立即生效，请确定设置数据的保留日期，以免误删
 

- 2 修改配置文件，永久生效 <br /><br /> 

  MySQL 配置文件的位置请根据实际情况进行修改
    ```shell
    vim /etc/my.cnf
    ```
  修改有关binlog过期相关的配置
    ```
    [mysqld]
    expire_logs_days=15
    ```
  注意：在配置文件修改后，需要重启才能永久生效。设置为 0 表示永不过期，单位是天。

## 21 问： The table 'xxx' is full 或者 No space left on device

一般情况是目标磁盘空间不足导致的，需要升级目标磁盘空间，或者清理目标数据库中的数据。

The table 'xxx' is full 还有一种特殊的情况是 table 使用了 memory 存储引擎  
可以通过修改 MySQL 的配置文件的下面两个值来解决(或者将 table 修改为其它存储引擎)，该值默认为 16M  

```
tmp_table_size = 256M
max_heap_table_size = 256M
```

这个值设置成多少合适呢？这个需要根据实际情况处理
- 当前 MySQL 实例拥有的内存大小是多少，则该值为上限
- memory 引擎表期望存储的数据量大小是多少

## 22 问： Error 1785: When @@GLOBAL.ENFORCE_GTID_CONSISTENCY = 1, updates to non-transactional tables can only be done in either autocommitted statements or single-statement transactions, and never in the same statement as updates to transactional tables

在 MySQL 的同一个事务中，如果既操作了支持事务的表(比如 innodb)，又操作了不支持事务的表(比如 myisam)，在数据库开启 gtid 的情况下，会出现该错误。

解决方法，下面选择其一即可
- 在源数据库中，将非事务引擎的表(一般情况下是 myisam)修改为事务引擎的表(一般情况下是 innodb)
- 在目标数据库中，关闭 gtid

## 23 问： sql_mode may cause error. check sql_mode NO_ZERO_DATE/NO_ZERO_IN_DATE in target db

在预检查的过程中，如果目标数据库的 sql_mode 存在 NO_ZERO_DATE 或者 NO_ZERO_IN_DATE，则会提示该信息。是否可以忽略该信息需要确认源数据库中是否存在 0000-00-00 这样的日期值和 00:00:00 这样的时间值，如果存在这样的时间值，在加载数据的时候会出现错误。

解决方法：
- 如果确认源数据库中没有 0000-00-00 和 00:00:00 这样的日期或时间值，可以忽略该信息
- 修改目标数据库中的 sql_mode，去掉 NO_ZERO_DATE 和 NO_ZERO_IN_DATE，操作步骤可以参考 FAQ 第 5 条

## 24 问： error creating indexes for xx.xx: createIndex error: WiredTigerIndex::insert: key too large to index, failing  1089

MongoDB 在2.6 版本引入 failIndexKeyTooLong 参数，在4.2版本之后废弃该参数，用于限制 index 的长度

解决方法：在目标数据库执行
```
db.runCommand({
    "setParameter": 1,
    "failIndexKeyTooLong": false}
);
```

## 25 问： OOM command not allowed when used memory > 'maxmemory' or rss_memory not enough.

目标数据库是MongoDB分片集群时，目标分片的容量可能出现内存不够的情况

解决方法：升级目标集群的分片容量

