

# 支持数据类型

| 数据源        | 数据目标      | 全量迁移 | 增量同步      |
| ------------- | ------------- | -------- | ------------- |
| MySQL         | MySQL         | 支持     | 支持          |
| MySQL         | TiDB          | 支持     | 支持          |
| MySQL         | PostgreSQL    | 支持     | 支持全量+增量 |
| MySQL         | Kafka         | 支持     | 支持全量+增量 |
| MySQL         | ClickHouse    | 支持     | 支持          |
| TiDB          | TiDB          | 支持     | 暂不支持      |
| TiDB          | MySQL         | 支持     | 暂不支持      |
| MongoDB       | MongoDB       | 支持     | 支持          |
| PostgreSQL    | PostgreSQL    | 支持     | 支持全量+增量 |
| CSV           | MySQL         | 支持     | N/A           |
| CSV           | UDW           | 支持     | N/A           |
| US3-bucket    | US3-Bucket    | 支持     | 暂不支持      |
| Redis         | Redis         | 支持     | 支持全量+增量 |
| Dynamodb      | MongoDB       | 支持     | 支持全量+增量 |
| ElasticSearch | ElasticSearch | 支持     | 暂不支持      |
| SQL Server    | SQL Server    | 支持     | 支持全量+增量 |
| SQL Server    | MySQL         | 支持     | 支持全量+增量 |
| SQL Server    | PostgreSQL    | 支持     | 支持全量+增量 |
| SQL Server    | Kafka         | 支持     | 支持全量+增量 |