>实验要求 [Lab 5: Sharded Key/Value Service](https://pdos.csail.mit.edu/6.824/labs/lab-shard1.html)
___

在这个实验中, 我们将构建一个kv 存储系统, 系统将键分片或分区到一组Raft复制的键/值服务器组(shardgrp)上. 分片是键/值对的子集; 例如, 所有以"a"开头的键可能属于一个分片, 所有以"b"开头的键属于另一个分片. 进行分片的原因可能是性能考虑, 每个分片组只处理一个分片的Put和Get, 并且这些分片组并行运行, 因此系统总吞吐量随着分片数量的增加而增加
![[SharedKV.png]]
sharded KV service包含上图所示的所有组件, shardgrp存储Key的shard, clients通过实现了Put和Get的clerk和service交互. Clerk从kvsrv获取配置来找到对应的Key, 配置描述了从shards到shardgreps的映射. 

Administrator使用另外的客户端, 即controller, 在集群中添加或移除shardgrps, 并设置哪个shardgrp对应哪些shard. Controller有一个主要的方法: ChangeConfigTo, 该方法接收新的配置作为参数, 并将系统从当前配置更新为新配置; 这包括将分片移动到正在加入系统的新shardgrep, 以及将分片从正被移出系统的shardgrep移除. 为实现上述功能, Shard发起RPC 以及更新存储在kvsrv中的配置

controller存在的意义是, 分片存储系统必须能在分片组之间移动分片. 其中一个原因是, 某些分片组的负载可能比其他的分片组更高, 因此需要移动分片以平衡负载. 另一个原因是, 分片组可能会加入和离开系统: 可能会添加新的分片组以增加容量, 或者现有的分片组因维修或停用而下线.

本实验最大的挑战就是, 在处理一下任务时, 保证Get/Put的线性化操作
- 更改分片到分片组的分配
- 从故障中或者在ChangeConfigTo期间分区中恢复controller

ChangeConfigTo 将分片从一个 shardgrp 到另一个。风险在于某些客户端可能使用旧的 shardgrp，而其他客户端使用新的 shardgrp，这可能会破坏线性一致性。你需要确保任何时候最多只有一个 shardgrp 正在提供服务。
如果 ChangeConfigTo 在重新配置时失败，某些分片可能已开始但尚未完成从一个 shardgrp 移动到另一个。为了继续推进， tester会启动一个新的controller，你的工作是确保新的controller完成重新配置前一个controller启动的配置

Note
- 本实验使用“配置”来指代将分片分配到分片组。这与 Raft 集群成员资格变更不同。无需实现 Raft 集群成员资格变更。
- 一个 shardgrp server只能属于一个 shardgrp。给定 shardgrp 中的服务器集合永远不会改变。
- 客户端和服务器之间的交互只能使用 RPC。例如，服务器的不同实例不允许共享 Go 变量或文件。

本实验的分配KV数据库遵从和Flat DataCenter、BigTable、Spanner、FAWN、Apache Hbase、Rosebud、Spinnaker等众多相同相同的整体设计. 但这些系统在细节上与本实验有所不同, 且通常更加强大和复杂, 例如本实验不会变化每个Raft组中的peer, 数据和查询模型都非常简单等.

Note
- Lab5将会使用Lab2中的kvsrv和Lab4中的rsm和Raft, Lab4和Lab5都必须使用相同的rsm和Raft实现
- You may use late hours for Part A, but you may not use late hours for Parts B-D.
## Getting Started

代码框架和测试代码都在shardkv1中:
- client.go: shardkv clerk
- shardcfg: 计算shard配置
- shaardgrp: shardgrp的clerk和server
- shardctrler:  包括配置变更和配置查询
## PartA: Moving shards

第一个任务是实现无错情况下的shardgrps和InitConfig、Query和ChangeConfigTo. 我们已经在`shardkv1/shardcfg`中提供了描述配置的代码. 每个shardcfg.ShardConfig都有一个唯一标识号、一个从shard号到group号的映射, 以及一个从group号到复制这个组的server列表的映射. 通常shards会比group多, 以便能够更细粒度的负载转移.

Task1: 在`shardctrler/shardctrler.go`实现这个两个方法, 并通过TestinitQuery5A
- `InitConfig`方法接收tester以shardcfg.ShardConfig形式传递的第一个配置. InitConfig应该将配置存储在lab2的kvsrv实例中
- Query方法应该返回当前的Configuration, 应该从kvsrv中读取
- 通过存储和读取kvsrv的初始配置实现InitConfig和Query: 使用Put和Get方法与kvsrv交互, 使用ShardConfig的String方法将ShardConfig转换为可传递给Put的字符串, 并且使用shardcfg.FromString()将字符串转换为ShardConfig
Task2:
- 复制Lab4中的代码到`shardkv1/shardgrp/server.go`中实现shardgrp的初始版本和对应的在`shardkv1/shardgrp/client.go`中的Clerk.
- 通过Static测试
- shardkv1/client.go为整个系统提供了Put和Get: 他通过执行Query方法找出哪个shardgrp持有Key所需的分片, 然后和这个shardgrp交互
- 实现`shardkv1/client.go`, 包括他的Put和Get方法, 使用`shardcfg.Key2Shard`查找键的分片编号. tester将ShardCtrler对象传递给client的MakeClerk方法. 使用Query方法获取当前的配置.
- 为了从shardgrpPut/Get一个key, shardkv clerk应该通过调用shardgrp.MakeClerk为这个shardgrp创建一个shardgrp clerk, 将Configuration中找到的servers和shardkv clerk's的ck.clnt作为参数传入. 使用ShardConfig中的GidServers()获取一个shard的组
- 当这个Put调用shardgrp的Put但响应可能丢失时, client.go的Put必须返回ErrMaybe. 内部的Put可以通过error标识这个的发生
- 创建后, 第一个shardgrp(shardcfg.Gid1)应该初始化自己拥有的所有的shard

现在你应该实现`ChangeConfigTo`方法来支持在组之间移动shard, 以从旧配置转变为新配置. 新的配置需要包括不在旧配置中的新的shardgrps, 也需要排除出现在旧配置里的shardgrps. controller应该移动shard以便每个shardgrp根据配置存储对应的shard

我们建议移动分片的方式是 `ChangeConfigTo`先冻结原shardgrp, 然后shardgrp拒绝key在移动中shard的Put的请求. 然后, 将shard复制到目标shardgrp. 然后删除冻结的shardgrp. 最后发布新的配置, 使得client可以找到被移动的shard. 这种方法的一个优点是, 避免分片组之间任何直接的交互, 而且支持继续提供不受配置变更影响的shard

为了是配置变更有序, 所有的配置都有一个唯一的编号Num, PartA中的tester会顺序地调用changeConfigTo, 并且配转递到ChangeConfigTo的配置都会有一个递增的Num

网络可能会延迟RPC, 并且RPC可能会无序到达shardgrps. 为了拒绝旧的FreezeShard, InstallShard和DeleteShard请求, 他们需要包含Num, 并且shardgrps必须保存每个shard已收到的最大的Num

Task3: 实现ChangeConfigTo和扩展shardgrp以支持freeze, install和delete. ChangeConfigTo在PartA中不会失败,. 你需要使用shardrpc库实现shardgrp/client.go和server.go中的FreezeShard, InstallShard和DeleteShard, 并基于Num拒绝过期请求. 还需要修改shardkv clerk来处理ErrWrongGroup, 当shardgrp不在代表这个shard时需要返回. 通过JoinBasic和Deletebasic测试
- shardgrp需要对它不负责的key返回一个ErrWrongGroup错误. 你需要修改client.go来重新获取配置并重试
- 你需要想Put和Get一样通过rsm层运行FreezeShard, InstallShard和DeleteShard
- 你可以在RPC请求和响应中传递整个map作为状态, 这有助于保持shard传输代码的简洁
- 如果你的某个RPChandler在响应中包含了部分服务器状态, 如kv map, 你可能因为竞争而遇到bug. RPC系统必须读取map才能发送给调用者, 但他没有持有这个map上的锁. 但你的server可能需要在RPC系统处理的同时读取它. 一种解决方案是handler中返回map的副本

Task4: 扩展ChangeConfigTo来处理离开的分组, 如当前配置中存在但新配置中没有的shardgrp. 你需要通过TestJoinLeaveBasic5A.

Task5: 确保你的方案能通过Part A所有的测试, 这些测试将会检验你的分片kv服务是否支持多个group的加入和离开, shardgrps是否支持从快照重启, 某些shards离线的时候或涉及配置更改的时候仍能处理Get请求, 以及多个客户端与service交互同时teseter并发的调用ChangeConfigTo来重新平衡shards时保持线性一致性