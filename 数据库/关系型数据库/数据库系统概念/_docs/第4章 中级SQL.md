> [[数据库系统概念.pdf#page=112&selection=17,0,19,3|中级SQL]]

## 连接表达式

> [[数据库系统概念.pdf#page=112&selection=47,0,47,5|连接表达式]]


- 自然连接：natural join
	- 运算作用于两个关系，两个关系的所有同名属性相等的元组
	- 同名相等的属性会合并而不是重复出现

- 内连接
	- `from r join s using (A)` 指定连接两个表用的属性,通过这种方式设置的重复字段不会合并
	- 连接条件
		- on：在参与连接的关系上设置通用的谓词 如 `select * from r join s on r.id=s.id`
- 外连接
	- 通过创建包含空值的元素来保留连接中会丢失的元组(连接时不匹配on的谓词则会丢弃)
	- 左外连接：left outer join 只保留出现在左外连接运算符左边的关系中的元组
	- 右外连接：right outer join 只保留出现在右外连接运算符右边的关系中的元组
	- 全外连接：full outer join 保留出现在两个关系中的元组，是左外连接和右外连接的并集
	- on和where对外连接的表现是不同的，on可以认为是外连接的一部分而where是内连接的一部分。

## 视图

> [[数据库系统概念.pdf#page=119&selection=353,0,353,2|视图]]

- 视图：一个虚拟的关系，用于特定的查询
- 定义视图：`create view v(A) as <查询表达式>`
- 视图名可以直接作为常规的关系使用
- 每当查询视图时都会重新计算视图
- 物化视图
	- 存储视图关系，实际关系发生改变时视图也随着修改保持最新
- 视图更新
	- 对视图进行更新时，可能拒绝更新也可能翻译后操作对应的关系
	- 能更新的视图的要求
		- from子句只有一个数据库关系
		- 只包换关系的属性名，不包含表达式、聚集和distinct声明
		- 没有出现在select子句中的任何属性都可以取null值
		- 不包含group by 或 having

## 事务

> [[数据库系统概念.pdf#page=124&selection=218,0,218,2|事务]]

事务由查询或更新语句的序列组成，当一条sql语句被执行时就隐式的开始了一个事务，下列sql语句会结束事务
- commit work：提交当前事务，使事务执行的更新在数据库成为永久性的
- rollback work：回滚当前事务，回到执行第一条语句前的状态
- 保证了操作的原子性

## 完整性约束

> [[数据库系统概念.pdf#page=125&selection=299,1,300,5|完整性约束]]

- 单关系约束
	- A not null
		- 禁止插入空值，是域约束
	- unique(A)
		- A构成超键，不重复
		- 允许为null
	- check (P)
		- 要求元组满足谓词p
- 引用完整性
	- 外键：foreign key 确保了引用关系的完整性
	- 被引的属性必须是主键或者唯一约束
	- 如果对被引操作破坏了引用的完整性，默认是操作失败
	- 可以通过级联语句改变引用关系从而维护完整性约束，也可以用`set null`或`set default` 设置默认值
		 
	```mysql
	create table r (
	foreign key (A) references t
		on delete cascade <级联删除语句>
		on update cascade <级联更新语句>
	)
	```

	- 约束命名
		- 创建约束 A 类型, constraint 约束名 约束
		-  删除约束 alter table r drop constraint 约束名
	- 事务对完整性约束的违法
		- 事务可能包括多个步骤，中间步骤破坏了完整性约束，但后面有完善了
		- 此时需要用initially deferred子句，将完整性约束检查从中间步骤后置到事务结束前
		- 也可以指定一个属性的延迟检查，`set constraints 约束 deferred`
	- 断言
		- `create assertion 断言名 check p` 创建一个断言
		- 创建断言时，如果有效则对数据库的任何修改都需要满足断言
		- 断言会极大的增加开销

## SQL的数据类型与模式

>[[数据库系统概念.pdf#page=132&selection=9,0,10,8|SQL的数据类型与模式]]

 - 日期与时间
	 - date：年月日
	 - time：时分秒，time(p)指定秒的小数点后位数 time with timezone指定存储时区
	 - timestamp：date和time的结合
	 - extract(field from d)可以从中提取单独域，如year、seconde等
	 - current_date：获取当前时间
	 - 区间类型：x-y就是日期x到y间隔的天数
- 类型转换与格式化函数
	- `cast(e as t)`将e转换为类型t
	- `coalesce(A, v)` 如果属性为空则用v代替
- 缺省值
	- 通过default v指定默认值为v
- 大对象类型
	- clob：大字符对象
	- blob：大二进制对象
- 用户自定义类型
	- 独特类型
		- `create type 类型名 as 类型 final;`
		- 不同的类型名，相同的底层对象也认为不同
		- drop type 和 alter type来删除或修改类型
		- 不能声明约束或缺省值
	- 结构化数据类型
- 创建表的扩展
	-  创建与现有表结构相同的表：`create table r like s`
	- 从子查询创建表`create table r as (子查询) with date
* 模式、目录与环境
	* 数据库(目录)：体系结构的最顶层，每个目录都可以包含模式
	* 模式：定义关系、视图等对象
	* 唯一标识关系  目录名.模式名.模式名

## SQL中的索引

> [[数据库系统概念.pdf#page=139&selection=223,1,226,6|SQL中的索引定义]]

- 索引：一种数据结构，允许数据库系统高效的查询该属性特定值的元组。对于正确性来说不必需，是物理模式的一部分
	- create index <索引名> on <关系名> (<属性列表>)
	- drop index 删除索引

## 授权

> [[数据库系统概念.pdf#page=140&selection=289,0,289,2|授权]]

- 权限授予与收回
	- grant <权限列表> on <关系> to <用户> 授予权限
	- revoke <权限列表> on <关系> from <用户> 收回权限
- 角色
	- create role 角色名
	- 和权限一样的授予、删除方式
	- 将权限授予角色，将角色授予用户
- 引用授权
- references权限授予特定属性，允许用户创建外键
- 授予时加上with grant option子句允许将权限传递给其他用户
- 收回权限是级联的，restrict结尾可以防止级联收权
- 行级授权：一些数据允许元组级的细粒度权限控制