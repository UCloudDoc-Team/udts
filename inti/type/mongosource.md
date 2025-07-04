# MongoDB

UDTS 支持 MongoDB（自建/UDB MongoDB/其它云厂商提供的标准MongoDB服务）单节点、副本集、分片集之间的数据集成。

数据集成的基本要求与数据迁移相同，具体信息请参考[MongoDB数据迁移说明](/udts/type/mongonode)


## 功能限制

1. 不迁移系统内置库，config/local/admin
2. 增量迁移源库必须开启oplog
3. 源目数据库版本支持 4.2 至 6.0
4. 目标库不支持分片集群
5. 源库为分片集群时，不支持开启 DDL 同步

## 功能说明

### 执行顺序
- 数据源按添加显示顺序排序，前面的任务优先执行。
- “增量任务”与“全量+增量任务”均为并行执行。
- 当某一数据源的任务类型为全量任务时，只有该全量任务完成后才启动后续任务。 

### 是否保留原数据
数据源配置中提供【是否保留目标库的原数据】选择项，默认为“是”。 

- 如果保留默认“是”选择，新数据只会不断的添加到目标数据库中。 如果遇到冲突则按设定的冲突解决策略执行。
- 如果选择否，在任务开始之前会删除ETL设置中目标中对应的库表。

### 数据冲突解决策略
数据源配置中提供【数据冲突解决方式】选择项， 用于选择数据发生冲突时所采取的处理方式。

- 如果选择替换，则会使用 upsert 对存在的数据进行替换
- 如果选择保留，则保留原有数据，忽略新数据。

### 是否开启 DDL 同步
- 增量迁移时，可以根据需要选择是否开启 DDL同步功能。
- 当源库为分片集群时，不支持开启 DDL 同步。

### ETL
- 填写ETL信息时，如果勾选了数据库或者表，则必须填写映射名，如果不需要进行映射，默认填写与数据源相同的库名/表名
- 填写ETL信息时，如果勾选了库，未选择表，则迁移该库下的所有表