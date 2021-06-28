# MongoDB

UDTS 支持将 AWS Dynamodb 迁移到 MongoDB


## 源 Dynamodb 填写表单

| 参数名         | 说明                | 示例                                     |
|----------------|-------------------|------------------------------------------|
| 所在地域       | Dynamodb 所在地域   | us-east-2                                |
| AWS access key | AWS 授权 access key | AKIA5XXIWOJLFJSLW5DJNMFD                 |
| AWS secret key | AWS 授权 secret key | wlHcUhT/ysfl8lxaw7nx/UDcbwnB0t |
| AWS token      | AWS 授权 token      |                                          |
| 自建时 Dynamodb 地址|   自建 Dynamodb 的地址(参考备注1)    | http://192.168.100.103:8000 |
 
## 目标 MongoDB 填写表单

| 参数名            | 说明                 | 示例      |
|-----------------|----------------------|-----------|
| 地址              | 10.9.43.27           | us-east-2 |
| 所在地域/VPC/子网 | 在控制台页面选择即可 |           |
| 端口              | MongoDB连接端口      | 27017     |
| 用户名            | MongoDB 用户名       | root      |
| 密码              | MongoDB 密码         | 123456    |
| 授权DB           | MongoDB 的授权 db，udb 默认为 admin |  admin |
| 数据库名          | Dynamodb 迁移到 MongoDB 中对应的名称(具体请参考备注2) |  dynamodb |


## 备注说明
1. 当源为自建时， `自建时 Dynamodb 地址` 为必填，否则 `AWS access key` 和 `AWS secret key`，`所在地域` 为必填
2. 在 Dynamodb 中只有 table 的概念，没有 database 的概念，但迁移到 mongodb 中是有 database 的概念的，所以需要填写一个 db 名称，用于存储迁移后的数据。
