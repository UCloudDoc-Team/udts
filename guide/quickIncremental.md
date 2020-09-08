# 基于全量创建增量任务

UDTS 对以下表格中所列出的全量传输任务， 提供基于已完成的全量任务快速创建增量任务的功能。 

|源类型| 目标类型|
| --- | --- |
| MySQL | MySQL |
| MySQL | TiDB |

## 步骤一 在任务列表中找到已完成的全量迁移任务

创建任务是选择 “全量任务” 

数据源类型选择 “MySQL” 

## 步骤二 查看任务详情

查看任务详情，确认 sync 栏有 binlog 数据。 如果 sync 栏没有数据说明源数据库没有开启binlog, 无法创建增量任务进行数据同步。

![](http://udts-doc.cn-bj.ufileos.com/config002.png)

## 步骤三 选择 创建增量任务

![](http://udts-doc.cn-bj.ufileos.com/create006.png)

## 步骤四 填写剩余信息创建任务

用户需再次填写数据库密码，sync data中的ServerID (保证此ID唯一， 即不会与源MySQL及其从库 Server ID 重复)，任务名称。

![](http://udts-doc.cn-bj.ufileos.com/create007.png)





