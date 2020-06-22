---
title: Redis哨兵
date: 2020-06-03 22:15:07
tags:
categories: Redis
---

# 手动故障转移

在主从复制的架构中一旦主节点出现故障，需要手动将一个从节点晋升为主节点，同时需要修改应用方的主节点地址，还需要命令其他从节点去复制新的主节点，整个过程都需要人工干预。

**故障转移过程：**

1. 主节点发生故障后，客户端（client）连接主节点失败，所有的从节点与主节点连接失败造成复制中断。
2. 如果主节点无法正常启动，需要选出一个从节点，对其执行slaveof no one命令使其成为新的主节点。
3. 原来的从节点成为新的主节点后，更新应用方的主节点信息，重新启动应用方。
4. 客户端命令另一个从节点去复制新的主节点（new-master），slaveof host port
5. 待原来的主节点恢复后，让它去复制新的主节点。


# Redis Sentinel架构

当主节点出现故障时，Redis Sentinel能自动完成故障发现和转移，并通知应用方，从而实现真正的高可用。

Redis Sentinel是一个分布式架构，其中包含若干个Sentinel节点和Redis数据节点，每个Sentinel节点会对数据节点和其余Sentinel节点进行监控，当它发现节点不可达时，会对节点做下线标识。如果被标识的是主节点，它还会和其他Sentinel节点进行“协商”，当大多数Sentinel节点都认为主节点不可达时，它们会选举出一个Sentinel节点来完成自动故障转移的工作，同时会将这个变化实时通知给Redis应用方。

![Sentinel架构](/images/Redis/Sentinel架构.jpg)

## Sentinel故障转移过程

1. 每个Sentinel节点通过定期监控发现主节点出现了故障。
2. 多个Sentinel节点对主节点的故障达成一致，选举出一个节点作为领导者负责故障转移
3. Sentinel领导者节点执行了故障转移

## Sentinel功能

- 监控：Sentinel节点会定期检测Redis数据节点、其余Sentinel节点是否可达。
- 通知：Sentinel节点会将故障转移的结果通知给应用方。
- 故障转移：实现从节点晋升为主节点并维护后续正确的主从关系。
- 配置提供者：在Redis Sentinel结构中，客户端在初始化的时候连接的是Sentinel节点集合，从中获取主节点信息。

## Sentinel架构的好处
- 对于节点的故障判断是由多个Sentinel节点共同完成，这样可以有效地防止误判。
- Sentinel节点集合是由若干个Sentinel节点组成的，这样即使个别Sentinel节点不可用，整个Sentinel节点集合依然是健壮的。


# 环境部署
## 配置主节点和从节点
（1）主节点redis-6379.conf

```conf
#redis以守护进程方式运行
daemonize yes
#日志文件
logfile 6379.log      
#rdb文件
dbfilename dump-6379.rdb
```
（2）从节点redis-6380.conf
```conf
#redis以守护进程方式运行
daemonize yes
#日志文件
logfile "6380.log"
#rdb文件
dbfilename "dump-6380.rdb"
#设置主节点
slaveof 127.0.0.1 6379
```

## 启动主节点和从节点
```shell
redis-sever redis-6379.conf
redis-server redis-6380.conf
redis-server redis-6381.conf
```

## 确认主从关系
`info replication`或` role`命令

## 配置Sentinel节点
sentinel-26379.conf
```conf
port 26379
daemonize yes
logfile "26379.log"

#sentinel monitor <master-name> <ip> <port> <quorum>
#监控127.0.0.1:6379这个主节点，别名demo-master，至少2个Sentinel节点认为失败时做故障转移
sentinel monitor demo-master 127.0.0.1 6379 2

#sentinel down-after-milliseconds <master-name> <times>
#超过指定秒没有收到节点回复，判为故障下线
sentinel down-after-milliseconds demo-master 30000

#sentinel parallel-syncs <master-name> <nums>
#故障转移时的从节点向主节点发起并发复制请求的数量
sentinel parallel-syncs demo-master 1

#sentinel failover-timeout <master-name> <times>
#故障转移超时时间
sentinel failover-timeout demo-master 180000
```

