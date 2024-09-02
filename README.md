
# 概览

* 产品简介
    * [什么是数据传输服务](/udts/introduction/concept)
    * [实例类型](/udts/introduction/instancetype)
    * [使用限制](/udts/introduction/limitation)
* 计费说明  
    * [计费指南](/udts/introduction/billing)
    * [回收与删除](/udts/billing/recycle)
* 数据传输
    * [支持数据类型](/udts/introduction/supporttype)
    * 数据类型说明
        * 源为MySQL
            * [MySQL迁移到MySQL](/udts/type/mysql_source/mysql2mysql)
            * [MySQL迁移到ClickHouse](/udts/type/mysql_source/mysql2clickhouse)
            * [MySQL迁移到Kafka](/udts/type/mysql_source/mysql2kafka)
        * 源为SQLServer
            * [SQL Server迁移到SQL Server](/udts/type/sqlserver_source/sqlserversource)   
            * [SQL Server迁移到Kafka](/udts/type/sqlserver_source/sqlserver2kafka)
        * [TiDB](/udts/type/tidb)
        * [CSV](/udts/type/csvsource)
        * [MongoDB](/udts/type/mongonode)
        * [US3](/udts/type/ufilesource)
        * [Redis](/udts/type/redissource)
        * [PostgreSQL](/udts/type/pgsqlsource)
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
    * [双向同步](/udts/synchronization)
* 数据集成
    * [支持数据类型](/udts/inti/introduction/supporttype)
    * 数据类型说明
        * [MySQL](/udts/inti/type/mysql)
        * [TiDB](/udts/inti/type/tidb)
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
    * [消息订阅](/udts/monitor/notice)
* [FAQ](/udts/faq)
* [预检查FAQ](udts/precheck_faq)
* 最佳实践
    * [如何从阿里云RDS MySQL迁移至UCloud UDB](/udts/practice/alitouclud)
    * [如何从阿里云 云数据库Redis 迁移至UCloud 云内存Redis](/udts/practice/aliredis)
    * [自建IDC如何传输数据至UCloud](/udts/practice/connect)
    * [托管云如何传输数据至公有云区域](/udts/practice/hybrid)
    * [跨VPC/跨项目/跨账号数据迁移](/udts/practice/diffvpc)
