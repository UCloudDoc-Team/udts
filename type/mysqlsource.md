

# MySQL源

UDTS支持MySQL作为数据传输源

## MySQL源填写表单

| 参数名   | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| 地址类型       | 提供内网地址，外网地址，专线地址三种方式，内网地址需要填写VPC和子网信息，外网地址支持ip和域名两种 |
| 数据库名 | MySQL数据库名称，支持多库填写，使用"逗号"隔开。也可使用" * "除系统库外全库传输|                                         |
| 端口     | MySQL连接端口                                                |
| 用户名   | MySQL连接用户名                                              |
| 密码     | MySQL数据库对应用户密码                                      |
| 表名     | MySQL传输表名，若不填，系统默认为整库迁移                    |
| Nolocks     | 默认关闭，对于无法获取super权限的友商RDS服务需要开启，UDB获取super权限详见FAQ                  |
| ETL设置     | 用来描述将数据从来源端经过抽取、转换、加载 至目的端的功能,使用方法祥见ETL设置                    |
| 最大速率     | 不填写默认为峰值速率，外网/专线的速率限制为1-56 MB/s ，内网的速率限制为 1-1024 MB/s，源目皆可设置速率，仅在全量传输阶段有效                   |


备注：

如增量迁移，要求MySQL参数如下设置

binlog_format    为 ROW

binlog_row_image 为 FULL

查询方式：

show global variables like 'binlog_format';

show global variables like 'binlog_row_image'；

设置方式：

set global binlog_format = "ROW" ;

set global binlog_row_image = "FULL" ;
