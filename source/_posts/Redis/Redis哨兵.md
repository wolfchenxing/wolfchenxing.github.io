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
- Sentinel节点不应该部署在一台物理“机器”上。
同一台物理机意味着如果这台机器有什么硬件故障，所有的虚拟机都会受到影响，为了实现Sentinel节点集合真正的高可用。
- 部署至少三个且奇数个的Sentinel节点。
增加Sentinel节点的个数提高对于故障判定的准确性，因为故障转移时领导者选举采用过半原则，奇数个节点可以最低限度满足该条件。
- 为每个业务场景部署一套Sentinel。


# Sentinel实现原理




# Sentinel命令
