---
title: Redis集群
date: 2020-06-22 18:30:24
tags:
categories: Redis
---

# Redis数据分区

**主从复制的问题**

主从复制是通过将master上的数据全量的复制到一个或多个节点上，这是一种通过数据冗余的形式来保证数据的安全性，但是当主节点发生故障时需要从它的从节点中选出一个作为新的主节点，剩下的从节点要与这个新的主节点进行全量复制，如果节点的数据量非常大的时候会带来两个主要问题，网络阻塞和从节点恢复数据会导致进程阻塞，最终会影响到对外提供服务的稳定性。

解决这个问题的关键是要在不影响服务的前提如何做到这点至关重要减少Redis的数据量，通常我们会采用数据分片的形式来解决问题。将主节点上的全量数据通过分区规则，拆分到不同的分区中，每个节点上可以保存一个或多个分区的数据，将这些节点组成集群对外提供服务。

**Redis数据分区的规则**

 Redis Cluser采用虚拟槽分区，所有的键根据哈希函数映射到0~16383整数槽内，计算公式：`slot = CRC16（key）& 16383`。集群中的每个主节点（Master）都负责处理16384个哈希槽中的一部分，当集群处于稳定状态时，每个哈希槽都只由一个主节点进行处理，每个主节点可以有一个到N个从节点（Slave），当主节点出现宕机或网络断线等不可用时，从节点能自动提升为主节点进行处理。

![key值到槽映射](/images/Redis/key值到槽映射.jpg)


# Redis集群

**Redis集群的特点**

- 数据按照slot存储分布在多个节点，节点间数据共享，可动态调整数据分布
- 可扩展性，节点可动态扩容与缩容
- 高可用性，部分节点不可用时，集群仍可用。通过增加Slave做standby数据副本，能够实现故障自动failover，节点之间通过gossip协议交换状态信息，用投票机制完成Slave到Master的角色提升

**Redis集群与单机相比较在功能上存在一些限制**

- key批量操作支持有限。如mset、mget，目前只支持具有相同slot值的key执行批量操作。对于映射为不同slot值的key由于执行mget、mget等操作可能存在于多个节点上因此不被支持。
- key事务操作支持有限。同理只支持多key在同一节点上的事务操作，当多个key分布在不同的节点上时无法使用事务功能。
- key作为数据分区的最小粒度，因此不能将一个大的键值对象如hash、list等映射到不同的节点。
- 支持多数据库空间。单机下的Redis可以支持16个数据库，集群模式下只能使用一个数据库空间，即db0。
- 复制结构只支持一层，从节点只能复制主节点，不支持嵌套树状复制结构。


# 集群环境搭建

## 1. 准备节点

| 节点 |  角色  |  状态  |         工作         |
| :--: | :----: | :----: | :------------------: |
| 6379 | Master | online |   处理slot:0-5460    |
| 6380 | Master | online | 处理slot:5461-10922  |
| 6381 | Master | online | 处理slot:10923-16383 |
| 6382 | Slave  | online |     复制6379节点     |
| 6383 | Slave  | online |     复制6380节点     |
| 6384 | Slave  | online |     复制6381节点     |

**配置文件redis-{port}.conf**
```conf
port 6379
logfile log/6379.log
cluster-enabled yes
dir data
cluster-node-timeout 15000
cluster-config-file nodes-6379.conf
```

**启动**
```shell
redis-server conf/redis-6379.conf
redis-server conf/redis-6380.conf
redis-server conf/redis-6381.conf
redis-server conf/redis-6382.conf
redis-server conf/redis-6383.conf
redis-server conf/redis-6384.conf
```

节点启动成功后会生成一个nodes-{port}.conf的集群配置文件。当集群内节点信息发生变化，如添加节点、节点下线、故障转移等。节点会自动保存集群状态到配置文件中。需要注意的是，Redis自动维护集群配置文件，不要手动修改。

