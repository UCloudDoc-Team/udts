## 功能原理

### 全量迁移

![](http://antman-docs.cn-bj.ufileos.com/udtstechmysql001.png)

UDTS在全球很多个地域部署了服务，会根据源目位置来就近选择运行位置来保证迁移效率。

### 单向增量同步

![](http://antman-docs.cn-bj.ufileos.com/udtstechmysql002.png)

UDTS 模拟一个Slave节点，通过Binlog从全量导出点开始获取增量数据。

### 双向增量同步

![](http://antman-docs.cn-bj.ufileos.com/udtstechmysql003.png)

与单向增量同步类似， 模拟Slave来获取增量数据。 同时UDTS对写下去的数据做标记，当有新的Binlog Event的时候， 会先检查是否有标记。 如果有标记则说明是循环数据，直接丢弃，如果没有标记则加上标记写到对端。

## 技术优势

- 以服务的形式运行， 用户无需考虑物理资源
- 支持多种类型的迁移
- MySQL到MySQL支持不同维度的迁移， 可以按表， 按库， 多库， 全库来迁移。
- MySQL到MySQL 支持 ETL
- MySQL到MySQL 支持单向增量同步， 也支持双向增量同步。 既支持通过GTID同步， 也支持通过binlog file/pos同步。 支持断点续传。
- 从AWS 迁移MySQL 可以从主库直接进行迁移，不需要搭从库
- 支持通过内网、外网、及专线迁移。
- 提供迁移限速功能。

## 传输速度

不同场景下，单个UDTS 任务的运行效率。 UDTS 可以并发运行， 同时提供灵活的迁移方法， 既可以按库迁移， 又可以按表迁移。 迁移速率一般受限于迁移带宽，或者源、目的数据库的性能。 通过多任务并发，一般场景都可以100% 利用迁移带宽。

| **源**             | **目的**           | **类型** | **带宽(Mbps)** | **数据量（GB）** | **速率（Mb/s）** |
| ------------------------ | ------------------------ | -------------- | ------------------------------ | ---------------------- | ------------------------------------------ |
| AWS RDS for MySQL南美    | UCloud 巴西 MySQL        | 全量导出       | 200                            | 13                     | 200                                        |
| AWS RDS for MySQL南美    | UCloud 巴西 MySQL        | 全量导入       | 200                            | 13                     | 40                                         |
| 新加坡 MySQL             | 越南MySQL                | 全量导出       | 800                            | 10                     | 800                                        |
| 新加坡MySQL              | 越南 MySQL               | 全量导入       | 800                            | 10                     | 80                                         |
| UCloud 华北（北京） UDB 8GB HA | UCloud 华北（北京） UDB 8GB HA | 全量导出       | 200                            | 19                     | 200                                        |
| UCloud 华北（北京） UDB 8GB HA | UCloud 华北（北京） UDB 8GB HA | 全量导入       | 200                            | 19                     | 64                                         |
| UCloud 华北（北京） UDB 8GB HA | UCloud 华北（北京） UDB 8GB HA | 全量导出       | 内网                           | 19                     | 2104                                       |
| UCloud 华北（北京） UDB 8GB HA | UCloud 华北（北京） UDB 8GB HA | 全量导入       | 内网                           | 19                     | 578                                        |


单向增量同步峰值支持2W OPS （insert, update, delete）
					
