# FAQ

{{indexmenu_n>4}}

#### 问: 现在支持多少区域间的跨域区域？

UDTS跨域迁移利用到了UDPN，现在部署的服务区域为北京，上海，新加坡。以此3个区域有UDPN传输线路的区域皆可支持。

#### 问: MySQL源的全量任务完成后为什么没有Binlog信息？

MySQL(包括UDB MySQL) Binlog信息返回需要用户数据源的账号具备权限：SELECT ，RELOAD ，LOCK TABLES ，REPLICATION CLIENT 权限

增量任务创建需要权限： SELECT ， REPLICATION SLAVE ， REPLICATION CLIENT 权限。

增量任务需要保证binlog 为row 模式。如果完成全量任务后再改为row模式，需要重新跑一次全量任务。

#### 问：UDB MySQL作为传输目标，源数据库有视图的情况下全量任务失败。

传输视图需要super权限。udb 默认回收了 super 权限，可使用命令：    

update mysql.user set super_priv = 'Y' where user = 'root';   

flush privileges;

恢复权限





