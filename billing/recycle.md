# 回收与删除

## 通知渠道

所有通知消息将通过邮件、短信以及站内信的方式通知到您设置的通知接收人。

设置通知人：https://console.ucloud.cn/umon/contact

## 回收

### 预付费按年付/按月付

- 当您的资源过期前3天：给您设置的通知接收人发送资源即将过期预警；
- 当您的资源过期当天：给您设置的通知接收人发送资源已过期通知；
- 当您的资源过期后1天：给您设置的通知接收人发送UDTS任务即将被停止提醒，当天对UDTS任务执行停止操作，停止后的任务需先完成续费方可启动继续使用，因欠费停止的任务续费后会自动重启；
- 当您的资源过期后2天：给您设置的通知接收人发送UDTS任务即将被回收提醒，当天对UDTS任务执行回收操作（回收后的资源不可找回，还请及时续费）。

### 预付费按小时付
若您的账户可用余额充足，并开启了自动续费，将自动扣费。若您的账户可用余额不足以支持扣费，将产生欠费订单；

- 欠费订单产生时：发送UDTS任务可能被回收通知；
- 欠费订单+24H：发送UDTS任务停服通知，且当天停止任务。
- 欠费订单+48H：发送UDTS任务删除通知，且当天删除任务。

## 删除

如果您在资源有效期内删除资源，则系统将根据您的使用时长扣除费用，退还剩余的预付金额至您的账户，具体退费规则见[删除资源退费](https://docs.ucloud.cn/charge/refund)。
