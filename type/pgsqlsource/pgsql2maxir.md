

# PostgreSQL 迁移到MAXIR

UDTS 支持 PostgreSQL 作为数据传输源传输到MAXIR，支持版本9.4到16.x。

## 1. 前提条件

- 增量同步时用户需要开启数据日志，`wal_level`需要设置为`logical`, `max_replication_slots`需要大于1
- 待迁移的表需具备主键或着唯一索引，否则可能会产生重复数据。具体参考[增量同步处理表没有主键或者没有唯一索引的情况](#增量同步处理表没有主键或者没有唯一索引的情况)


## 2. 所需权限

| 类型      | 源库               | 目标库                                     |
| --------- | ------------------ | ------------------------------------------ |
| 全  量    | SELECT             | 待迁移库的 owner 权限，或 rolcreatedb 权限 |
| 全量+增量 | SELECT,REPLICATION | 待迁移库的 owner 权限，或 rolcreatedb 权限 |
| 全量+增量 | SELECT,REPLICATION，同步ddl需要super权限 | 待迁移库的 owner 权限，或 rolcreatedb 权限 |

## 3. 功能限制
- 源待迁移表必须要有主键。
- 一个“全量+增量”任务只能同步一个数据库，如果有多个数据库需要同步，则需要为每个数据库创建任务。
- 同步对象仅支持数据表,暂时不支持DDL语句的同步。如果需要同步DDL语句，需要手动在目标库中执行对应的DDL操作，然后重启数据同步任务。
- 增量同步不支持GEOMETRY、TSQUERY、TSVECTOR、TXID_SNAPSHOT类型的数据同步。
- 由于数据库自身的特性，不支持从高版本迁移到低版本。
- 增量阶段不支持处理generated列。

## 4. 注意事项

### 增量同步进度保存与复制槽 slot 说明
- 增量同步期间，UDTS会在目标库中创建`public.udts_pgsync_progress`的数据表，用于记录同步的进度等信息，同步过程中请勿删除，否则会导致任务异常.
- 增量同步期间，UDTS会在源库中创建前缀为`udts_`的`replication slot`用于复制数据。
- 客户暂停同步任务后，由于源库中存在前缀为`udts_`的`replication slot`用于复制数据, 会导致wal清理不掉持续占用磁盘，如果需要长时间停止任务建议删除此slot,并删除任务。
- 客户删除任务后，需要手动在源库删除对应 `slot`，操作步骤如下:

**操作步骤:**       
 
```

1. 例如使用UDTS增量迁移的数据库为 db_service_car。
2. 执行 select * from pg_replication_slots ，返回结果如下:


-------------------------------------------------------------------------------------------------------------------------------
| slot_name                               |   plugin        |   slot_type   |   datoid      |   database         |   active   |
-------------------------------------------------------------------------------------------------------------------------------
| udts_dd27ef9195294b49a5d424eda8f399f7   |   test_decoding |   logical     |   4839920     |   db_service_car   |   f        |
| udts_050a1f4b1cc84b4bb6460e5477ec3d79   |   test_decoding |   logical     |   2695822     |   db_service_xxx   |   t        |
-------------------------------------------------------------------------------------------------------------------------------


3. 找到 database 为 db_service_car 并且前缀为 udts 的 slot_name。
4. 执行 select pg_drop_replication_slot('udts_dd27ef9195294b49a5d424eda8f399f7');
5. 再执行一次 select * from pg_replication_slots, 确认已删除。

```
### 全+增任务开启DDL同步后， 任务于启动阶段，会在源库分别创建TABLE(udts_ddl_command), FUNCTION(udts_capture_ddl), EVENT TRIGGER(udts_intercept_ddl)，任务同步期间请不要在源库修改这些对象，否则会导致任务异常，任务迁移完成后可手动清理掉这些对象。


## 5. 迁移内容

| 迁移内容 | 说明                                                                                               |
| -------- | -------------------------------------------------------------------------------------------------- |
| 迁移结构 | DATABASE, TABLE 结构及数据                                                                         |
| 迁移范围 | 仅迁移创建任务时可以查到的库表， 任务运行中新增的表暂时不会自动迁移                                |
| DDL      | 开启后迁移 (CREATE TABLE,ALTER TABLE,DROP TABLE,CREATE SCHEMA,DROP SCHEMA,CREATE INDEX,DROP INDEX) |
| DML      | INSERT/UPDATE/DELETE                                                                               |

## 6.PostgreSQL 填写表单

### 数据源表单

| 参数名   | 说明                                                                                                                  |
| -------- | --------------------------------------------------------------------------------------------------------------------- |
| 地址类型 | 提供内网地址，外网地址，专线地址三种方式，内网地址需要填写VPC和子网信息，外网地址支持ip和域名两种                     |
| 端口     | PostgreSQL 连接端口                                                                                                   |
| 用户名   | PostgreSQL 连接用户名                                                                                                 |
| 密码     | PostgreSQL 数据库对应用户密码                                                                                         |
| 数据库名 | PostgreSQL 待迁移数据库名称                                                                                           |
| 表名     | PostgreSQL 传输表名，可选项。 若不填，整库迁移； 若填写，迁移指定的表. 具体参考 [表单表名填写规则](#表单表名填写规则) |
| 最大速率 | 外网/专线的速率范围为 1-256 MB/s，默认40 MBps (即 320 Mbps); 内网的速率范围为 1-1024 MB/s，默认 80 MBps(即 640 Mbps)  |
| 传输DDL | 控制增量个阶段是否同步DDL，默认关闭，打开需要源库的super权限 |

###  传输目标表单
  
| 参数名   | 说明                                  |
| -------- | ------------------------------------- |
| 地址类型 | 目标暂时只支持内网                    |
| 端口     | MAXIR 连接端口                        |
| 用户名   | MAXIR 连接用户名                      |
| 密码     | MAXIR 数据库对应用户密码              |
| 数据库名 | 迁移到MAXIR 目标库中的 Database名称。 |

### 表单表名填写规则

- 1. 表名仅支持[a-z][A-Z][0-9]以及_等字符。
- 2. 多个表名以,隔开
- 3. 表名没有添加schema的默认追加 `public.` schema。
- 4. 表名填写 `public.tb_test01,schema01.tb_test02`，表示只迁移 `public.tb_test01,schema01.tb_test02` 这两张表。


## 7. 数据类型映射
<table>
    <thead>
        <tr>
            <th>PostgreSQL数据类型</th>
            <th>MAXIR数据类型</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>bigint</td>
            <td>bigint</td>
        </tr>
        <tr>
            <td>integer</td>
            <td>integer</td>
        </tr>
        <tr>
            <td>smallint</td>
            <td>smallint</td>
        </tr>
        <tr>
            <td>date</td>
            <td>date</td>
        </tr>
        <tr>
            <td>double precision</td>
            <td>double precision</td>
        </tr>
        <tr>
            <td>real</td>
            <td>real</td>
        </tr>
        <tr>
            <td>boolean</td>
            <td>boolean</td>
        </tr>
        <tr>
            <td>text</td>
            <td>text</td>
        </tr>
        <tr>
            <td>character<br>character varying</td>
            <td>varchar</td>
        </tr>
        <tr>
            <td>numeric</td>
            <td>numeric/numeric(38,10)</td>
        </tr>
        <tr>
            <td>money<br>cidr<br>inet<br>macaddr<br>uuid<br>xml<br>json<br>jsonb<br></td>
            <td>text</td>
        </tr>
        <tr>
            <td>BIGINT</td>
            <td>INT8</td>
        </tr>
        <tr>
            <td>bit<br> bit varying<br> bytea<br> interval<br> tsquery<br> tsvector<br> txid_snapshot</td>
            <td>text</td>
        </tr>
        <tr>
            <td>data</td>
            <td>date</td>
        </tr>
        <tr>
            <td>time</td>
            <td>text</td>
        </tr>
        <tr>
            <td>timestamp</td>
            <td>timestamp</td>
        </tr>
        <tr>
            <td>timestamptz</td>
            <td>timestamptz</td>
        </tr>
    </tbody>
</table>