## 启动Sentinel节点
```shell
#两种方式启动Sentinel节点
#1.使用redis-sentinel命令：
redis-sentinel sentinel-26379.conf
#2.使用redis-server命令加--sentinel参数：
redis-server sentinel-26379.conf --sentinel
```

## 确认Sentinel
`info sentinel`

## 注意点
- Sentinel节点不应该部署在同一台物理“机器”上。
同一台物理机意味着如果这台机器有什么硬件故障，所有的虚拟机都会受到影响，为了实现Sentinel节点集合真正的高可用。
- 部署至少三个且奇数个的Sentinel节点。
增加Sentinel节点的个数提高对于故障判定的准确性，因为故障转移时领导者选举采用过半原则，奇数个节点可以最低限度满足该条件。
- 为每个业务场景部署一套Sentinel。


# Sentinel实现原理

## 三个定时监控任务

1. 每隔10秒，每个Sentinel节点会向主节点和从节点发送info命令获取Redis数据节点的信息。

![Sentinel定时监控任务1](/images/Redis/Sentinel定时监控任务1.jpg)

**作用：**
- 通过向主节点执行info命令，获取从节点的信息，这也是为什么Sentinel节点不需要显式配置监控从节点。
- 当有新的从节点加入时都可以立刻感知出来。
- 节点不可达或者故障转移后，可以通过info命令实时更新节点拓扑信息。

2. 每隔2秒，每个Sentinel节点会向Redis数据节点的`__sentinel__：hello`频道上发送该Sentinel节点对于主节点的判断以及当前Sentinel节点的信息，同时每个Sentinel节点也会订阅该频道，来了解其他Sentinel节点以及它们对主节点的判断。

![Sentinel定时监控任务2](/images/Redis/Sentinel定时监控任务2.jpg)

**消息格式：**
<Sentinel节点IP> <Sentinel节点端口> <Sentinel节点runId> <Sentinel节点配置纪元>
<主节点名字> <主节点Ip> <主节点端口> <主节点配置纪元>

**作用：**
- 发现新的Sentinel节点：通过订阅主节点的`__sentinel__：hello`了解其他的Sentinel节点信息，如果是新加入的Sentinel节点，将该Sentinel节点信息保存起来，并与该Sentinel节点创建连接。
- Sentinel节点之间交换主节点的状态，作为后面客观下线以及领导者选举的依据。

3. 每隔1秒，每个Sentinel节点会向主节点、从节点、其余Sentinel节点发送一条ping命令做一次心跳检测，来确认这些节点当前是否可达。

![Sentinel定时监控任务3](/images/Redis/Sentinel定时监控任务3.jpg)

## 主观下线和客观下线

当一个Redis数据节点可能下线时，Sentinel节点需要经过主观下线检查和客观下线检查，如果检查结果都表明Redis数据节点已经下线时，则进入故障转移阶段。

1. 主观下线

每个Sentinel节点会每隔1秒对主节点、从节点、其他Sentinel节点发送ping命令做心跳检测，当这些节点超过down-after-milliseconds没有进行有效回复，Sentinel节点就会对该节点做失败判定。
但这不能的断言节点已经下线，需要结合其他Sentinel节点判断结果进行综合评估。

![Sentinel主观下线](/images/Redis/Sentinel主观下线.jpg)

2. 客观下线

当Sentinel主观下线的节点是主节点时，该Sentinel节点会通过`sentinel ismaster-down-by-addr`命令向其他Sentinel节点询问对主节点的判断，当超过<quorum>个数，Sentinel节点认为主节点确实有问题，这时该Sentinel节点会做出客观下线的决定。

sentinel is-master-down-by-addr <ip> <port> <current_epoch> <runid>
例如：sentinel is-master-down-by-addr 127.0.0.1 6379 0 *
- ip：主节点IP。
-port：主节点端口。
- current_epoch：当前配置纪元。
- runid：此参数有两种类型，当runid等于“*”时，作用是Sentinel节点直接交换对主节点下线的判定。在领导者Sentinel节点选举时，runid等于当前Sentinel节点的runid，作用是当前Sentinel节点希望目标Sentinel节点同意自己成为领导者的请求。

