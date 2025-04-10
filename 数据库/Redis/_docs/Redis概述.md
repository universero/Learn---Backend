___

## Redis简介

Redis是一个内存数据结构存储，用作数据库、缓存、消息代理和流引擎。Redis提供了可进行原子操作的数据结构，例如字符串(string)、散列(hash)、列表(list)、集合(set)、带范围查询的排序集合(Zset)、位图(Bitmaps)、基数统计(HyperLog)、地理空间索引(GEO)和流(Steam)。Redis内置了复制、Lua脚本、LRU驱逐、事务和不同级别的磁盘持久性，并通过以下方式提供高可用的Redis Sentinel和Redis Cluster的自动分区。
Redis支持异步复制，具有快速非阻塞同步和自动重新连接以及网络拆分上的部分重新同步, 发布/订阅模式，内存淘汰机制，过期删除机制等

Redis的性能高，单机QPS是MySQL的十倍，单机QPS轻松破10w，而MySQL难破1w
## Redis 和 Memcached 的区别

- 共同点:
	- 都是基于内存的数据库，一般当作缓存使用
	- 都有过期策略
	- 都是高性能
- 区别点:
	- Redis的数据类型更加丰富，而Memcached只支持最简单的k-v数据类型
	- Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用，而Memcached没有持久化功能。
	- Redis原生支持集群模式，Memcached没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据
	- Redis支持发布/订阅模式、Lua脚本、事务等功能，memcached没有