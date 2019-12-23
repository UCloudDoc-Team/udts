# 如何通过专线地址传输数据至UCloud

## 搭建专线

专线是将用户源端和希望的目标段原本内网互相无法通信的情况下，搭建专属线路用以保持内网通信。

可以通过罗马，VPC互通等功能保证两端端网络互通。

![](http://udts-doc.cn-bj.ufileos.com/connect001.png)

## 创建UDTS任务

地址类型选择专线地址

![](http://udts-doc.cn-bj.ufileos.com/connect002.png)

目标端填写打通的UCloud端内网地址和对应VPC

![](http://udts-doc.cn-bj.ufileos.com/connect003.png)

图中示例源端地址和目标端地址所处不同VPC，原本无法通信，通过VPC互通的功能联通两端。

## 运行UDTS任务

![](http://udts-doc.cn-bj.ufileos.com/connect004.png)
