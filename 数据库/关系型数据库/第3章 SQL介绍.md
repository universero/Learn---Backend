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

