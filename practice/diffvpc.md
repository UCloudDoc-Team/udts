# 跨VPC/跨项目/跨账号数据迁移

当用户因为一些特殊需求，已经将不同的vpc，或者不同的项目，甚至不同的账号进行内网打通后，需要用到数据迁移的情况，也可以使用UDTS来完成。


## 创建UDTS任务

由于UDTS当前版本目标端仅支持内网的地址，会对当前账号项目下的资源进行校验，所以创建任务需要创建在目标库所在的账号，项目下。

即：如果用户要将A项目下的数据库，迁移至B项目的数据库。需要在B项目下创建任务。


地址类型选择专线地址（需要提前保证两边内网可通信），仅填写源端内网地址即可。

![](http://udts-doc.cn-bj.ufileos.com/speed001.png)

目标端填写公有云区域内网地址和对应VPC

![](http://udts-doc.cn-bj.ufileos.com/connect003.png)



## 运行UDTS任务

![](http://udts-doc.cn-bj.ufileos.com/connect004.png)
