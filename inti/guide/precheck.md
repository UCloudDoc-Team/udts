# 预检查

为了避免任务启动时，因为密码错误、缺少权限、无法连通等问题，导致任务失败。UDTS提供预检查功能。

## 数据源预检查

![](http://udts-doc.cn-bj.ufileos.com/integration/precheck-src.png)

检查结果

![](http://udts-doc.cn-bj.ufileos.com/integration/pre-check-result.png)

## 目标预检查

![](http://udts-doc.cn-bj.ufileos.com/integration/precheck-tgt.png)

检查结果

![](http://udts-doc.cn-bj.ufileos.com/integration/precheck-tgt-result.png)

## 预检查失败
### 连通性检查失败
- 请检查端口密码是否正确。
- 如果有白名单，请检查白名单设置。
- 如果有防火墙， 请检查防火墙设置。

### 权限检查失败
- 出现权限错误，请对应[FAQ](https://docs.ucloud.cn/udts/faq?id=%e9%97%ae%ef%bc%9amysql-%e5%85%a8%e9%87%8f%e8%bf%81%e7%a7%bb%e9%9c%80%e8%a6%81%e6%bb%a1%e8%b6%b3%e5%93%aa%e4%ba%9b%e6%9d%a1%e4%bb%b6)，查询该任务类型所需权限。

### 配置检查失败
错误信息会提示出有问题的配置项， 按提示修正即可。