返回结果包含三个参数：
- down_state：目标Sentinel节点对于主节点的下线判断，1是下线，0是在线。
- leader_runid：当leader_runid等于“*”时，代表返回结果是用来做主节点是否不可达，当leader_runid等于具体的runid，代表目标节点同意runid成为领导者。
- leader_epoch：领导者纪元。

![Sentinel客观下线](/images/Redis/Sentinel客观下线.jpg)

## 领导者Sentinel节点选举

Sentinel节点对于主节点已经做了主观下线和客观下线后，需要从众多的Sentinel节点中选出一个节点，进行故障转移的工作，所以Sentinel节点之间会在故障转移前线做一个领导者选举的工作。

Redis使用了Raft算法实现领导者选举，大体思路：
1) 每个在线的Sentinel节点都有资格成为领导者，当它确认主节点客观下线检查时候，会向其他Sentinel节点发送`sentinel is-master-down-by-addr`命令，要求将自己设置为领导者。
2) 每个节点在每个选举轮次中只有一次投票权，收到命令的Sentinel节点，如果没有同意过其他Sentinel节点的`sentinelis-master-down-by-addr`命令，将同意该请求，否则拒绝。
3) 如果该Sentinel节点发现自己的票数已经大于等于max（quorum，num（sentinels）/2+1），那么它将成为领导者。
4) 如果此过程没有选举出领导者，将进入下一次选举，current_epoch加1。

例：
1. s1(哨兵节点)节点首先完成客观下线的检查，然后向s2和s3发送成为领导者的请求，则s1接受的同意为s2、s3
2. s2节点完成客观下线的检查，然后向s1和s3发送成为领导者的请求，则s2接受的同意为s1
3. s3节点完成客观下线的检查，然后向s1和s2发送成为领导者的请求，则s3未接受到同意

## 故障转移

1. 在从节点列表中选出一个作为新的主节点
[1] 过滤：“不健康”（主观下线、断线）、5秒内没有回复过Sentinel节点ping响应、与主节点失联超过down-after-milliseconds*10秒。（断线时间越长主从数据不一致问题越严重）
[2] 选择slave-priority（从节点优先级）最高的从节点列表，如果存在则返回，不存在则继续。
[3] 选择复制偏移量最大的从节点（复制的最完整），如果存在则返回，不存在则继续。
[4] 选择runid最小的从节点。
**挑选从节点的重要原则：在从节点列表中挑选与主节点数据最一致的节点。**
![故障转移选举主节点](/images/Redis/故障转移选举主节点.jpg)
2. Sentinel领导者节点会对第一步选出来的从节点执行`slaveof no one`命令让其成为主节点。
3. Sentinel领导者节点会向剩余的从节点发送命令，让它们成为新主节点的从节点，复制规则和parallel-syncs参数有关。`slaveof ip port`
4. Sentinel节点集合会将原来的主节点更新为从节点，并保持着对其关注，当其恢复后命令它去复制新的主节点。


# Sentinel命令

```shell
#1.展示所有被监控的主节点状态以及相关的统计信息
sentinel masters

#2.展示指定<master name>的主节点状态以及相关的统计信息
sentinel master<master name>

#3.展示指定<master name>的从节点状态以及相关的统计信息
sentinel slaves<master name>

#4.展示指定<master name>的Sentinel节点集合（不包含当前Sentinel节点）
sentinel sentinels<master name>

#5.返回指定<master name>主节点的IP地址和端口
sentinel get-master-addr-by-name<master name>

#6.对指定<master name>主节点进行强制故障转移
sentinel failover<master name>

#7.检测当前可达的Sentinel节点总数是否达到<quorum>的个数
sentinel ckquorum<master name>

#8.取消当前Sentinel节点对于指定<master name>主节点的监控
sentinel remove<master name>

#9.通过命令的形式来完成Sentinel节点对主节点的监控
sentinel monitor<master name><ip><port><quorum>

#10.动态修改Sentinel节点配置选项
sentinel set<master name>

#11.Sentinel节点之间用来交换对主节点是否下线的判断，根据参数的不同，还可以作为Sentinel领导者选举的通信方式
sentinel is-master-down-by-addr
```