# MySQL 迁移到 PostgreSQL
UDTS 支持 从 MySQL 迁移到 MAXIR。
  MySQL 支持版本有 MySQL(包含Percona版) 5.5/5.6/5.7/8.x。
  PostgreSQL 支持版本 9.4到16.x。
 

## 1. 功能限制

### 1.1 源MySQL限制
1. 增量/全+增迁移时，源库需要开启binlog，且格式设置为ROW, image设置为FULL。

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

## 2. 迁移内容

  
| 迁移内容 | 说明                                                                |
| -------- | ------------------------------------------------------------------- |
| 迁移结构 | 1. Database、Table 结构及数据<br>  2. 迁移开始前不会清理目标库已存在的库表，如果自动建表不满足要求，客户可以提前在目标库手动建表 |
| 迁移范围 | 仅迁移创建任务时可以查到的库表， 任务运行中新增的表暂时不会自动迁移 |
| DDL      | 不支持                                                              |
| DML      | insert/update/delete                                                |


## 3. 表单填写

### 数据源表单
  
| 参数名   | 说明                                                                                                                                                                                   |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 地址类型 | 支持内网地址，外网地址，专线地址三种方式。内网地址需要填写VPC和子网信息；外网地址支持IP和域名两种方式；专线地址既支持IP，也支持域名，如果使用域名需要用户网络有外网出口。              |
| 端口     | MySQL连接端口                                                                                                                                                                          |
| 用户名   | MySQL连接用户名                                                                                                                                                                        |
| 密码     | MySQL数据库对应用户密码                                                                                                                                                                |
| 数据库名 | MySQL数据库名称。 仅支持单库迁移                                                                                                                                                       |  |
| 表名     | MySQL传输表名。 只有当“数据库名”为指定一个数据库时有效。 若不填，默认为迁移指定库中的所有表； 指定一张表传输， 请填写表名； 指定多张表传输，依次输入多张表名，表名之间使用英文逗号隔开 |
| 最大速率 | 外网/专线的速率范围为 1-56 MB/s; 内网的速率范围为 1-1024 MB/s                                                                                                                          |


###  传输目标表单
  
| 参数名   | 说明                                                                                                       |
| -------- | ---------------------------------------------------------------------------------------------------------- |
| 地址类型 | 目标暂时只支持内网                                                                                         |
| 端口     | PostgreSQL 连接端口                                                                                             |
| 用户名   | PostgreSQL 连接用户名                                                                                           |
| 密码     | PostgreSQL 数据库对应用户密码                                                                                   |



## 4. 权限要求
  
| 类别           | 权限说明                                                              |
| -------------- | --------------------------------------------------------------------- |
| 源MySQL        | SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT |
| 目标PostgreSQL | 必须拥有待迁移库的 owner 权限，或 rolcreatedb 权限                    |


## 5. 数据类型映射
  
<table>
    <thead>
        <tr>
            <th>MySQL数据类型</th>
            <th>PostgreSQL数据类型</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>BIT(1)<br>TINYINT(1)</td>
            <td>BOOL</td>
        </tr>
        <tr>
            <td>TINYINT</td>
            <td>INT2</td>
        </tr>
        <tr>
            <td>TINYINT UNSIGNED<br>SMALLINT</td>
            <td>INT2</td>
        </tr>
        <tr>
            <td>SMALLINT UNSIGNED<br>MEDIUMINT<br>MEDIUMINT UNSIGNED<br>INT<br>INTEGER<br>YEAR</td>
            <td>INT4</td>
        </tr>
        <tr>
            <td>INT UNSIGNED<br>INTEGER UNSIGNED<br>BIGINT</td>
            <td>INT8</td>
        </tr>
        <tr>
            <td>BIGINT UNSIGNED</td>
            <td>NUMERIC(20,0)</td>
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
            <td>CHAR<br>VARCHAR<br>TINYTEXT<br>MEDIUMTEXT<br>TEXT<br>LONGTEXT<br>ENUM<br>JSON<br>ENUM</td>
            <td>VARCHAR/TEXT</td>
        </tr>
        <tr>
            <td>DATE</td>
            <td>DATE</td>
        </tr>
        <tr>
            <td>TIME(s)</td>
            <td>TIME(s)</td>
        </tr>
        <tr>
            <td>DATETIME<br>TIMESTAMP(s)</td>
            <td>TIMESTAMP(s)</td>
        </tr>
        <tr>
            <td>BINARY<br>VARBINAR<br>BIT(p)<br>TINYBLOB<br>MEDIUMBLOB<br>BLOB<br>LONGBLOB <br>GEOMETRY</td>
            <td>BYTEA</td>
        </tr>
    </tbody>
</table>
