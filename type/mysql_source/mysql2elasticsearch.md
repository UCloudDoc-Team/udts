# MySQL 迁移到 Elasticsearch（UDTS 工具）
UDTS 支持将 MySQL 数据迁移至 Elasticsearch，以下是完整的迁移配置指南、限制说明及操作规范。

## 一、支持版本
| 数据源（MySQL）                    | 目标端（Elasticsearch） |
| ---------------------------------- | ----------------------- |
| MySQL 5.6/5.7/8.x（含 Percona 版） | Elasticsearch 6.x/7.x   |

## 二、功能限制

### 2.1 源 MySQL 限制
1. **binlog 配置要求**（增量/全+增迁移必选）：
   - 必须开启 binlog，格式设置为 `ROW`，行镜像设置为 `FULL`
   - 查询命令：
     ```sql
     show global variables like 'binlog_format';
     show global variables like 'binlog_row_image';
     ```
   - 设置命令：
     ```sql
     set global binlog_format = "ROW";
     set global binlog_row_image = "FULL";
     ```
2. 待迁移表必须包含主键，否则任务直接报错
3. 库/表名限制：
   - 不得存在同名但大小写不一致的库或表（建议统一使用小写）
   - 不得包含特殊字符 `.`（会导致迁移失败）
   - 数据库名含 `$` 时，仅支持单库迁移
4. binlog 文件限制：
   - 单个文件大小不得超过 4G（建议设置为 1G）
   - 不支持 MySQL 8.0 的 binlog 事务压缩（Transaction_payload_event）
5. 迁移过程中限制：
   - 禁止修改表结构（字段数量、类型、顺序变更会导致数据不一致或任务失败）
   - 禁止在转储期间执行 DDL 语句（会导致任务失败）
6. 不迁移除 `test` 以外的内置数据库

### 2.2 目标 Elasticsearch 限制
1. 仅支持 6.x/7.x 版本
2. 迁移过程中会自动创建前缀为 `udts_`和`syncer_meta_schema_udts` 的系统索引，禁止手动删除（否则任务失败）

### 2.3 其他特殊限制
#### 2.3.1 MyISAM 引擎表限制
- 全量迁移时，MyISAM 表仅允许读操作，写操作会阻塞读操作直至转储完成
- 不支持同一事务中同时更新 MyISAM 和 InnoDB 表
- 损坏表需修复后再迁移
- 若目标库开启 GTID，可能触发 MySQL 1785 错误，解决方案：
  1. 转换引擎：`alter table 表名 ENGINE = InnoDB;`
  2. 关闭 GTID：
     ```sql
     set global gtid_mode = "ON_PERMISSIVE";
     set global gtid_mode = "OFF_PERMISSIVE";
     set global gtid_mode = "OFF";
     ```

#### 2.3.2 数据量与任务限制
- 单个全量迁移任务建议数据量≤200G，最大支持 1000G（超量需拆分任务或联系技术支持）
- 不支持 event 和 trigger（增量同步前需关闭 event）：
  ```sql
  -- 停止所有 event
  SET GLOBAL event_scheduler = OFF;
  -- 查找并停止指定 event
  SHOW EVENTS;
  ALTER EVENT 事件名 DISABLE;
  ```
- 增量迁移时，DDL 语句仅支持字符集：`ascii/latin1/binary/utf8/utf8mb4`

## 三、迁移内容说明
### 3.1 全量迁移
1. 迁移范围：指定库的表结构+全量数据（每张表对应一个 ES 索引）
2. 索引命名规则：`数据库名_表名`
3. 数据覆盖规则：
   - 默认删除目标端同名索引
   - 需保留原数据时，任务配置中选择「保留目标库原数据」
4. 强制要求：待迁移表必须包含主键

### 3.2 增量迁移
1. 同步机制：实时解析 MySQL binlog，同步增量数据
2. 支持操作：
   - DML：INSERT、UPDATE、DELETE
   - DDL：CREATE TABLE、DROP TABLE
3. 断点续传：binlog 位点信息记录在 UDTS 任务中，下次从上次位点继续解析

