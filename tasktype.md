

# 全量任务和增量任务

UDTS支持全量任务和增量任务两种模式，使用源为MySQL或者 UDB MySQL的类型 可以支持增量任务模式（持续迭代）。


## 步骤一 完成全量迁移任务

创建任务是选择 “全量任务” 

数据源类型选择 “MySQL” “UDB-MySQL”

## 步骤二 查看任务详情

当完成全量任务后，查看任务详情，查看sync data数据。

![](http://udts-doc.cn-bj.ufileos.com/config002.png)

## 步骤三 选择 创建增量任务

![](http://udts-doc.cn-bj.ufileos.com/create006.png)

## 步骤四 填写剩余信息创建并启动任务

用户需再次填写数据库密码，sync data中的ServerID (保证和MySQL Server ID 不重复)，任务名称。

![](http://udts-doc.cn-bj.ufileos.com/create007.png)





