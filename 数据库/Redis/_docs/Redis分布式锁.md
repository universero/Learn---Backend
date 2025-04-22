## 互斥

通过NX或SETNX实现锁不存在时设置
在 Redis 中， `SETNX` 命令是可以帮助我们实现互斥。`SETNX` 即 **SET** if **N**ot e**X**ists (对应 Java 中的 `setIfAbsent` 方法)，如果 key 不存在的话，才会设置 key 的值。如果 key 已经存在， `SETNX` 啥也不做。

```lua
> SETNX lockKey uniqueValue
(integer) 1
> SETNX lockKey uniqueValue
(integer) 0
```

释放锁的话，直接通过 `DEL` 命令删除对应的 key 即可。

```lua
> DEL lockKey
(integer) 1
```

为了防止误删到其他的锁, 建议使用 Lua 脚本通过 key 对应的 value（唯一值）来判断。

选用 Lua 脚本是为了保证解锁操作的原子性。因为 Redis 在执行 Lua 脚本时，可以以原子性的方式执行，从而保证了锁释放操作的原子性。

```lua
// 释放锁时，先比较锁对应的 value 值是否相等，避免锁的误释放
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

## TTL
通过EX设置过期时间, 如果不设置锁的过期时间, 可能出现锁一直无法释放
**设置key或过期时间需要是原子操作**, 不然可能出现锁无法释放

## 锁的续期

**如果操作共享资源的时间大于过期时间，就会出现锁提前过期的问题，进而导致分布式锁直接失效。如果锁的超时时间设置过长，又会影响到性能。**

以Redisson中的分布锁自动续期机制为例, 存在一个专门用来监控和续期锁的线程 **Watch Dog（ 看门狗）**，如果操作共享资源的线程还未执行完成的话，Watch Dog 会不断地延长锁的过期时间，进而保证锁不会因为超时而被释放。
![[Redisson锁续期.png]]

## 可重入锁

将锁关联一个计数器, 如整数类型, 然后根据线程的id就可以实现可重入锁

## 锁误解除

当线程A的锁过期后, 线程B获得了锁, 然后线程A执行完毕执行DEL, 导致了线程B的锁被释放, 也就导致了锁误解除.
此时需要每个线程一个都有一个唯一的id来标识, 避免锁的误删除

## RedLock

一种存在于Redis集群中的分布式锁, 相对于单机Redis来说更为复杂而且不一定更为好用
	