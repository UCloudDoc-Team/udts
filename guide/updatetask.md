

# 修改任务

当用户出现任务信息填错等需要修改的时候，UDTS提供了修改任务的功能

## 修改UDTS迁移任务

用户无法对同步中的任务进行修改参数。目前开放的修改状态和类型为，全量任务/全量+增量任务 ，状态为“已创建”，“停止”，“失败”，“成功”的任务。

![](http://udts-doc.cn-bj.ufileos.com/update001.png)


进入任务表单对参数进行修改，可修改资源类型、任务名称、数据源/目标的信息等。

![](http://udts-doc.cn-bj.ufileos.com/update004.png)

被修改的任务状态，会初始化为 “已创建”，全+增任务修改后重新启动时，会从全量阶段开始。

备注：目前不提供修改 数据源目类型 和 传输任务类型。

## 更新任务名称和备注

在任务列表页面编辑UDTS迁移任务的名称和备注

![](http://udts-doc.cn-bj.ufileos.com/transfer/guide/transform_update_remark004.png)

名称只能包含中英文、数字以及`-_,.[:]`最大支持长度128字节；备注最大支持长度255字节

![](http://udts-doc.cn-bj.ufileos.com/transfer/guide/transform_update_remark005.png)
