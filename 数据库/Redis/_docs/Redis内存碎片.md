内存碎片指的是空闲但无法利用的内存空间

## 产生原因

- Redis存储数据时申请的空间大于实际需要空间
	- Redis使用zmalloc方法分配内存时, 出来需要分配size大小的内存外, 还会多分配PREFIX_SIZE大小的内容
- 频繁修改Redis中的数据也会产生内存碎片
	- 当Redis中某个数据被删除时, Redis通常不会轻易释放内存给操作系统

## 查看碎片信息

info memory可以查看Redis内存相关的信息
内存碎片率 mem_fragmentation_ratio = 操作系统实际分配大小 used_memory_rss / Redis实际使用的大小 used_memory

一般内存碎片率大于1.5时, 需要清理内存碎片

## 清理内存碎片

通过命令开启自动碎片清理
config set activedefrag yes
触发条件由以下两个参数决定
```lua
# 内存碎片占用空间达到 500mb 的时候开始清理
config set active-defrag-ignore-bytes 500mb
# 内存碎片率大于 1.5 的时候开始清理
config set active-defrag-threshold-lower 50
```
限制内存清理占用资源可以减少对Redis性能的影响
```lua
# 内存碎片清理所占用 CPU 时间的比例不低于 20%
config set active-defrag-cycle-min 20
# 内存碎片清理所占用 CPU 时间的比例不高于 50%
config set active-defrag-cycle-max 50
```
