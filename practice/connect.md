# 自建IDC如何通过传输数据至UCloud

用户可能需要将自己私有云数据迁移至UCloud，但是不希望通过公网的手段，希望通过搭建专线的模式来进行传输。UDTS也提供专线地址的传输模式。

## 搭建专线

用户可以通过[罗马](https://docs.ucloud.cn/network/roma/overview)，或自建专线保证源目两端专线网络互通。

![](http://udts-doc.cn-bj.ufileos.com/connect005.png)

## 创建UDTS任务

地址类型选择专线地址

![](http://udts-doc.cn-bj.ufileos.com/speed001.png)

目标端填写打通的UCloud端内网地址和对应VPC

![](http://udts-doc.cn-bj.ufileos.com/connect003.png)

图中示例源端地址和目标端地址所处不同VPC，原本无法通信。通过产品功能实现专线通信。

如果对于专线的带宽有要求防止被UDTS的传输任务大量使用。可以对任务的传输进行限速，目前仅支持MySQL-MySQL的全量阶段。


## 运行UDTS任务

![](http://udts-doc.cn-bj.ufileos.com/connect004.png)
