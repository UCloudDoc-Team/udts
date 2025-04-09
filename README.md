
# 概览

* 产品简介
    * [什么是数据传输服务](/udts/introduction/concept)
    * [实例类型](/udts/introduction/instancetype)
    * [使用限制](/udts/introduction/limitation)
* 计费说明  
    * [计费指南](/udts/introduction/billing)
    * [回收与删除](/udts/billing/recycle)
* 公网迁移
    * [NAT网关实现公网迁移](/udts/inti/guide/nat_pub_dts) 
* 数据传输
    * [支持数据类型](/udts/introduction/supporttype)
    * 源为MySQL
        * [MySQL迁移到MySQL](/udts/type/mysql_source/mysql2mysql)
        * [MySQL迁移到TiDB](/udts/type/mysql_source/mysql2tidb)
        * [MySQL迁移到ClickHouse](/udts/type/mysql_source/mysql2clickhouse)
        * [MySQL迁移到Kafka](/udts/type/mysql_source/mysql2kafka)
        * [MySQL迁移到PostgreSQL](/udts/type/mysql_source/mysql2postgres)
        * [MySQL迁移到MAXIR](/udts/type/mysql_source/mysql2maxir)
        * [MySQL双向同步](/udts/type/mysql_source/mysql_synchronization)
    * 源为SQLServer
        * [SQL Server迁移到SQL Server](/udts/type/sqlserver_source/sqlserver2sqlserver)   
        * [SQL Server迁移到MySQL](/udts/type/sqlserver_source/sqlserver2mysql)
        * [SQL Server迁移到Kafka](/udts/type/sqlserver_source/sqlserver2kafka)
    * 源为TiDB
        * [TiDB](/udts/type/tidb)
    * 源为CSV
        * [CSV](/udts/type/csvsource)
    * 源为MongoDB
        * [MongoDB](/udts/type/mongonode)
    * 源为 AWS DynamoDB
        * [DynamoDB](/udts/type/dynamodbsource)
    * 源为Redis
        * [Redis](/udts/type/redissource)
    * 源为PostgresSQL
        * [PostgreSQL迁移到PostgreSQL](/udts/type/pgsqlsource/pgsqlsource)
        * [PostgreSQL迁移到MAXIR](/udts/type/pgsqlsource/pgsql2maxir)
    * 源为ElasticSearch
        * [ElasticSearch](/udts/type/essource)
    * 功能原理与传输速度
        * [MySQL迁移](/udts/tech/mysql)
    * 操作指南
        * [创建任务](/udts/guide/createtask)
        * [预检查](/udts/guide/checkconnection)
        * [启动任务](/udts/guide/starttask)
        * [获取任务详情](/udts/guide/getconfig)
        * [查看进度](/udts/guide/getprogress)
        * [查看运行历史](/udts/guide/gethistory)
        * [停止任务](/udts/guide/stoptask)
        * [修改任务](/udts/guide/updatetask)        
        * [基于全量任务创建增量任务](/udts/guide/quickIncremental)
        * [删除任务](/udts/guide/deletetask)
        * [数据校验](/udts/guide/validation)
* 数据集成
    * [支持数据类型](/udts/inti/introduction/supporttype)
    * 源为MySQL
        * [MySQL迁移到MySQL](/udts/inti/type/mysql_source/mysql2mysql)
        * [MySQL迁移到TiDB](/udts/inti/type/mysql_source/mysql2tidb)
    * 操作指南
        * [创建任务](/udts/inti/guide/createtask)
        * [预检查](/udts/inti/guide/precheck)
        * [启动任务](/udts/inti/guide/starttask)               
        * [获取任务详情](/udts/inti/guide/getconfig)
        * [查看进度](/udts/inti/guide/getprogress)
        * [停止任务](/udts/inti/guide/stoptask) 
        * [删除任务](/udts/inti/guide/deletetask)
* 监控告警
    * [监控指标](/udts/monitor/monitor)
    * [监控指标告警](/udts/monitor/alarm)
    * [任务失败告警](/udts/monitor/notice)
* [FAQ](/udts/faq)
* [预检查FAQ](udts/precheck_faq)
* 最佳实践
    * [如何从阿里云RDS MySQL迁移至UCloud UDB](/udts/practice/alitouclud)
    * [如何从阿里云 云数据库Redis 迁移至UCloud 云内存Redis](/udts/practice/aliredis)
    * [自建IDC如何传输数据至UCloud](/udts/practice/connect)
    * [托管云如何传输数据至公有云区域](/udts/practice/hybrid)
    * [跨VPC/跨项目/跨账号数据迁移](/udts/practice/diffvpc)
