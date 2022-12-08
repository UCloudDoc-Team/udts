

# PostgreSQL

UDTS 支持 PostgreSQL 作为数据传输源/目标，支持版本9.4到13.x。

## 前提条件

- 增量同步时用户需要开启数据日志，`wal_level`需要设置为`logical`, `max_replication_slots`需要大于1
- 待迁移的表需具备主键，否则无法做增量迁移。


### 所需权限

|  类型    | 源库                   | 目标库                                                                                                      |
|-------| --------------------- | --------------------------------------------------------------------------------------------------------- |
|  全  量  | SELECT                | SELECT，INSERT，UPDATE，DELETE，TRUNCATE，REFERENCES，TRIGGER，CREATE，CONNECT，TEMPORARY，EXECUTE，USAGE     |
|  全量+增量 | SELECT,REPLICATION    | SELECT，INSERT，UPDATE，DELETE，TRUNCATE，REFERENCES，TRIGGER，CREATE，CONNECT，TEMPORARY，EXECUTE，USAGE |

## 功能限制
- 一个“全量+增量”任务只能同步一个数据库，如果有多个数据库需要同步，则需要为每个数据库创建任务。
- 同步对象仅支持数据表,暂时不支持DDL语句的同步。如果需要同步DDL语句，需要手动在目标库中执行对应的DDL操作，然后重启数据同步任务。
- 增量同步不支持GEOMETRY、TSQUERY、TSVECTOR、TXID_SNAPSHOT类型的数据同步。
- 由于数据库自身的特性，不支持从高版本迁移到低版本。

## 注意事项
- 增量同步期间，UDTS会在目标库中创建`public.udts_pgsync_progress`的数据表，用于记录同步的进度等信息，同步过程中请勿删除，否则会导致任务异常.
- 增量同步期间，UDTS会在源库中创建前缀为`udts_`的`replication slot`用于复制数据。
- 客户暂停同步任务后，由于源库中存在前缀为`udts_`的`replication slot`用于复制数据, 会导致wal清理不掉持续占用磁盘，如果需要长时间停止任务建议删除此slot,并删除任务。
- 客户删除任务后，需要手动在源库删除对应 `slot`，操作步骤如下:
```
1. 例如使用UDTS增量迁移的数据库为 db_service_car。
2. 执行 select * from pg_replication_slots ，返回结果如下:
-------------------------------------------------------------------------------------------------------------------------------
| slot_name                               |   plugin        |   slot_type   |   datoid      |   database         |   active   |
-------------------------------------------------------------------------------------------------------------------------------
| udts_dd27ef9195294b49a5d424eda8f399f7   |   test_decoding |   logical     |   4839920     |   db_service_car   |   f        |
| udts_050a1f4b1cc84b4bb6460e5477ec3d79   |   test_decoding |   logical     |   2695822     |   db_service_xxx   |   t        |

3. 找到 database 为 db_service_car 并且前缀为 udts 的 slot_name。
4. 执行 select pg_drop_replication_slot('udts_dd27ef9195294b49a5d424eda8f399f7');
5. 再执行一次 select * from pg_replication_slots, 确认已删除。
```





## PostgreSQL 填写表单

数据源表单

| 参数名   | 说明                                                                                                                 |
| -------- | -------------------------------------------------------------------------------------------------------------------- |
| 地址类型 | 提供内网地址，外网地址，专线地址三种方式，内网地址需要填写VPC和子网信息，外网地址支持ip和域名两种                    |
| 端口     | PostgreSQL 连接端口                                                                                                  |
| 用户名   | PostgreSQL 连接用户名                                                                                                |
| 密码     | PostgreSQL 数据库对应用户密码                                                                                        |
| 数据库名 | PostgreSQL 数据库名称                                                                                                |
| 表名     | PostgreSQL 传输表名，可选项。 若不填，整库迁移； 若填写，只迁移指定的一张表                                          |
| 最大速率 | 外网/专线的速率范围为 1-256 MB/s，默认40 MBps (即 320 Mbps); 内网的速率范围为 1-1024 MB/s，默认 80 MBps(即 640 Mbps) |

传输目标表单

| 参数名   | 说明                                                    |
| -------- | ------------------------------------------------------- |
| 地址类型 | 目标暂时只支持内网                                      |
| 端口     | PostgreSQL连接端口                                      |
| 用户名   | PostgreSQL连接用户名                                    |
| 密码     | PostgreSQL数据库对应用户密码                            |
| 最大速率 | 内网的速率范围为 1-1024 MB/s，默认 80 MBps(即 640 Mbps) |
