

# MySQL双向同步

为满足用户多地域甚至跨云容灾等需求，UDTS推出了MySQL-MySQL双向同步的功能。

当前双向同步相当于是增量同步的模式，并不处理全量数据传输，适用于双库初始数据一致的情况。


## 创建双向同步任务

选择“双向同步”任务类型

填写双库信息（当前不支持2个数据库都是外网数据库的场景）

![](http://antman-docs.cn-bj.ufileos.com/createtype4.png)


## 注意事项

1 双库需要开启binlog, 格式为ROW, MODE 为FULL

2 库内的表必须都要有主键。

3 数据库需要开启GTID模式 ，但非必填。如果不填写会从当前位置开始同步。

4 双向同步的两个表如果是主键自增，需要两边设置对应的自增步进值，比如一边使用奇数 ID，一边使用偶数 ID，这样数据记录创建时，不会出现同样 ID 的数据，或者使用UUID/snowflake，来确保两表的ID不会重复。

5 用户不能在双库同时更新一条相同的数据。

6 目前双向同步版本仅支持MySQL5.6/5.7 ，且需要源库和目标库版本一致。