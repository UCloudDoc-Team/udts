# 基于全量任务创建增量任务

UDTS 对以下表格中所列出的全量传输任务， 提供基于已完成的全量任务快速创建增量任务的功能。 

|源类型| 目标类型|
| --- | --- |
| MySQL | MySQL |
| MySQL | TiDB |

## 步骤一 列出全量任务

在UDTS产品首页默认会列出所有任务， 从中找到已完成的 “全量任务” 

![](http://udts-doc.cn-bj.ufileos.com/connect004.png)

点击“详情”按钮进入下一步。

## 步骤二 查看任务详情

确认 sync 栏有 binlog 数据。 如果 sync 栏没有数据说明源数据库没有开启binlog, 无法创建增量任务进行数据同步。

![](http://udts-doc.cn-bj.ufileos.com/create006.png)

点击“创建增量任务”按钮进入下一步。

## 步骤三 创建增量任务

填写数据库密码，以及任务名称来完成创建。

![](http://udts-doc.cn-bj.ufileos.com/create007-2.png)