**启动后集群配置文件里显示的内容：**
<id> <ip:port> <flags> <master> <ping-sent> <pong-recv> <config-epoch> <link-state> <slot> <slot> ... <slot>
- id: 节点ID,是一个40字节的随机字符串，这个值在节点启动的时候创建，并且永远不会改变。
- ip:port: 客户端与节点通信使用的地址.
- flags: 逗号分割的标记位，可能的值有: myself, master, slave, fail?, fail, handshake, noaddr, noflags. 下一部分将详细介绍这些标记.
- master: 如果节点是slave，并且已知master节点，则这里列出master节点ID,否则的话这里列出”-“。
- ping-sent: 最近一次发送ping的时间，这个时间是一个unix毫秒时间戳，0代表没有发送过.
- pong-recv: 最近一次收到pong的时间，使用unix时间戳表示.
- config-epoch: 节点的epoch值。每当节点发生失败切换时，都会创建一个新的，独特的，递增的epoch。如果多个节点竞争同一个哈希槽时，epoch值更高的节点会抢夺到。
- link-state: node-to-node集群总线使用的链接的状态，我们使用这个链接与集群中其他节点进行通信.值可以是 connected 和 disconnected.
- slot: 哈希槽值或者一个哈希槽范围. 从第9个参数开始，后面最多可能有16384个 数(limit never reached)。代表当前节点可以提供服务的所有哈希槽值。如果只是一个值,那就是只有一个槽会被使用。如果是一个范围，这个值表示为起始槽-结束槽，节点将处理包括起始槽和结束槽在内的所有哈希槽。

## 2. 节点握手

节点握手是指一批运行在集群模式下的节点通过Gossip协议彼此通信，达到感知对方的过程。

从6个节点中任意打开一个客户端输入：`cluster meet {ip} {port}`
```shell
127.0.0.1:6379> cluster meet 127.0.0.1 6380
127.0.0.1:6379> cluster meet 127.0.0.1 6381
127.0.0.1:6379> cluster meet 127.0.0.1 6382
127.0.0.1:6379> cluster meet 127.0.0.1 6383
127.0.0.1:6379> cluster meet 127.0.0.1 6384
```

> `cluster info`命令：打印集群信息

## 3. 分配槽

Redis集群把所有的数据映射到16384个槽中。每个key会映射为一个固定的槽，只有当节点分配了槽，才能响应和这些槽关联的键命令。
`cluster addslots`命令：为节点分配槽。所有槽分配完成后集群的状态会变为ok。


## 4. 主从复制

作为一个完整的集群，每个负责处理槽的节点应该具有从节点，保证当它出现故障时可以自动进行故障转移。集群模式下，Reids节点角色分为主节点和从节点。首次启动的节点和被分配槽的节点都是主节点，从节点负责复制主节点槽信息和相关的数据。

`cluster replicate <master_node_id>`命令：将当前从节点设置为node_id指定的master节点的slave节点，只能针对slave节点操作。


# redis-trib.rb搭建集群

redis-trib.rb是采用Ruby实现的Redis集群管理工具。内部通过Cluster相关命令帮我们简化集群创建、检查、槽迁移和均衡等常见运维操作，使用之前需要安装Ruby依赖环境。

1. Ruby安装

```shell
# 下载ruby
wget https://cache.ruby-lang.org/pub/ruby/2.3/ruby-2.3.1.tar.gz
# 安装ruby
tar xvf ruby-2.3.1.tar.gz
./configure -prefix=/usr/local/ruby
make
make install
cd /usr/local/ruby
sudo cp bin/ruby /usr/local/bin
sudo cp bin/gem /usr/local/bin

# 安装rubygems redis依赖
wget http://rubygems.org/downloads/redis-3.3.0.gem
gem install -l redis-3.3.0.gem
gem list --check redis gem

# 安装redis-trib.rb
sudo cp /{redis_home}/src/redis-trib.rb /usr/local/bin
```

2. 准备6个节点，与手动搭建方式一致

3. 创建集群

创建集群为每个主节点指定一个从节点
redis-trib.rb create --replicas <slave num> <master ...> <slave ...>

```shell
redis-trib.rb create --replicas 1 127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384
```

4. 集群完整性检查

集群完整性指所有的槽都分配到存活的主节点上，只要16384个槽中有一个没有分配给节点则表示集群不完整。可以使用`redis-trib.rb check`命令检测之前创建的集群是否成功。


# 重新分片

Redis集群提供了灵活的节点扩容和收缩方案。在不影响集群对外服务的情况下，可以为集群添加节点进行扩容,也可以下线部分节点进行缩容。扩容与缩容的本质是，对Redis的槽进行重新的分配。

## 重新分片流程
0) 制订槽分配计划，确保每个节点负责相似数量的槽，从而保证各节点的数据均匀。
1) 对目标节点发送`cluster setslot {slot} importing {sourceNodeId}`命令，让目标节点准备导入槽的数据。
2) 对源节点发送`cluster setslot {slot} migrating targetNodeId}`命令，让源节点准备迁出槽的数据。
3) 源节点循环执行`cluster getkeysinslot {slot} {count}`命令，获取count个属于槽{slot}的键。
4) 在源节点上执行`migrate {targetIp} {targetPort} "" 0 {timeout} keys{keys...}`命令，把获取的键通过流水线（pipeline）机制批量迁移到目标节点。
5) 重复执行步骤3）和步骤4）直到槽下所有的键值数据迁移到目标节点。
6) 向集群内所有主节点发送`cluster setslot {slot} node {targetNodeId}`命令，通知槽分配给目标节点。为了保证槽节点映射变更及时传播，需要遍历发送给所有主节点更新被迁移的槽指向新节点。

