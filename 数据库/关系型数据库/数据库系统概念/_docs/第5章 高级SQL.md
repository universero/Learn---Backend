> [](数据库系统概念.pdf#page=152&selection=16,0,18,3|高级SQL)

## 使用程序设计语言访问SQL

- JDBC Java数据库连接
	- 连接到数据库
		- DriverManager.getConnection()打开一个连接，参数如下
		- 1字符串：连接的url与各种协议等
		- 2字符串：用户标识
		- 3字符串：密码
	- 传递SQL语句
		- Statement类
			- statement.executeUpdate(): 执行非查询语句
			- statement.executeQuery(): 执行查询
		- prepareStatement类
			- 通过？作为占位符代替一些值，在实际使用时提供
			- set类型(index,value)将?变成指定值
			- 通过转义以及检查可以减少SQL注入风险
	- 获取查询结果
		- ResultSet对象存储
	- 元数据特性
		- getMetaData: 包含结果集的元数据的ResultSetMeta对象
			-  getColumns获取列
			- getTables获取表
- ODBC 开发数据库连接
- 嵌入式SQL
	- EXEC SQL <嵌入式SQL语句> 可以将SQL请求替换为宿主语言的声明

## 函数和过程

>[](数据库系统概念.pdf#page=163&selection=6,0,6,5|函数和过程)

- 声明函数
	- 可能返回标准也可能返回表
	```MYSQL
	create function function_name([parameter1 type,...])
		returns return_type /*table则是返回一张表*/
		begin
		/*sql语句*/
		return 值
		end
```
- 声明存储过程
```mysql
create procedure proc_name(in parameter1 type, out parameter2 type, inout parameter3 type)
	begin
		/*sql语句*/
	end
```
	- in：待赋值
	- out：返回值
	- inout：赋值，会被改变
	- declare 声明变量
	- call：调用过程
	
```mysql
while 布尔表达式 do 
	语句;
end while

repeat
	语句
until 布尔表达式
end repeat

declare n integer default 0
for r as
	语句
do
	set n = 变化n
end for
/*leave 跳出循环，iterate 跳过当前*/

if 布尔表达式
	then 语句
elseif 布尔表达式
	then 语句
else 语句
end if

declare exit handler for 异常名
begin
	signal 异常名; /*引发异常*/
end
```

## 触发器

> [](数据库系统概念.pdf#page=168&selection=128,0,128,3|触发器)

作为对数据库修改的连带而由系统自动执行的一条语句，可以在操作前也可以在操作后触发
可以用于维护完整性约束

```mysql
create trigger 触发器名称 after 操作 /*如update 属性 of 表 或 delete on 表*/
referencing new row as nrow  /*存储新的行*/
referencing old row as orow  /*存储老的行*/
begin
/*过程语句*/
end
```

`alter trigger 触发器名称 disable` 禁用触发器

有其他方案时最好不要使用，可能出现无限触发链、降低性能等等

## 递归查询

> [](数据库系统概念.pdf#page=173&selection=53,3,55,4|递归查询)

使用迭代的传递闭包

- 通过repeat和util实现

SQL中的递归

- with recursive子句支持递归的受限形式
- 任何递归视图都被定义为两个子查询的并：
	- 非递归的基查询
	- 使用递归视图的递归查询
- 首先计算基查询，将所有结果元组添加到递归定义的视图关系中
- 然后用视图关系的当前内容计算递归查询，并把所有结果元组加回到视图关系中。
- 重复上述步骤直至没有新的元组添加到视图关系中，如此得到的视图关系实例被称为递归视图定义的不动点
- 不可使用递归查询的情况，会导致查询的非单调性 
	- 递归视图上的聚集
	- 在使用递归视图的子查询上的not exists运算
	- 右端使用递归视图的集差运算(execpt)

## 高级聚集特性

> [](数据库系统概念.pdf#page=176&selection=285,0,285,6|高级聚集特性)

- 排名
```mysql
rank() over (partition 属性A order by 属性B desc nulls last) as alias

-- 根据partition分组单独依据属性B排序rank，nulls last/first指定null在最前面还是最后面

percent_rank() -- 以分数形式给定排名，如果分区中有n个元组且该元组排名为r，在百分比排名为(r-1)/(n-1)

cume_dist() --  累计分布的简写，对于一个元组的定义是p/n，p是分区中排序值小于或等于元组排序值的元组数，n是分区中的元组数

row_number() -- 对行进行排列，并且按行在排序中所处位置给每行一个唯一的行号，相同排序值的不同行按照非确定的方式给到不同的行号

ntitle(n) -- 按照指定顺序取到每个分区中的元组，并分成n个具有相同元组数目的桶。然后对于每个元组，ntitle(n)给出它所在桶的编号
```

- 分窗
	窗口查询在一定范围内的元组上的聚集函数，可以简化对每一个窗口执行相同的操作
```mysql
avg(属性) over (partition by 属性 order by 属性 rows unbounded/n preceding/following)

-- 分区，排序，取出前n个或无边界，preceding前/follow后
```

- 旋转
	构造交叉表(数据透视表)，将属性的值作为列名
	交叉表是从一个关系派生出来的表，其中关系R的某个属性A的值成为结果中的属性名称；属性A是轴向属性
	```mysql
	select *
	from r
	pivot(
		sum(quantity)
		for A in ('val1','val2')
	)
	-- for子句指定一个轴向属性，该属性的值作为属性名出现在pivot的结果中，以及用于计算新属性值的聚集函数。结果中不包含计算的属性和轴向属性，但是会保留其他属性
```

- 上卷和立方体
	上卷(roll up)和立方体(cube)操作支持group by结构的泛化形式，允许在单个查询中运行多个group by查询并以单个关系的形式返回结果
	```mysql
	select 属性, 属性,...
	from R
	group by rollup(A,B)

	-- 等价于group by A,B ; group A; group ();的并集
	-- 为了将不同分组结果纳入同一个scheme，不存在的值用null表示

	group by cube (A,B);
	
	-- 等价于group by A,B ; group by A; group by B;

	-- 多个roll up或者cube语句时，类似于笛卡尔积，左右的都会完整做一遍然后拼接

	group by grouping sets((A,B),(B));

	-- 指定需要的分组方式
```