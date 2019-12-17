# 如何从阿里云RDS MySQL版本迁移至UCloud UDB

适用场景：通过UDTS将阿里云 RDS MySQL数据库迁移至UCloud UDB MySQL上。

## 确保RDS 已经配置完了外网IP和白名单，可以由阿里云外部进行访问

如图：

![](http://udts-doc.cn-bj.ufileos.com/ali001.png)

准备好UDB目标端资源

![](http://udts-doc.cn-bj.ufileos.com/udb001.png)

## 创建UDTS任务

![](http://udts-doc.cn-bj.ufileos.com/udtsali001.png)

由于阿里云RDS并非root账号，需要在UDTS任务中开启nolocks

## 启动UDTS任务

![](http://udts-doc.cn-bj.ufileos.com/rdsstart001.png)


