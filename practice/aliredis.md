# 如何从阿里云 云数据库Redis 迁移至UCloud 云内存Redis

适用场景：通过UDTS将阿里云 云数据库Redis迁移至 云内存Redis上（暂不支持分布式Redis迁移）。

### 配置外网地址和白名单

确保UDTS可以从外部访问阿里云的数据库，如图：

![](http://antman-docs.cn-bj.ufileos.com/aliredis001.png)

外网白名单设置可咨询技术支持

### 增量同步需设置账号权限

此步骤适用于阿里云 Redis 4.0+版本，Reids 2.8版本或仅使用全量同步请忽略此步骤。

进入侧边栏 “账号管理”

![](http://antman-docs.cn-bj.ufileos.com/aliredis002.png)

创建一个有 “复制” 权限的账号。

![](http://antman-docs.cn-bj.ufileos.com/aliredis003.png)

### 创建UDTS任务

根据需要，选择 全量 或 全量+增量 任务

![](http://antman-docs.cn-bj.ufileos.com/aliredis004.png)

注：增量时 密码 需要填写具有 “复制” 权限的  账号:密码  来作为密码。

例如： 用户名 ucloud  密码 password  UDTS任务中 数据源 密码请填写 ucloud:password
