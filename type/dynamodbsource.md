# MongoDB

UDTS 支持将 AWS DynamoDB 迁移到 MongoDB


## 源 DynamoDB 填写表单

| 参数名         | 说明                | 示例                                     |
|----------------|-------------------|------------------------------------------|
| 所在地域       | DynamoDB 所在地域   | us-east-2                                |
| AWS access key | AWS 授权 access key | AKIA5XXIWOJLFJSLW5DJNMFD                 |
| AWS secret key | AWS 授权 secret key | wlHcUhT/ysfl8lxaw7nx/UDcbwnB0t |
| AWS token      | AWS 授权 token      |                                          |
| 自建时 DynamoDB 地址|   自建 DynamoDB 的地址(参考备注1)    | http://192.168.100.103:8000 |
 
## 目标 MongoDB 填写表单

| 参数名            | 说明                 | 示例      |
|-----------------|----------------------|-----------|
| 地址              | 10.9.43.27           | us-east-2 |
| 所在地域/VPC/子网 | 在控制台页面选择即可 |           |
| 端口              | MongoDB 连接端口      | 27017     |
| 用户名            | MongoDB 用户名       | root      |
| 密码              | MongoDB 密码         | 123456    |
| 授权DB            | MongoDB 的授权 DB，UDB 默认为 admin |  admin |
| 数据库名           | DynamoDB 迁移到 MongoDB 中对应的名称(具体请参考备注2) |  mydb |


## 备注说明
1. 当源为自建时， `自建时 DynamoDB 地址` 为必填，否则 `AWS access key` 和 `AWS secret key`，`所在地域` 为必填
2. 在 DynamoDB 中只有 Table，没有 Database，迁移到 MongoDB 时需要指定一个 DB 名，迁移后的数据会存储在该 DB 中。
