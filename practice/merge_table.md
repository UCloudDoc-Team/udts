# MySQL多源分库分表数据集成

适用场景：通过UDTS将多源分库分表MySQL集成到一个库表中

## 1 创建合并目标库表

创建数据集成任务前，需先在下游数据库创建合并目标库表。

来自多张分表的数据可能会引发主键或者唯一索引的数据冲突, 创建目标库表时可参考下一节主键或唯一索引冲突解决方案处理。

### 1.1 主键或唯一索引冲突解决

当合并表发生主键或唯一索引的数据冲突时，需要结合分表逻辑对每个主键或唯一索引进行检查。我们在此列举主键或唯一索引的三种情况：

1. 分片键：通常来讲，相同的分片键始终会划分到同一张分表之中，因此分片键不会产生数据冲突。
2. 自增主键：每个分表的自增主键会单独计数，因此会出现范围重叠的情况，这需要参照下一节自增主键冲突处理来解决。
3. 其他主键或唯一索引：需要根据业务逻辑判断。如果出现数据冲突，也可参照下一节自增主键冲突处理预先在下游创建表结构。

#### 1.1.1 自增主键冲突处理

推荐使用以下两种处理方式。

##### 1.1.1.1 去掉自增主键的主键属性
假设上游分表的表结构如下：

```sql
CREATE TABLE `tbl_no_pk` (
  `auto_pk_c1` bigint NOT NULL AUTO_INCREMENT,
  `uk_c2` bigint NOT NULL,
  `content_c3` text,
  PRIMARY KEY (`auto_pk_c1`),
  UNIQUE KEY `uk_c2` (`uk_c2`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
```

如果满足下列条件：

auto_pk_c1 列对业务无意义，且不依赖该列的 PRIMARY KEY 属性。  
uk_c2 列有 UNIQUE KEY 属性，且能保证在所有上游分表间全局唯一。  

则可以在开始执行全量数据迁移前，在下游数据库创建用于合表迁移的表，但将 auto_pk_c1 的 PRIMARY KEY 属性修改为普通索引。
```sql
CREATE TABLE `tbl_no_pk_2` (
  `auto_pk_c1` bigint NOT NULL DEFAULT 0,
  `uk_c2` bigint NOT NULL,
  `content_c3` text,
  INDEX (`auto_pk_c1`),
  UNIQUE KEY `uk_c2` (`uk_c2`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
```

##### 1.1.1.2 使用联合主键
假设上游分表的表结构如下：

```sql
CREATE TABLE `tbl_multi_pk` (
  `auto_pk_c1` bigint NOT NULL AUTO_INCREMENT,
  `uuid_c2` bigint NOT NULL,
  `content_c3` text,
  PRIMARY KEY (`auto_pk_c1`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
```
如果满足下列条件：

业务不依赖 auto_pk_c1 的 PRIMARY KEY 属性。  
auto_pk_c1 与 uuid_c2 的组合能确保全局唯一。  
业务能接受将 auto_pk_c1 与 uuid_c2 组成联合 PRIMARY KEY。  

则可以在开始执行全量数据迁移前，在下游数据库创建用于合表迁移的表，但不为 auto_pk_c1 指定 PRIMARY KEY 属性，而是将 auto_pk_c1 与 uuid_c2 一起组成 PRIMARY KEY。

```sql
CREATE TABLE `tbl_multi_pk_c2` (
  `auto_pk_c1` bigint NOT NULL DEFAULT 0,
  `uuid_c2` bigint NOT NULL,
  `content_c3` text,
  PRIMARY KEY (`auto_pk_c1`,`uuid_c2`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
```

## 2 创建UDTS任务

创建数据集成时，根据需要，选择 全量 或 全量+增量 任务
> 根据业务需求选择是否保留目标库数据  
> 数据冲突方式可根据业务需求选择替换或保留  

![](https://udts-doc.cn-bj.ufileos.com/integration/integration001.png)

编辑表映射，选择要合并库表
> 合并目标映射表名需保持一致  
> 选择指定库表迁移  
> 编辑待合并库表列表  

![](https://udts-doc.cn-bj.ufileos.com/integration/integration002.png)