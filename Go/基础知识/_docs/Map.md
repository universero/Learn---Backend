___

Map是一个无序的key/value对的集合，其中所有的key都是不同的，然后通过给定的key可以在常数时间复杂度内检索、更新或删除对应的value, 可以写为`map[K]V`, 其中K和V分别对应key和value。map中所有的key都有相同的类型，所有的value也有着相同的类型，但是key和value之间可以是不同的数据类型。其中K对应的key必须是支持\==比较运算符的数据类型，所以map可以通过测试key是否相等来判断是否已经存在。

可以通过如下方式声明与初始化
```go
ages := make(map[string]int) // mapping from strings to ints

ages := map[string]int{
    "alice":   31,
    "charlie": 34,
}

ages := map[string]int{} // empty map
```

通过内置的方法`delete(map, key)`可以删除元素, 即使不存在也没有影响

访问元素`map[key]`时, 若不存在会得到类型零值
不能对map中的值并不是一个变量, 不能对其取地址. 禁止对map元素取址的原因是map可能随着元素数量的增长而重新分配更大的内存空间，从而可能导致之前的地址无效。

要想遍历map中全部的key/value对的话，可以使用range风格的for循环实现，和之前的slice遍历语法类似。下面的迭代语句将在每次迭代时设置name和age变量，它们对应下一个键/值对：
```go
for name, age := range ages {
    fmt.Printf("%s\t%d\n", name, age)
}
```
Map的迭代顺序是不确定的，并且不同的哈希函数实现可能导致不同的遍历顺序。