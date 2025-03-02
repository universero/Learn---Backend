> [[数据库系统概念.pdf#page=152&selection=16,0,18,3|高级SQL]]

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

>[[数据库系统概念.pdf#page=163&selection=6,0,6,5|函数和过程]]

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

> [[数据库系统概念.pdf#page=168&selection=128,0,128,3|触发器]]

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

> [[数据库系统概念.pdf#page=173&selection=53,3,55,4|递归查询]]

