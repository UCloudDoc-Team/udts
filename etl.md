# ETL设置

## 全量任务

以场景为示例

全量任务下

* 将满足 `id`>1 且 `id`<3的行，排除掉列名为`id`的列数据导入到目标库表
* 在目标数据库将db名映射为`mapped_db`, 表名映射为`mapped_table`，如果列名存在，则会继续做相关列名映射，列名为`name`的列映射为`mapped_name`，类型映射为varchar(100)

| 控件名 | 说明 |  示例值 |
| --- | --- | --- |
| ETL设置 | `ON` |  `ON` |
| 映射DB  | 迁移数据库在目标实例中映射的库名，为空时不映射 | `mapped_db` |
| 映射表 | 待迁移表在目标实例映射的表名，为空时不映射 | `mapped_table` |
| 条件 | 表的过滤条件，多个条件以逗号为间隔 | `id>1,id<3` |
| 列名 | 匹配的列名 | `id` |
| 列反选 | 反向过滤列名时为`ON`，否则为`OFF` | `ON` |
| 映射列 | 列映射结构，数据类型为JSON List | [{ </br>"ColumnName": "name",</br>"NewColumnName": "mapped_name",</br>"NewColumnType": "varchar(100)"</br> }] |

映射列key说明

| 参数名称 | 类型   | 是否必填 | 说明 | 示例 |
|----------|--------|----------|------|------|
| ColumnName  | string | 是       | 待迁移表中需迁移列名 | id     |
| NewColumnName | string | 是       | 待迁移列在目标实例中映射的列名  | no     |
| NewColumnType | string | 否       | 待迁移列在目标实例中映射的列类型 |   varchar(255)   |

## 增量任务

以场景为示例

增量任务下，排除 DB名为 `dtest`，表名为 `t_cal_`为开头的所有表时，设置的值分别为

| 控件名 | 说明 | 示例值 |
| --- | --- | --- |
| ETL设置 | `ON` |  `ON` |
| ETL数据库 | 需要匹配的库名 | `dtest` |
| ETL表名 | 需要匹配的表名，支持正则规则须满足语句必须以 `~` 开始，如 `~^t_test.*`其效果就是将table名匹配`t_test`开始的所有表 |  `~^t_cal_.*` |
| 表反选 | 排除 ETL表名时为`ON`，否则为`OFF` | `ON` |