## 四、表单填写规范
### 4.1 数据源表单（MySQL）
| 参数名   | 说明                                                                              |
| -------- | --------------------------------------------------------------------------------- |
| 地址类型 | 支持内网（需填写 VPC+子网）、外网（IP/域名）、专线（IP/域名，域名需外网出口）     |
| 端口     | MySQL 连接端口（默认 3306）                                                       |
| 用户名   | MySQL 连接账号                                                                    |
| 密码     | 对应账号密码                                                                      |
| 数据库名 | 支持：`*`（所有库）、单个库名、多个库名（英文逗号分隔）；库名含空格不支持增量迁移 |
| 表名     | 仅指定单个数据库时有效；支持：空（所有表）、单个表名、多个表名（英文逗号分隔）    |
| 最大速率 | 外网/专线：1-56 MB/s；内网：1-1024 MB/s                                           |

### 4.2 传输目标表单（Elasticsearch）
| 参数名                 | 说明                                                                           |
| ---------------------- | ------------------------------------------------------------------------------ |
| 地址                   | ES 接入地址（HTTP 协议），多个地址用英文逗号分隔（示例：http://10.9.1.1:9200） |
| 用户名                 | ES 连接账号（无则留空）                                                        |
| 密码                   | 对应账号密码（无则留空）                                                       |
| 副本数                 | 自动创建索引的默认副本数（默认 1）                                             |
| 分片数                 | 自动创建索引的默认分片数（默认 1）                                             |
| 限速值                 | 内网：1-1024 MB/s                                                              |
| 是否保留目标库的原数据 | 默认「否」（删除同名索引）；选择「是」则保留原数据                             |

## 五、权限要求
| 迁移类型 | 源库权限（开启 NoLocks 模式）                                     | 源库权限（未开启 NoLocks 模式）                                                        |
| -------- | ----------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| 全量     | SELECT、REPLICATION SLAVE、REPLICATION CLIENT、SHOW VIEW、PROCESS | SELECT、REPLICATION SLAVE、REPLICATION CLIENT、SHOW VIEW、RELOAD、LOCK TABLES、PROCESS |
| 全+增    | SELECT、REPLICATION SLAVE、REPLICATION CLIENT、SHOW VIEW、PROCESS | SELECT、REPLICATION SLAVE、REPLICATION CLIENT、SHOW VIEW、RELOAD、LOCK TABLES、PROCESS |
> 说明：默认配置需源库账号拥有 SUPER 权限；无 SUPER 权限时，选择「NoLocks」模式


## 六、数据类型映射表
| MySQL 数据类型                                                                               | Elasticsearch 数据类型                        |
| -------------------------------------------------------------------------------------------- | --------------------------------------------- |
| BIT(1)                                                                                       | BOOLEAN                                       |
| TINYINT、SMALLINT、MEDIUMINT、INT                                                            | INTEGER                                       |
| TINYINT UNSIGNED、SMALLINT UNSIGNED、MEDIUMINT UNSIGNED、INT UNSIGNED                        | INTEGER                                       |
| BIGINT、BIGINT UNSIGNED                                                                      | LONG                                          |
| DECIMAL(p, s)                                                                                | TEXT                                          |
| FLOAT                                                                                        | FLOAT                                         |
| DOUBLE                                                                                       | DOUBLE                                        |
| CHAR                                                                                         | KEYWORD                                       |
| VARCHAR                                                                                      | 长度≤255：KEYWORD；长度>255：TEXT             |
| TINYTEXT、MEDIUMTEXT、TEXT、LONGTEXT、JSON                                                   | TEXT                                          |
| ENUM                                                                                         | KEYWORD                                       |
| DATETIME                                                                                     | DATE（格式：yyyy-MM-dd'T'HH:mm:ss，UTC 时间） |
| TIMESTAMP、DATE、TIME、YEAR                                                                  | DATE（格式：yyyy-MM-dd HH:mm:ss）             |
| BINARY、VARBINARY、BIT(p)、TINYBLOB、MEDIUMBLOB、BLOB、LONGBLOB                              | KEYWORD（ES 6.x）/ BINARY（ES 6.x 以上）      |
| POINT                                                                                        | GEO_POINT（WKT 格式）                         |
| GEOMETRY、LINESTRING、POLYGON、MULTIPOINT、MULTILINESTRING、MULTIPOLYGON、GEOMETRYCOLLECTION | GEO_SHAPE（WKT 格式）                         |