

# MySQL源

UDTS支持MySQL作为数据传输源

## MySQL源填写表单

| 参数名   | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| IP       | 提供内网地址和外网地址两种方式传输，内网地址需要填写VPC和子网信息。 |
| 数据库名 | MySQL数据库名称，支持多库填写，使用"逗号"隔开。也可使用" * "除系统库外全库传输|                                         |
| 端口     | MySQL连接端口                                                |
| 用户名   | MySQL连接用户名                                              |
| 密码     | MySQL数据库对应用户密码                                      |
| 表名     | MySQL传输表名，若不填，系统默认为整库迁移                    |
| Nolocks     | 默认关闭，对于无法获取super权限的友商RDS服务需要开启，UDB获取super权限详见FAQ                  |
| ETL设置     | 用来描述将数据从来源端经过抽取、转换、加载 至目的端的功能,使用方法祥见ETL设置                    |

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
