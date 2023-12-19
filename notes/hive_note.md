# 1. 复习
#### 1. 组成
- 外部
	- 元数据默认存储在 `Derby` 中，通过配置更改为 `MySQL`
	- 数据存储在 `HDFS` 中
	- 计算引擎：`MR, Spark, Tez`
- 内部
	- 解析器： HQL 转换为 AST
	- 编译器： AST 转换为查询块 QB，进一步将 QB 转换为逻辑执行计划
	- 优化器：对逻辑执行计划进行参数级别的优化
	- 编译器：将逻辑执行计划转换为物理执行计划( TaskTree )
	- 优化器：针对不同计算引擎进行不同的优化

#### 2. 与 MySQL 对比

- 相同点：有类似的查询语言
- 不同点：
	- `MySQL` 是 `OLTP`, 侧重于 `CRUD`，而 `Hive` 是 `OLAP`，侧重于查
	- 数据量规模不同，`Hive` 支持很大规模的数据计算
	- 存储位置不同，`Hive` 存储在 `HDFS`，`MySQL` 将数据保存在文件系统中
	- `Hive` 不建议对数据进行修改
	- `Hive` 执行延迟较高，数据库执行延迟较低

#### 3. 内部表和外部表
- 删表时
	- 内部表删除元数据和数据本身
	- 外部表只删除元数据信息，数据本身还在
	- 可以相互转换
``` sql
alter table t set TBLPROPERTIES('EXTERNAL'='true/false')
```
- 绝大部分情况下使用外部表，测试使用的临时表可以创建内部表

#### 4. 四个 By

- order by
	- 默认升序
	- 全局排序，将所有数据集中在一个 reduce 中
	- 严格模式下，需要使用 limit
- sort by
	- 区内排序，在数据进入 Reducer 前完成排序，为每个 reducer 产生排序后的文件
- distribute by
	- 控制 Map 端如何拆分数据给 Reduce 端，类似 MR 中的分区器
	- hive 根据 distribute by 后面的列，将数据发给对应的 Reducer
	- 一般和 `sort by` 搭配使用
- cluster by
	- 当 `distribute by` 和 `sort by` 字段相同且为 `asc` 时，可以直接使用 `cluster by`

#### 5. 函数
##### 5.1 系统函数
- 时间类：
	- `date_add()`
	- `date_sub()`
	- `datediff()`
	- `date_format()`
	- `unix_timestamp` # time -> ts
	- `from_unixtime` # ts -> time
	- `next_day('', 'MO')` 下一个周 x (有可能是同一周)
	- `last_day` 当月最后一天
##### 5.2 窗口函数 over()
- 聚合函数
	- `sum, max, min, avg, count`
- 排名函数
	- `rank(1, 2, 2, 4), dense_rank(1, 2 ,2, 3), row_number(1, 2, 3, 4)`
- 跨行取值函数
	- `lead, lag`
	- `first_value, last_value`
	- `ntile(INTEGER x)`: 将有序分区分为 x 份并标号，适用于求前 x%的数据
- 开窗范围
	- `partition by a order by b`
	- `distribute by a sort by b` 作用同上，固定搭配
	- `rows between`:
		- `current row`
		- `n rows preceding`
		- `n rows following`
- eg:
```sql
+-----+                                               +--------------+
| num |  select sum(num) over(order by num) from t    |  sum_window  |
+-----+  ------------------------------------------>  +--------------+  
|  1  |   over 不指定窗口范围但是有 order by 时       |      1       |
|  2  |   默认范围是从开始到当前值                    |      3       |
|  3  |   order by <= 当前值的所有数据都会开一个窗口  |      9       |
|  3  |   因此两个 3 会放在同一个窗口里               |      9       |
|  4  |                                               |      13      |
|  5  |                                               |      18      |
+-----+                                               +--------------+
```
##### 5.3 多维分析函数
```sql
GROUP BY a, b, c WITH CUBE
-- 等价于
GROUP BY a, b, c GROUPING SETS ((a, b, c), (a, b),  ...)

GROUP BY a, b, c with ROLLUP -- 适用于层级维度(如：年月日)
-- 等价于
GROUP BY a, b, c GROUPING SETS ((a, b, c), (a, b), (a), ( ))
```
##### 5.4 自定义函数(按行算)
- UDF: 一进一出
- UDTF: 一进多出
- UDAF：多进多出

#### 6. 优化
##### 6.1 建表
- 分区：根据分区存储到不同的路径下
- 分桶：根据分桶的字段将数据存储到相应的文件中
- 文件格式：列存
- 压缩