![重新分片流程](/images/Redis/重新分片流程.jpg)

## redis-trib相关命令

**加入节点**
redis-trib.rb add-node new_host:new_port existing_host:existing_port --slave--master-id 

**重新分配槽**
redis-trib.rb reshard host:port --from --to --slots --yes --timeout --pipeline
- host：port：必传参数，集群内任意节点地址，用来获取整个集群信息。
- ·--from：制定源节点的id，如果有多个源节点，使用逗号分隔，如果是all源节点变为集群内所有主节点，在迁移过程中提示用户输入。
- ·--to：需要迁移的目标节点的id，目标节点只能填写一个，在迁移过程中提示用户输入。
- ·--slots：需要迁移槽的总数量，在迁移过程中提示用户输入。
- ·--yes：当打印出reshard执行计划时，是否需要用户输入yes确认后再执行reshard。
- ·--timeout：控制每次migrate操作的超时时间，默认为60000毫秒。
- ·--pipeline：控制每次批量迁移键的数量，默认为10


# 命令路由

## 请求重定向

在集群模式下，Redis接收任何键相关命令时首先计算键对应的槽，再根据槽找出所对应的节点，如果节点是自身，则处理键命令；否则回复MOVED重定向错误，通知客户端请求正确的节点。

![请求重定向流程图](/images/Redis/请求重定向流程图.jpg)

请求重定向最重要的两个步骤是，计算槽和查找槽所在的节点。

### 计算槽

Redis首先需要计算键所对应的槽。根据键的有效部分使用CRC16函数计算出散列值，再`&16383`获得0～16383之间的整数值作为槽数。

- 执行`cluster keyslot {key}`命令可以反key对应的槽。
- 键内部使用大括号包含的内容又叫做hash_tag，它提供不同的键可以具备相同slot的功能。mget等命令优化批量调用时，键列表必须具有相同的slot，否则会报错，此时可以使用hash_tag，使键分配在统一个槽中.
例如：获取店铺1、店铺2、店铺3的最近10名关注者
mget follows:{10}:shop1 follows:{10}:shop2 vfollows:{10}:shop3
- 使用redis-cli命令时，可以加入-c参数支持自动重定向，简化手动发起重定向操作

### 查找槽所在的节点

集群内通过ping/pong消息交换每个节点都会知道所有节点的槽信息，这些信息会保存在ClusterState对象中

![查找槽所在的节点](/images/Redis/查找槽所在的节点.jpg)

请求重定向会增加网络开销，可以考虑在应用方缓存一份集群节点与槽的对照关系表，在发生MOVE重定向消息时，更新节点对照表，最大限度的减少MOVE几率发生。

## ASK重定向

Redis集群支持在线迁移槽（slot）和数据来完成水平伸缩，当slot对应的数据从源节点到目标节点迁移过程中可能出现一部分数据在源节点，而另一部分在目标节点，客户端需要做到智能识别，保证键命令可正常执行。

槽在迁移中发生请求的流程：
1）客户端根据本地slots缓存发送命令到源节点，如果存在键对象则直接执行并返回结果给客户端。
2）如果键对象不存在，则可能存在于目标节点，这时源节点会回复ASK重定向异常。格式如下：（error）ASK{slot}{targetIP}：{targetPort}。
3）客户端从ASK重定向异常提取出目标节点信息，发送asking命令到目标节点打开客户端连接标识，再执行键命令。如果存在则执行，不存在则返回不存在信息。

![ASK重定向流程](/images/Redis/ASK重定向流程.jpg)

## ASK与MOVED重定向的对比

ASK与MOVED虽然都是对客户端的重定向控制，但是有着本质区别：
- ASK重定向说明集群正在进行slot数据迁移，客户端无法知道什么时候迁移 完成，因此只能是临时性的重定向，客户端不会更新slots缓存
- 但是MOVED重定向说明键对应的槽已经明确指定到新的节点，因此需要更新slots缓存


# 故障转移

Redis集群自身实现了高可用，当集群内少量节点出现故障时通过自动故障转移保证集群可以正常对
外提供服务。

## 故障发现

