

# PostgreSQL

UDTS支持PostgreSQL作为数据传输源/目标。

## 所需权限

| 源库/目标库   | 权限                                                         |
| -------- | ------------------------------------------------------------ |
| 源库       | select |
| 目标库 | SELECT，INSERT，UPDATE，DELETE，TRUNCATE，REFERENCES，TRIGGER，CREATE，CONNECT，TEMPORARY，EXECUTE，USAGE|

## PostgreSQL填写表单

| 参数名   | 说明                                                                                          |
|-------|---------------------------------------------------------------------------------------------|
| 地址类型 | 提供内网地址，外网地址，专线地址三种方式，内网地址需要填写VPC和子网信息，外网地址支持ip和域名两种 |
| 数据库名 | PostgreSQL数据库名称|                                         |
| 端口     | PostgreSQL连接端口                                                |
| 用户名   | PostgreSQL连接用户名                                              |
| 密码     | PostgreSQL数据库对应用户密码                                      |
| 表名     | PostgreSQL传输表名，可选项。 若不填，整库迁移； 若填写，只迁移指定的一张表  |
| 源最大速率    | 外网/专线的速率范围为 1-256 MB/s，默认40 MBps (即 320 Mbps); 内网的速率范围为 1-1024 MB/s，默认80 MBps(即640Mbps)          |
| 目标最大速率    | 内网的速率范围为 1-1024 MB/s，默认80 MBps(即640Mbps)        |
