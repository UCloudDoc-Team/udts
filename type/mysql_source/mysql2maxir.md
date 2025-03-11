# MySQL 迁移到 MAXIR
UDTS 支持 从 MySQL 迁移到 MAXIR。
  MySQL 支持版本有 MySQL(包含Percona版) 5.5/5.6/5.7/8.x。
 

## 1. 功能限制

### 1.1 源MySQL限制

#### 1.1.1 增量/全+增迁移时，源库需要开启binlog，且格式设置为ROW, image设置为FULL。

```
查询方式:
show global variables like 'binlog_format';
show global variables like 'binlog_row_image';

设置方式：
set global binlog_format = "ROW" ;
set global binlog_row_image = "FULL" ;
```
#### 1.1.2 源待迁移表必须要有主键。
#### 1.1.3 源库不能存在同名但大小写不一致的库或表，否则同步可能会异常， 建议全部采用小写格式。
#### 1.1.4 源库不能有超过4G的binlog文件， 建议大小是1G, 否则同步会出错或失败。
#### 1.1.5 不支持 MySQL 8.0 的新特性 binlog 事务压缩 Transaction_payload_event。使用 binlog 事务压缩有导致上下游数据不一致的风险。

### 1.2 目标MAXIR限制

#### 1.2.1 MAXIR建表时需要指定DISTRIBUTED BY 和CLUSTER BY 列属性，例如:
   
```sql
CREATE TABLE trade (
    id bigint,
    "date" date,
    shopid int NOT NULL,
    sku varchar NOT NULL,
    price decimal(18, 4),
    primary key(date, id)
) DISTRIBUTED BY (id) CLUSTER BY (date, id);


a. CLUSTER BY 列建议需要跟primary key 相同，或者是pimary key的⼀部分，例如是 date 或者是 {date, id}
b. DISTRIBUTED BY 的列需要是primary key中⼀列，并且需要是 cardinality ⼤的列，在本例中只能是id，不能是date
```

#### 1.2.2 UDTS迁移到MAXIR，如果目标库中表不存在则会自动建表，建表规则如下，客户需要提前评估，如果不满足业务要求客户可以在目标库中提前建表:

```
1. CLUSTER BY 会和表的主键保持一致。
2. DISTRIBUTED BY 会使用主键， 如果是联合主键则会使用第一个字段。
3. 自动创建的表结构会去掉字段的 NOT NULL 属性。
```

## 2. 迁移内容

### 2.1 全量迁移
1. 全量迁移会迁移任务指定的源库中的库表结构和数据到目标库。
2. 表结构中索引仅迁移主键信息(MAXIR仅支持主键)。
3. 全量迁移会清理指定库表在目标库中的数据，但不会删除表结构，如果需要重建表结构需要客户自己在目标库删除或重建。


### 2.2 增量迁移
1. 增量迁移会实时解析源库binlog，将增量数据同步到目标库。
2. DML操作支持INSERT、UPDATE、DELETE。
3. DDL操作支持 1. CREATE/DROP DATABASE; 2. CREATE/DROP/RENAME TABLE; 3. ALTER TABLE ADD/DROP/MODIFY/CHANGE COLUMN。 ALTER TABLE时，不支持同时修改数据类型和字段名。
4. 增量迁移会将源库的binlog位点信息记录在UDTS的任务中，下次增量迁移时会从上次的binlog位点开始解析。

## 3. 表单填写

### 数据源表单
  
| 参数名   | 说明                                                                                                                                                                                      |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 地址类型 | 支持内网地址，外网地址，专线地址三种方式。内网地址需要填写VPC和子网信息；外网地址支持IP和域名两种方式；专线地址既支持IP，也支持域名，如果使用域名需要用户网络有外网出口。                 |
| 端口     | MySQL连接端口                                                                                                                                                                             |
| 用户名   | MySQL连接用户名                                                                                                                                                                           |
| 密码     | MySQL数据库对应用户密码                                                                                                                                                                   |
| 数据库名 | MySQL数据库名称。 所有库传输请填写 *； 指定一个数据库传输，请填写数据库名；指定多个数据库传输，依次输入多个数据库名，库名之间使用英文逗号隔开。(如果数据库名称中包含空格则无法做增量迁移) |  |
| 表名     | MySQL传输表名。 只有当“数据库名”为指定一个数据库时有效。 若不填，默认为迁移指定库中的所有表； 指定一张表传输， 请填写表名； 指定多张表传输，依次输入多张表名，表名之间使用英文逗号隔开    |
| 最大速率 | 外网/专线的速率范围为 1-56 MB/s; 内网的速率范围为 1-1024 MB/s                                                                                                                             |