Redis集群内节点通过ping/pong消息实现节点通信，消息不但可以传播节点槽信息，还可以传播其他状态如：主从状态、节点故障等。因此故障发现也是通过消息传播机制实现的，主要环节包括：主观下线（pfail）和客观下线（fail）。

### 主观下线

主观下线指某个节点认为另一个节点不可用，即下线状态，这个状态并不是最终的故障判定，只能代表一个节点的意见，可能存在误判情况。

![主观下线流程](/images/Redis/主观下线流程.jpg)

### 客观下线

Redis集群对于节点最终是否故障需要与集群中的其他主节点功能协商，多个节点协作完成故障发现的过程叫做客观下线。

客观下线指标记一个节点真正的下线，集群内多个节点都认为该节点不可用，从而达成共识的结果。如果是持有槽的主节点故障，需要为该节点进行故障转移。

![客观下线流程](/images/Redis/客观下线流程.jpg)

1) 当消息体内含有其他节点的pfail状态会判断发送节点的状态，如果发送节点是主节点，则对报告的pfail状态处理，从节点则忽略。

2) 找到pfail对应的节点结构，更新clusterNode内部下线报告链表。
**维护下线报告链表：**
每个节点ClusterNode结构中都会存在一个下线链表结构，保存了其他主节点针对当前节点的下线报告。
```C
typedef struct clusterNodeFailReport {
     struct clusterNode *node; /* 报告该节点为主观下线的节点 */
     mstime_t time; /* 最近收到下线报告的时间 */
} clusterNodeFailReport;
```

3) 根据更新后的下线报告链表告尝试进行客观下线。

![尝试客观下线流程](/images/Redis/尝试客观下线流程.jpg)

如果在`cluster-node-time * 2`时间内无法收集到一半以上槽节点的下线报告，那么之前的下线报告将会过期，也就是说主观下线上报的速度追赶不上下线报告过期的速度，那么故障节点将永远无法被标记为客观下线从而导致故障转移失败。因此不建议将cluster-node-time设置得过小。

向集群广播下线节点的fail消息:
- 通知集群内所有的节点标记故障节点为客观下线状态并立刻生效。
- 通知故障节点的从节点触发故障转移流程。


## 故障恢复

故障节点变为客观下线后，如果下线节点是持有槽的主节点则需要在它的从节点中选出一个新的主节点，从而保证集群的高可用。下线主节点的所有从节点承担故障恢复的义务，当从节点通过内部定时任务发现自身复制的主节点进入客观下线时，将会触发故障恢复流程。

![故障恢复流程](/images/Redis/故障恢复流程.jpg)

### 资格检查

每个从节点都要检查最后与主节点断线时间，判断是否有资格替换故障的主节点。如果从节点与主节点断线时间超过`cluster-node-time * cluster-slave-validity-factor`，则当前从节点不具备故障转移资格。cluster-slave-validity-factor从节点有效因子可以在配置文件中修改。

### 准备选举时间

计算下线主节点所有从节点的优先级排名（复制偏移量越大优先级越高），排名高的从节点优选获得选举权，排名每降低一位则延迟1秒开始选举。

![准备选举时间](/images/Redis/准备选举时间.jpg)

### 发起选举

当从节点定时任务检测到达故障选举时间到达后，发起选举流。

在集群内广播选举消息（FAILOVER_AUTH_REQUEST），并记录已发送过消息的状态，保证该从节点在一个配置纪元内只能发起一次选举。

### 选举投票

只有持有槽的主节点才会处理故障选举消息（FAILOVER_AUTH_REQUEST），因为每个持有槽的节点在一个配置纪元内都有唯一的一张选票，当接到第一个请求投票的从节点消息时回复FAILOVER_AUTH_ACK消息作为投票，之后相同配置纪元内其他从节点的选举消息将忽略。

如集群内有N个持有槽的主节点代表有N张选票。由于在每个配置纪元内持有槽的主节点只能投票给一个从节点，因此只能有一个从节点获得N/2+1的选票，保证能够找出唯一的从节点。

投票作废：每个配置纪元代表了一次选举周期，如果在开始投票之后的`cluster-node-timeout * 2`时间内从节点没有获取足够数量的投票，则本次选举作废。从节点对配置纪元自增并发起下一轮投票，直到选举成功为止。

![选举投票](/images/Redis/选举投票.jpg)

### 替换主节点

a. 当前从节点取消复制变为主节点。
b. 执行clusterDelSlot操作撤销故障主节点负责的槽，并执行clusterAddSlot把这些槽委派给自己。
c. 向集群广播消息，通知集群内所有的节点当前从节点变为主节点并接管了故障主节点的槽信息。
