# 自建IDC如何传输数据至UCloud

当用户需要将自建IDC数据迁移至UCloud，可以通过数据库的外网地址或者搭建专线进行传输。

## 搭建专线

如用专线地址，用户需要提前搭建自建IDC与UCloud之间的专线。

## 创建UDTS任务

地址类型选择专线地址/外网迁移请选择外网地址。

![](http://udts-doc.cn-bj.ufileos.com/speed001.png)

目标端填写打通的UCloud端内网地址和对应VPC

![](http://udts-doc.cn-bj.ufileos.com/connect003.png)

图中示例源端地址为自建IDC内网地址，和目标端地址所处UCloud内网，原本无法通信。通过专线实现内网传输。

如果对于专线的带宽有要求防止被UDTS的传输任务大量使用。可以对任务的传输进行限速，目前仅支持MySQL-MySQL的全量阶段。


## 运行UDTS任务

![](http://udts-doc.cn-bj.ufileos.com/connect004.png)
