
# MySQL

UDTS支持MySQL作为数据传输源/目标，支持版本有 MySQL(包含Percona版)5.5/5.6/5.7/8.0; MariaDB 10.1.2 及以上，以及PolarDB(MySQL兼容版本)。


## 迁移内容

全量和增量迁移以下内容
- Database、Table 结构及数据
- 视图(View)
- 函数(Function)、存储过程(Procedure)

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

        ```sql
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

        ```
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

- 连接性检查，包括主机地址、端口、用户名和密码等信息的正确性
- 权限检查，检查迁移时需要的权限
- sql_mode 检查
- binlog 格式检查

### 连接性检查
连接性检查会对填写的 `主机地址`,`端口`，`用户名`，`密码`进行检查

- 主机连通性检查：如果填写了错误的`主机地址`或者`端口`，会出现连通性检查失败： `Project OR IP:10.19.37.212 OR Vpc:uvnet-u0ecvp, Subnet:subnet-2slodr error, please check again`, 
请确认当前的 `主机地址`，`端口`，选择的 `VPC ID`,`所属子网`，以及`项目`是否正确。

- 用户名密码检查：如果填写了错误的用户名或者密码，会出现连通性检查失败: `Error 1045: Access denied for user 'root1'@'10.42.255.213' (using password: YES)`, 请确认用户名和密码正确。

- 防火墙和白名单：如果主机地址，端口，用户名和密码均正确
  * 如果 MySQL 为主机自建，需要排查主机的 iptables 规则
  * 如果设置了白名单，请联系技术支持获取白名单地址

### 权限检查

默认配置迁移需要源数据库账号拥有 super 权限， 如果没有 super 权限，请选择 "NoLocks" 模式。 在 "NoLocks" 模式下，需要所有表都有显式主键或者唯一键，否则增量同步过程中可能产生重复数据。

| 类型/权限 | 源库权限(开启 NoLocks)                                                          | 源库权限(未开启 NoLocks)                                                                                 | 目标库权限                                                                                                                   |
|---------|-------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| 全量      | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW" , "PROCESS" | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW", "RELOAD", "LOCK TABLES" , "PROCESS" | "SELECT", "INSERT", "UPDATE", "CREATE", "DROP", "ALTER", "DELETE", "INDEX", "CREATE VIEW", "CREATE ROUTINE"                  |
| 增量      | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW"             | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW"                                      | "SELECT", "INSERT", "UPDATE", "CREATE", "DROP", "ALTER", "DELETE", "INDEX", "CREATE VIEW", "CREATE ROUTINE", "ALTER ROUTINE" |
| 全+增     | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW" , "PROCESS" | "SELECT", "REPLICATION SLAVE", "REPLICATION CLIENT", "SHOW VIEW", "RELOAD", "LOCK TABLES" , "PROCESS" | "SELECT", "INSERT", "UPDATE", "CREATE", "DROP", "ALTER", "DELETE", "INDEX", "CREATE VIEW", "CREATE ROUTINE", "ALTER ROUTINE" |

### sql_mode 检查

为了保证迁移能正确执行，最好保持 源和目标数据库的 sql_mode 一致，可以通过一下命令查询 sql_mode

如果 sql_mode 不一致，可以通过以下命令查看和修改 sql_mode

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

### binlog 格式检查

如果迁移的任务类型为 `增量`、`全量+增量`, 或者全量任务后期需要进行增量迁移，需要源库开启 binlog，且格式设置为ROW, image设置为FULL

```
binlog_format    为 ROW
binlog_row_image 为 FULL
```

查询方式：
```
show global variables like 'binlog_format';
show global variables like 'binlog_row_image'；
```

设置方式：
```
set global binlog_format = "ROW" ;
set global binlog_row_image = "FULL" ;
```

备注： 如果是 MySQL 5.5 ，没有 binlog_row_image 这个变量，不需要设置

## 功能限制
### MyISAM 引擎表

UDTS 支持 MyISAM 引擎表的全量迁移及增量同步，但是有以下限制：
* 导出数据时会锁MyISAM表(读不受影响)，直到全表数据导出。可能会影响业务。
* 不支持一条事务中同时更新MyISAM引擎表和InnoDB引擎表。
* 对于损坏的表，在迁移之前要进行修复。

建议用户将MyISAM 引擎表转换为InnoDB引擎表以后再使用UDTS迁移。

### 数据量

建议单个全量迁移任务所迁移的数据量不超过200G， 最大支持500G。 如果所要迁移的数据量超过了500G，可以将任务拆分为多个任务进行迁移。 UDTS 提供按库、按表、按多库、按多表等多维度的迁移方式。 

如果迁移任务超过了500G， 且无法拆分为多个任务， 请联系技术支持。

### 其它限制
- 数据库名与表名中不能包含点(.)字符。
- 全量迁移阶段， 空数据库不迁移。
- 不迁移除 test 以外的内置数据库。
- 在转储过程中，如果待迁移的数据库上有执行 DDL 语句，UDTS 任务会失败，用户可以选择一个不会执行 DDL 语句的时间段重启任务。
- 全量和增量阶段暂时不支持 event 和 trigger。
- 增量同步开启之前，需要关闭 event.
        ```
        -- 停止所有 event

        SET GLOBAL event_scheduler = OFF;

        -- 如果按库迁移，停止指定库的 event 即可
        -- 查找对应的 event 并停止
        SHOW EVENTS;

        ALTER EVENT hello DISABLE;
        ```
