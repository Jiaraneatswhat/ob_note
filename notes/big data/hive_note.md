# 1. 复习
#### 1.1 组成
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

#### 1.2 与 MySQL 对比

- 相同点：有类似的查询语言
- 不同点：
	- `MySQL` 是 `OLTP`, 侧重于 `CRUD`，而 `Hive` 是 `OLAP`，侧重于查
	- 数据量规模不同，`Hive` 支持很大规模的数据计算
	- 存储位置不同，`Hive` 存储在 `HDFS`，`MySQL` 将数据保存在文件系统中
	- `Hive` 不建议对数据进行修改
	- `Hive` 执行延迟较高，数据库执行延迟较低

#### 1.3 内部表和外部表
- 删表时
	- 内部表删除元数据和数据本身
	- 外部表只删除元数据信息，数据本身还在
	- 可以相互转换
``` sql
alter table t set TBLPROPERTIES('EXTERNAL'='true/false')
```
- 绝大部分情况下使用外部表，测试使用的临时表可以创建内部表

#### 1.4 四个 By

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

#### 1.5 函数
##### 1.5.1 系统函数
- 时间类：
	- `date_add()`
	- `date_sub()`
	- `datediff()`
	- `date_format()`
	- `unix_timestamp` # time -> ts
	- `from_unixtime` # ts -> time
	- `next_day('', 'MO')` 下一个周 x (有可能是同一周)
	- `last_day` 当月最后一天
##### 1.5.2 窗口函数 over()
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
##### 1.5.3 多维分析函数
```sql
GROUP BY a, b, c WITH CUBE
-- 等价于
GROUP BY a, b, c GROUPING SETS ((a, b, c), (a, b),  ...)

GROUP BY a, b, c with ROLLUP -- 适用于层级维度(如：年月日)
-- 等价于
GROUP BY a, b, c GROUPING SETS ((a, b, c), (a, b), (a), ( ))
```
##### 1.5.4 自定义函数(按行算)
- UDF: 一进一出
- UDTF: 一进多出
- UDAF：多进多出

#### 1.6 优化
##### 1.6.1 建表
- 分区：根据分区存储到不同的路径下，防止后续全表扫描
- 分桶：根据分桶的字段将数据存储到相应的文件中，对未知的复杂的数据进行提前采样
- 文件格式：列存(orc, parquet)
- 压缩
##### 1.6.2 写 SQL
- 单表：
	- 行列过滤(提前进行 where，不使用 select * )
	- 矢量计算：批量读取数据，默认 1024 条
	- map-side 聚合 `hive.map.aggr=true; 默认true`，预聚合，减少 `shuffle` 数据量
	- 某些没有依赖关系的 Stage 可以同时执行 `set hive.exec.parallel=true`
- 多表：
	- CBO (Cost Based Optimizer): 在 join 时计算成本，选择最佳的 join 方式，默认开启
	- 谓词下推
		- 尽量将过滤作前移，以减少后续计算的数据量
```sql
-- 过滤条件为关联条件，下推至两张表
select * from t1 join t2 on t1.id = t2.id where t1.id > 50
-- 过滤条件不是关联条件，只会下推到相关表
select * from t1 join t2 on t1.id = t2.id where t1.age > 50
-- left join 相同，无论什么join，只要是关联条件就会下推到两张表
select * from t1 left join t2 on t1.id = t2.id where t1.id > 50
select * from t1 left join t2 on t1.id = t2.id where t1.age > 50
-- 下推到两张表
select * from t1 left join t2 on t1.id = t2.id where t2.id > 50
-- 特殊情况，下推到t2表会导致结果发生变化
-- 2.x 谓词下推失效
-- 3.x 下推到相关表，将left join 变为 join
select * from t1 left join t2 on t1.id = t2.id where t2.age > 50
```
- 大表 join 小表(数据量 < 25M) -> map join
- 大表 join 大表 
	- SMB(Sort Merge Bucket) Map Join
	- 参与 join 的表是分桶表，分桶字段为 join 的关联字段
	- 分桶数有倍数关系，将相对小表分桶后尽量达到可以 merge 的条件，让每个桶小于 25M
- 整体：
	- 本地模式
	- Fetch 抓取(默认开启)，简单的 SQL 就不会执行 MR
	- 严格模式
	- 调整 Mapper 个数：
		- `splitSize = max(1, min(blockSize, LONG_MAX_VALUE))`
		- 增加 `Mapper` 个数需要减小 `splitSize`，`blockSize` 一般不做更改，减小 `LONG_MAX_VALUE`
		- 减少 `Mapper` 个数需要增大 `splitSize`，增大 1
	- 调整 Reducer 个数：
		- 默认 -1
		- $min(ceil(\frac{totalInputBytes}{bytesPerReducer}),maxReducers)$
		- `hive.exec.reducers.bytes.per.reducer` 默认 256M
		- `hive.exec.reducers.max` 默认 1009
##### 1.6.3 数据倾斜
- 现象：绝大部分 Task 已经完成，只有一个或少数几个没有完成，且执行很慢，甚至有 OOM
- 原因
	- 单表 `group by`
	- 多表 `join`
	 
	 ![[unoptimized_big2big_join.svg]]
	 
	- map 端文件不可切时，文件大小差距也可能造成数据倾斜
		- 解决方法：HDFS Sink 限制了单个文件的大小
- 解决
	- 单表
		- 不影响业务逻辑，可以进行一次预聚合
		- 否则，可以给分区字段添加随机数实现双重聚合，先对随机数拼接字段进行一次分组聚合，打散数据，再进行第二次聚合
	- 多表
		- 大表 join 小表 -> map join
		- 大表 join 大表
			- 不能使用 SMB Map Join，因为分桶后 join 时仍然是倾斜的
			- 相对大表加随机数打散，相对小表加随机数扩容
	
	![[optimized_big2big_join.svg]]
	
