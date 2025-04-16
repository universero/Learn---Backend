前缀和数组实现了求前缀和O(1), 修改一个元素O(n)
FenwickTree/BIT 实现了平均两个操作为O(logn)

树状数组通过​**​二进制低位技术​**​将前缀和分解为若干子区间的和。每个节点 `i` 维护的区间长度为 `i & -i`（即 `i` 的最低位1对应的值），覆盖范围是 `[i - lowbit(i) + 1, i]`。

每个数都可以被拆成二进制和, 如15 = 8+4+2+1, 只需要算每个区间的和


```go
type FenwickTree struct {
    tree []int
}

func fenwickTree(size int) *FenwickTree {
    return &FenwickTree{tree: make([]int, size+1)}
}

// 单点更新
func (ft *FenwickTree) update(index, delta int) {
    index++
    for index < len(ft.tree) {
        ft.tree[index] += delta
        index += index & -index  // 找到最低有效位的1
    }
}

// 前缀查询
func (ft *FenwickTree) query(index int) int {
    index++
    res := 0
    for index > 0 {
        res += ft.tree[index]
        index -= index & -index
    }
    return res
}
```
