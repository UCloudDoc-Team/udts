

# PostgreSQL

UDTS 支持 PostgreSQL 作为数据传输源/目标。

## 所需权限

|  类型    | 源库                   | 目标库                                                                                                      |
|-------| --------------------- | --------------------------------------------------------------------------------------------------------- |
|  全  量  | SELECT                | SELECT，INSERT，UPDATE，DELETE，TRUNCATE，REFERENCES，TRIGGER，CREATE，CONNECT，TEMPORARY，EXECUTE，USAGE     |
|  全+增 | SELECT,REPLICATION    | SELECT，INSERT，UPDATE，DELETE，TRUNCATE，REFERENCES，TRIGGER，CREATE，CONNECT，TEMPORARY，EXECUTE，USAGE |

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

## 增量同步相关

### 前提条件
- 暂时只支持源库为9.4至13.x的PostgreSQL进行增量迁移
- 用户需要开启数据日志，源库的账号需具备REPLICATION权限，并且`wal_level`需要是`logical`,`max_replication_slots`至少大于1
- 待迁移的表需具备主键或唯一键，否则可能会导致目标数据库中出现重复数据。
  
###  注意事项
- 一个数据同步任务只能同步一个数据库，如果有多个数据库需要同步，则需要为每个数据库创建数据同步任务。
- 同步对象仅支持数据表,暂时不支持DDL语句的同步，需要手动在目标库中执行对应的DDL操作，然后重启数据同步任务
- 在数据同步的过程中，如果要将源库中创建的新表作为同步对象，您需要对该表执行如下操作以保障该表数据同步的一致性。
  `ALTER TABLE schema.table REPLICA IDENTITY FULL;`
- 不支持BIT、VARBIT、GEOMETRY、ARRAY、UUID、TSQUERY、TSVECTOR、TXID_SNAPSHOT类型的数据同步。
- 增量同步的过程中UDTS会在目标库中创建`public.udts_pgsync_progress`的数据表，用于记录同步的进度等信息，同步过程中请勿删除，否则会导致任务异常.
- 同步期间，UDTS会在源库中创建前缀为`udts_`的`replication slot`用于复制数据。