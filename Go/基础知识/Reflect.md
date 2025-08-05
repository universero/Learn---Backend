___

reflect包为Go提供了运行时反射的能力, 允许程序操作任意类型的对象.
但是要慎用, 反射的性能相对较低, 且代码更为脆弱
## reflect.Type

Type是一个接口, 表示一个Go的类型, 通过`reflect.TypeOf(x)`获取, 包含详细的类型信息: 方法, 字段, 种类等

相关方法
- Kind(): 获取底层类型
- NumField(): 获取字段个数
- Filed(n): 获取第n个字段
- NumMethod(): 获取方法数
- Method(n): 获取第n个方法
- Elem(): 获取容器类型(引用类型)的元素类型
- relfect.New(T): 创建对象
## reflect.Value

Value表示变量的值, 通过`reflect.ValueOf(x)`获取, 提供读写, 调用方法等能力
reflect.Value和interface{}都能承载任意类型的值, 但一个空的接口隐藏了值内部的表达方式和所有的方法, 必须要断言后才能当作对应的类型使用.

相关方法
- 类型()/Set类型(value): 读写对应类型的值
- reflect.ValueOf(&x).Elem(): 解引用指针, 可修改原变量
- Field(n): 获取第n个字段, 与Type类似
- 调用函数
	```go
	funcValue := reflect.ValueOf(myFunc)
	args := []reflect.Value{reflect.ValueOf(arg1)}
	results := funcValue.Call(args) // 调用函数，返回结果切片 []Value
	```
## reflect.Kind

Kind表示类型的底层分类, 如Int, Slice, Struct等, 通过`Type.Kind()`或`Value.Kind`获取
## reflect.Filed

Filed表示一个字段, 具备如下属性和方法

| ​**​属性/方法​**​  | ​**​说明​**​                                      |
| -------------- | ----------------------------------------------- |
| `Name`         | 字段名称（如 `"Name"`、`"Age"`）                        |
| `Type`         | 字段类型（`reflect.Type`，如 `int`、`string`）           |
| `Tag`          | 字段标签（`reflect.StructTag`，如 `` `json:"name"` ``） |
| `Index`        | 字段在结构体中的索引（`[]int`，用于嵌套结构体）                     |
| `Anonymous`    | 是否为匿名字段（如嵌入结构体）                                 |
| `IsExported()` | 是否是可导出字段（首字母大写）                                 |
| `Interface()`  | 获取字段的值（返回 `interface{}`）                        |
| `CanSet()`     | 是否可以修改该字段的值（需可寻址且可导出）                           |
## reflect.Method

Method表示一个方法, 具备如下属性和方法

|​**​属性/方法​**​|​**​说明​**​|
|---|---|
|`Name`|方法名称（如 `"Greet"`、`"SetName"`）|
|`Type`|方法类型（`reflect.Type`，描述输入/输出参数）|
|`Func`|方法的 `reflect.Value`，可用于调用|
|`PkgPath`|方法所属包的路径（非导出方法才有值）|
|`IsExported()`|是否是可导出方法（首字母大写）|