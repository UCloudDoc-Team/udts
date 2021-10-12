# MongoDB

UDTS 支持 MongoDB（自建/UDB MongoDB/其它云厂商提供的标准MongoDB服务）单节点、副本集、分片集之间的相互迁移。 源目数据库版本支持3.x/4.x

## 功能限制

1. 不迁移系统内置库，config/local/admin
2. 增量仅支持迁移数据，不同步DDL

### 分片集群的迁移

1. 需要为每一个shard分片单独建立迁移任务，实现分片集群数据库的整体迁移
2. 目标为分片集时，迁移地址需要填写mongos路由地址
3. 在迁移前需要关闭源库Balancer
4. 清理源库中的孤儿文档，防止迁移时遇到id冲突的问题

## 注意事项

1. MongoOplogTs需填写UTC时间（与CST时间差8小时），例如：期望同步点为北京时间`2021-03-01 20:10:10`时，应填写`2021-03-01T12:10:10Z`
2. UDB MongoDB 填写示例，填写属性为 primary 的地址  
![mongo](http://udts-doc.cn-bj.ufileos.com/integration/mongodb/mongosrc.png)

## MongoDB填写表单

| 参数名   | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| 地址      | 提供内网地址，外网地址，专线地址三种方式，内网地址需要填写VPC和子网信息，外网地址支持ip和域名两种。副本集填写primary地址(UDB MongoDB 请参考备注 1)；分片集源库填写该分片的primary地址，分片集目标库填写mongos路由地址。 |
| 授权DB      |授权数据库名  ，UDB MongoDB默认为admin|
| Collection       | 集合， MongoDB 文档组 |
| 数据库名 | MongoDB数据库名称|                                         |
| 端口     | MongoDB连接端口                                                |
| 用户名   | MongoDB连接用户名                                              |
| 密码     | MongoDB数据库对应用户密码                                      |
| MongoOplogTs | （仅增量任务需填写）增量开始的oplog位置，即增量同步点，格式为`1970-01-01T00:00:00Z`（UTC time） |