###  传输目标表单
  
| 参数名   | 说明                                                                                                                                                                                                                                                             |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 地址类型 | 目标暂时只支持内网                                                                                                                                                                                                                                               |
| 端口     | MAXIR 连接端口                                                                                                                                                                                                                                                   |
| 用户名   | MAXIR 连接用户名                                                                                                                                                                                                                                                 |
| 密码     | MAXIR 数据库对应用户密码                                                                                                                                                                                                                                         |
| 数据库名 | 迁移到MAXIR 目标库中的 Database名称。 <br> 源MySQL的多个Database名称，会分别创建为目标MAXIR指定Database下的Schema。例如:<br>  `源MySQL迁移 A.t1,A.t2,B.t1,B.t2 到目标MAXIR的 target数据库下， 则在MAXIR中看到的迁移结果为：`<br> `target.A.t1,target.A.t2,target.B.t1,target.B.t2` |


## 4. 权限要求
  
| 类别      | 权限说明                                                              |
| --------- | --------------------------------------------------------------------- |
| 源MySQL   | SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT |
| 目标MAXIR | 必须拥有待迁移库的 owner 权限，或 rolcreatedb 权限                    |


## 5. 数据类型映射
  
<table>
    <thead>
        <tr>
            <th>MySQL数据类型</th>
            <th>MAXIR数据类型</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>BIT(1)</td>
            <td>BOOL</td>
        </tr>
        <tr>
            <td>TINYINT<br>SMALLINT</td>
            <td>INT2</td>
        </tr>
        <tr>
            <td>MEDIUMINT<br> INT<br> INTEGER<br> YEAR</td>
            <td>INT4</td>
        </tr>
        <tr>
            <td>TINYINT UNSIGNED<br>SMALLINT UNSIGNED<br>MEDIUMINT UNSIGNED<br>INT UNSIGNED<br>INTEGER UNSIGNED<br>BIGINT UNSIGNED</td>
            <td>NUMERIC(20, 0)</td>
        </tr>
        <tr>
            <td>BIGINT</td>
            <td>INT8</td>
        </tr>
        <tr>
            <td>DECIMAL(p, s) <br>DECIMAL(p, s) UNSIGNED <br>NUMERIC(p, s) <br>NUMERIC(p, s) UNSIGNED</td>
            <td>NUMERIC(p,s)</td>
        </tr>
        <tr>
            <td>FLOAT<br>FLOAT UNSIGNED</td>
            <td>FLOAT4</td>
        </tr>
        <tr>
            <td>DOUBLE<br>DOUBLE UNSIGNED<br>REAL<br>REAL UNSIGNED</td>
            <td>FLOAT8</td>
        </tr>
        <tr>
            <td>CHAR<br>VARCHAR</td>
            <td>VARCHAR</td>
        </tr>
        <tr>
            <td>TINYTEXT<br>MEDIUMTEXT<br>TEXT<br>LONGTEXT<br>ENUM<br>JSON</td>
            <td>TEXT</td>
        </tr>
        <tr>
            <td>DATE</td>
            <td>DATE</td>
        </tr>
        <tr>
            <td>TIME</td>
            <td>TIME</td>
        </tr>
        <tr>
            <td>DATETIME</td>
            <td>TIMESTAMP</td>
        </tr>
        <tr>
            <td>TIMESTAMP</td>
            <td>TIMESTAMPTZ</td>
        </tr>
        <tr>
            <td>BINARY<br>VARBINAR<br>BIT(p)<br>TINYBLOB<br>MEDIUMBLOB<br>BLOB<br>LONGBLOB <br>GEOMETRY</td>
            <td>BYTEA(但不支持写入数据)</td>
        </tr>
    </tbody>
</table>