- 如果源是阿里云 MySQL RDS, 所有表必须要有显式主键， 否则增量同步功能不支持。

## 注意事项
### 存储空间

迁移的过程中可能会在目标数据库产生slow log, 其存储于mysql数据库中。 所以迁移结束以后目标数据库占用的存储空间可能会大于源数据库占用的存储空间。 建议使用UDTS迁移数据库的过程中关闭目标数据库的slow log， 迁移结束以后重新开启 slow log。 在迁移较大的数据库时， 产生的slow log可能会很大， 如果迁移的过程中没有关闭slow log, 请预留好slow log的存储空间。 

如果迁移目标MySQL数据库开启了Binlog， 目标数据库产生的 Binlog 会占用存储空间。 当数据量较大时（超过200G）， 建议用户打开 NoBinlog 选项，这样在全量迁移的过程中在目标数据库不会产生Binlog，减少迁移对磁盘的额外需求。 也可加快全量迁移速度， 缩短进程。 如果在迁移的过程中一定要开启Binlog， 请为目标数据库创建较大的存储空间或者定时清理不需要的Binlog文件， 以免存储空间不够导致任务失败（根据经验，迁移3TB的数据会产生3TB以上的Binlog文件， 即源数据库存储空间为3TB， 目标需要6TB存储空间）。

如果任务失败，在调整好目标数据库配置之后，可以重新启动任务，任务会重新开始。
### 主从切换

如果源MySQL数据库是主从结构，当主从发生切换时，对UDTS产生的影响如下：
- 发生在全量迁移阶段，全量迁移失败，需要重启任务。
- 发生的增量同步阶段，如果使用GTID同步，则无影响；如果没有使用GTID同步， 则同步失败， 需要重新做全+增。

## MySQL 填写表单

数据源表单

| 参数名   | 说明                                                                                                                                                              |
|--------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 地址类型 | 支持内网地址，外网地址，专线地址三种方式。内网地址需要填写VPC和子网信息；外网地址支持IP和域名两种方式；专线地址既支持IP，也支持域名，如果使用域名需要用户网络有外网出口。 |
| 端口     | MySQL连接端口                                                                                                                                                     |
| 用户名   | MySQL连接用户名                                                                                                                                                   |
| 密码     | MySQL数据库对应用户密码                                                                                                                                           |
| 数据库名 | MySQL数据库名称。 所有库传输请填写 *； 指定一个数据库传输，请填写数据库名；指定多个数据库传输，依次输入多个数据库名，库名之间使用英文逗号隔开。|                                         |
| 表名     | MySQL传输表名。 只有当“数据库名”为指定一个数据库时有效。 若不填，默认为迁移指定库中的所有表； 指定一张表传输， 请填写表名； 指定多张表传输，依次输入多张表名，表名之间使用英文逗号隔开 |
| 最大速率    | 外网/专线的速率范围为 1-56 MB/s，默认28 MBps (即 224 Mbps); 内网的速率范围为 1-1024 MB/s，默认64 MBps(即512Mbps)         |
| Nolocks     | 默认关闭，对于无法获取super权限的友商RDS服务需要开启，UDB获取super权限详见FAQ                  |
| SSL 安全连接  | 默认关闭，当您需要使用证书连接数据库时，可以打开该选项，具体可以[参考](#ssl-安全连接)             |

传输目标表单

| 参数名       | 说明                                                                                  |
|--------------|-------------------------------------------------------------------------------------|
| 地址类型     | 目标暂时只支持内网                                                                    |
| 端口         | MySQL连接端口                                                                         |
| 用户名       | MySQL连接用户名                                                                       |
| 密码         | MySQL数据库对应用户密码                                                               |
| 最大速率     | 内网的速率范围为 1-1024 MB/s，默认64 MBps(即512Mbps)                                   |
| NoBinlog     | 当数据源和目标都是MySQL时，在全量阶段可以设置关闭目标端binlog文件的写入，默认不关闭     |
| SSL 安全连接 | 默认关闭，当您需要使用证书连接数据库时，可以打开该选项，具体可以[参考](#ssl-安全连接) |

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


目标限速配置参考值

| 数据库可用内存大小 (G) | 内网建议值 (MB/s) | 
| ------------------ | ---------- | 
| (0, 1)            | 1       |
| [1, 2)            | 2        | 
| [2, 4)           | 5        |
| [4, 6)           | 10       |
| [6, 8)           | 20       |
| [8, 10)          | 30       |
| [10, 20)         | 50       |
| [20, 40)         | 60       |
| [40, 60)         | 70       |
| [60, 80)         | 80       |
| [80, 100)        | 90       |
| >=100             | 100      |

### SSL 安全连接
为了提高备份链路的安全性，可以在备份过程中使用 SSL 证书连接数据库，SSL 在传输层对数据进行加密，提升通信数据的安全性，但会增加一定的网络连接响应时间。

UDTS 当前仅支持 pem 格式的证书，如果您使用的是其它格式的证书，可以先转换为 pem 格式，SSL 证书可以在源或者目标中设置。

如何使用:  
在源或者目标中，将 `SSL 安全连接` 选项打开，同时上传 ca 证书
![](http://udts-doc.cn-bj.ufileos.com/create-ssl.png)
