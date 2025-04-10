>[[数据库系统概念.pdf#page=72&selection=14,0,16,2|SQL介绍]]
***
## SQL数据定义

> [[数据库系统概念.pdf#page=73&selection=49,0,51,4|SQL数据定义]]

- 基本类型
	- char(n)：长度n的定长字符串
		- 长度不够会在后面加空格
	- varchar(n)：最大长度n的可变长字符串
	- int：整数，依赖于机器的整数
	- smallint：小整数
	- numeric(p,d)：指定精度的定点数，p位数字且小数点右边有d位数字
	- real，doubleprecision：浮点数与双精度浮点数
	- float(n)：精度至少为n位数字的浮点数

- 基本模式
```sql
create table r
	(A1 D1,
	A2 D2,
	...
	An Dn,
	<完整性约束1>,
	<完整性约束2>);
```
	- 完整性约束
		- 主键：primary key(A1,...) 罗列的属性构成主键，需要唯一且非空
		- 外键：foreign key(A1,...) references s 指定属性需要是s的主键
		- 非空：不允许包含空值
		- 默认值：default <表达式>
		- ...
	- drop table r：删除所有的元组和模式
	- delete table r：删除所有的元组
	- alter table r add A D：添加属性
	- alter table r drop A：删除属性
## SQL查询结构

> [[数据库系统概念.pdf#page=76&selection=264,0,266,7|SQL查询的基本结构]]

- select：查询结果中需要的属性
	- * 标识所有属性
- from：需要访问的关系列表
- where：作用在from子句中的关系的属性上的谓词
	- between：标识区间
	- 行构造器：(v1,v2)，构造一行简化比较运算符 
- order by：排序依据，默认升序，asc升序，desc降序

- 单关系查询
	- SELECT column_name FROM table_name WHERE 条件
	- 查找时可能会出现重复，如果要去重需要DISTINCT，显示不去重需要ALL
	- SELECT的列名可以运算，如SELECT a*2 FROM table得到列a的2倍

- 多关系查询
	- 查询多个关系时会构建笛卡尔积，where子句用谓词限制创建的集合只留下符合要求的

## 附加的基本运算

> [[数据库系统概念.pdf#page=82&selection=132,0,132,7|附加的基本运算]]

- 更名运算：old_name as new_name 能重命名属性或关系 
	- 新名称称为 相关名称、表别名、相关变量或元组变量

- 字符串运算：sql使用单引号标识字符串，区分大小写
	- 连接字符串 ||  , 提取子串 
	- 大小写转换 upper(s), lower(s)
	- 去除字符串后的空格trim(s)
	- 模式匹配 : like , not like不匹配
		- % 任意字符串
		- _ 任意一个字符串
		- escape定义转义字符

## 集合运算

> [[数据库系统概念.pdf#page=85&selection=228,0,228,4|集合运算]]

- 并运算
	- union：求并集，但自动去重
	- union all：求并集，保留重复项
- 交运算
	- intersect：求交集，但自动去重
	- intersect：求交集，保留重复项
- 差运算
	- except：求差集，自动去重
	- except all：求差集，保留重复项

## 空值

> [[数据库系统概念.pdf#page=88&selection=173,0,173,2|空值]]

如果算数表达式任一输入为空则结果为空
设计空值的比较运算的结果是为unknown
- and:
	- true and unknown = unknown
	- false and unknown = false
- or
	- true or unknown = true
	- false or unknown = unknown
- not
	- not unknown = unknown
- where子句的谓词对一个元组计算出false或unknown则不加入结果中
- 测试空值与未知: is null, is unknown 
- 对应distinct关键字，两个值都为空或者非空相等，则认为是相等的

## 聚集函数

> [[数据库系统概念.pdf#page=89&selection=222,0,222,4|聚集函数]]

聚集函数(aggregate function)是以值集为输入并返回单个值的函数
SQL提供五个标准聚集函数
- avg：平均值，输入必须是数字集
- min：最小值
- max：最大值
- sum：总和，输入必须是数字集
- count：计数

- 基本聚集
	- 在select子句中添加对应的标准聚集函数
- 分组聚集
	- group by 子句中给出的一个或多个属性用于构造分组，在子句中所有属性上取值相等的元组分在同一个组内
	- 确保出现在select语句中但是没有被聚集的属性只能是出现在group by子句中的属性
- having子句
	- 在分组后才应用having，因此可以在having子句中使用聚集函数
- 对空值和布尔值的聚集
	- 除了count (\*)之外的所有聚集函数都忽略其输入集合中的空值。
	- 规定空集的count运算为0，且作用在空集上时，其他聚集运算符返回空值
	- boolean类型可以取true、false和unknow三种类型，聚集函数some和every可应用与布尔值的集合并分别计算这些值的or和and
	- 

## 嵌套子查询

> [[数据库系统概念.pdf#page=94&selection=14,0,14,5|嵌套子查询]]

子查询是嵌套在where子句中查询语句

- 集合成员资格
	- 连接词in测试集合成员资格，如where A in (子查询)
- 集合比较
	- some：关系中的某个元组 如 where A > some (子查询)
	- all：关系中所有的元组 如 where A > all (子查询)
- 空关系测试
	- exists：测试子查询中是否存在元组
	- 外层查询的相关名称可以用到子查询中
- 重复元组存在性测试
	- unique：测试是否存在重复元组，没有重复返回true
- from 子句中的子查询
	- from中的子查询作为外层查询的数据来源，可以用as r(A1,A2)重命名
	- from子句嵌套的子查询不能使用来自同一from子句的其他关系的相关变量
	- lateral (子句)，修饰的from子句中的子查询可以访问同一个from子句中其他关系的相关变量
- with子句
	- 定义临时关系，用于简化查询语句
- 标量子查询
	- 只要子查询返回一个包含单个属性的元组，子查询可以出现在任何一个返回单个值表达式能出现的地方。
- 不带from子句的标量
	- 某些查询有包含from子句的子查询，外层查询不需要from，有些数据库中可以直接省略from，有序需要用预定义关系作为占位

## 数据库修改

> [[数据库系统概念.pdf#page=101&selection=256,0,256,6|数据库修改]]

- 删除
	- 只能删除元组，不能删除属性上某些值
	- delete from r wher p
- 插入
	- insert into r(A1,A2,A3) values (a1,a2,a2) 插入一个元组
	- 元组属性值排列与对应属性排列一致时可以省略属性
	- 可以插入查询语句生成的元组集合 insert into r (select子句)
- 更新
	- update r set A=a where p 改变某个元组的属性值
	- update set A=a where p (子查询) 改变一个元组集合的属性值
	- case语句、
	```sql
	CASE
		WHEN pred1 then result1
		...
		WHEN predn then resultn
		ESLE result0
	END
	```
