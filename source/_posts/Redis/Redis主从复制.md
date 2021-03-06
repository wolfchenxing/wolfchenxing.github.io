---
title: Redis主从复制
date: 2020-05-08 21:03:15
tags:
categories: Redis
---

# 什么是主从复制

## 概述

在分布式系统中为了解决单点问题，通常会把数据复制多个副本部署到其他机器，满足故障恢复和负载均衡等需求。Redis也是如此，它为我们提供了复制功能，实现了相同数据的多个Redis副本。复制功能是实现**高可用**Redis的基础。

## 作用

- 数据冗余：主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式
- 故障恢复：当主服务器出现问题时，可由从服务器提供服务，实现快速的故障恢复，实际上是一种服务的冗余
- 负载均衡：在主从复制的基础上，配合读写分离，可以由主服务器提供写服务，从服务器提供读服务，分担服务器负载；尤其在写少读多的场景，通过多个从服务器分担读负载，可大大提高Redis服务器的并发量
- 高可用基石：主从复制还是哨兵和集群能够实施的基础，因此主从复制是Redis高可用的基础

## 拓扑结构

### 一对一

主从一对一，适用于故障转移。

### 星形结构

适用于读多写少的场景，可以把读命令发送到从节点来分担主节点压力。如：keys命令，可以在其中一台从节点上执行，防止慢查询对主节点造成阻塞从而影响线上服务的稳定性。

面对写多读少的场景，多个从节点会导致主节点写命令的多次发送从而过度消耗网络带宽，同时也加重了主节点的负载影响服务稳定性。

### 树状结构

从节点不但可以复制主节点数据，同时可以作为其他从节点的主节点继续向下层复制。通过这种结构可以有效降低主节点负载和需要传送给从节点的数据量。


# 主从复制原理

## 复制过程

1. 从节点保存主节点（master）信息。
2. 从节点（slave）内部通过每秒运行的定时任务维护复制相关逻辑，当定时任务发现存在新的主节点后，会尝试与该节点建立网络连接。
如果从节点无法建立连接，定时任务会无限重试直到连接成功或者执行slaveof no one取消复制。
3. 发送ping命令，检测主从之间网络套接字是否可用和主节点当前是否可接受处理命令。返回pong则继续复制流程，否则断开连接。
4. 权限验证。如果主节点设置了requirepass参数，则需要密码验证，从节点必须配置masterauth参数保证与主节点相同的密码才能通过验证；如果验证失败复制将终止，从节点重新断开重连。
5. 连接成功，发送从节点的ip与port给主节点。
6. **数据同步**。从服务向主服务器发送psync命令开始同步，可以分为全量复制和部分复制。
7. **命令传播**，当主节点把当前的数据同步给从节点后，便完成了复制的建立流程。接下来主节点会持续地把写命令发送给从节点，保证主从数据一致性。

## 数据同步

### 全量复制

psync ? -1命令发起全量复制请求，一般用于初次复制场景，这时master服务器会将自己的rdb文件发送给slave服务器进行数据同步，并记录同步期间的其他写入，再发送给slave服务器，以达到完全同步的目的。

![全量复制流程图](/images/Redis/全量复制流程图.jpg)

RDB文件从创建到传输完毕消耗的总时间超过repl-timeout所配置的值（默认60秒），从节点将放弃接受RDB文件并清理已经下载的临时文件，导致全量复制失败，数据量较大的节点容易出现主从数据同步超时，需要响应的调整repl-timeout的值。

Redis支持无盘复制，生成的RDB文件不保存到硬盘而是直接通过网络发送给从节点，通过repl-diskless-sync参数控制，默认关闭。

client buffer是在server端实现的一个读取缓冲区。redis server在接收到客户端的请求后，把响应结果写入到client buffer中，而不是直接发送给客户端。

client-output-buffer-limit slave 256MB 64MB 60，表示主节点输出给从节点的缓存(output-buffer)大小，从库的复制客户端如果60秒内缓冲区消耗持续大于64MB或者直接超过256MB时，主节点将直接关闭复制客户端连接。

如果主节点创建和传输RDB的时间过长，对于高流量写入场景非常容易造成主节点复制客户端缓冲区溢出。

### 部分复制

psync  <runid> <offset>命令发起部分复制请求，用于处理在主从复制中因网络闪断等原因造成的数据丢失场景，当从节点再次连上主节点后，如果条件允许，主节点会补发丢失数据给从节点。因为补发的数据远远小于全量数据，可以有效避免全量复制的过高开销。

![部分复制流程图](/images/Redis/部分复制流程图.jpg)

1. 当主从节点之间网络出现中断时，如果超过repl-timeout时间，主节点会认为从节点故障并中断复制连接。
2. 主从连接中断期间主节点依然响应命令并且保存在主节点的复制积压缓冲区中，但因复制连接中断命令无法发送给从节点。
3. 当主从节点网络恢复后，从节点会再次连上主节点。
4. 当主从连接恢复后，由于从节点之前保存了自身已复制的偏移量和主节点的运行ID。因此会把它们当作psync参数发送给主节点，要求进行部分复制操作。
5. 主节点接到psync命令后首先核对参数runId是否与自身一致，如果一致，说明之前复制的是当前主节点；之后根据参数offset在自身复制积压缓冲区查找，如果偏移量之后的数据存在缓冲区中，则对从节点发送+CONTINUE响应，表示可以进行部分复制。
6. 主节点根据偏移量把复制积压缓冲区里的数据发送给从节点，保证主从复制进入正常状态。

**构成部分复制的三要素**：复制偏移量、复制积压缓冲区、服务器运行ID

#### 复制偏移量

参与复制的主从节点都会维护自身复制偏移量。主节点（master）在处理完写入命令后，会把命令的字节长度做累加记录。从节点（slave）每秒钟上报自身的复制偏移量给主节点，因此主节点也会保存从节点的复制偏移量。可以通过主节点的master_repl_offset信息，判断主从节点复制相差的数据量，根据这个差值判定当前复制的健康度。

#### 复制积压缓冲区

复制积压缓冲区是一个固定长度的FIFO，默认1M，当主节点有连接的从节点（slave）时被创建，这时主节点（master）响应写命令时，不但会把命令发送给从节点，还会写入复制积压缓冲区。

> 正确的估算和设置复制积压缓冲区的大小，可以避免主服务器执行全量复制；
复制积压缓冲区大小=second(服务断开重连时间)*write_size_per_second(每秒写入数据量)

#### 服务器运行ID

主节点根据runid判断能否进行部分复制：

如果从服务器保存的runid与主服务器现在的runid相同，说明主从服务器之前同步过，主服务器会继续尝试使用部分复制(到底能不能部分复制还要看offset和复制积压缓冲区的情况)；

如果从服务器保存的runid与主服务器现在的runid不同，说明从节点在断线前同步的Redis节点并不是当前的主服务器，只能进行全量复制。

## 命令传播

在命令传播阶段，除了发送写命令，主从服务器还维持着心跳机制：PING和REPLCONF ACK

![命令传播阶段](/images/Redis/命令传播阶段.jpg)

1）主节点默认每隔10秒对从节点发送ping命令，判断从节点的存活性和连接状态。可通过参数repl-ping-slave-period控制发送频率。
2）从节点在主线程中每隔1秒发送replconf ack {offset}命令，给主节点上报自身当前的复制偏移量。

replconf命令主要作用如下：

- 实时监测主从节点网络状态。
- 上报自身复制偏移量，检查复制数据是否丢失，如果从节点数据丢失，再从主节点的复制缓冲区中拉取丢失数据。
- 实现保证从节点的数量和延迟性功能，通过min-slaves-to-write、min-slaves-max-lag参数配置定义。


# 操作与配置

## 建立复制

slaveof本身是异步命令，执行slaveof命令时，节点只保存主节点信息后返回，后续复制流程
在节点内部异步执行。

方式一：在配置文件中加入 slaveof <masterIp> <masterPort>，随Redis启动生效
方式二：直接使用命令 slaveof <masterIp> <masterPort>生效

> 用info replication命令查看复制相关信息

## 关闭复制

在从节点执行slaveof no one来断开与主节点复制关系。

关闭复制主要流程：
1）关闭与主节点复制关系。
2）从节点晋升为主节点。

切换主节点操作流程如下：
1）关闭与旧主节点复制关系。
2）与新主节点建立复制关系。
3）删除从节点当前所有数据。
4）对新主节点进行复制操作。

切换主节点后从节点会清空之前所有的数据，但是如果是从节点关闭与主节点的复制关系时不会清空当前节点上的数据。

## 安全性设置

为了提升安全性，可以为主节点设置requirepass参数进行密码验证，这时所有的客户端访问必须使用auth命令实行校验，配置从节点的masterauth参数与主节点密码保持一致，这样从节点才可以正确地连接到主节点并发起复制流程。

```
// 主节点设置密码
config set requirepass 123456
// 从节点配置主节点密码
config set materauth 123456
```

## 只读

由于复制模式是单向的，从节点接收写命令的话无法让主节点感知到，这会造成数据不一致，因此从节点使用slave-read-only=yes配置为只读模式。

## 数据传输延迟

repl-disable-tcp-nodelay参数用于控制是否关闭TCP_NODELAY，默认关闭。

- 当关闭时，主节点产生的命令数据无论大小都会及时地发送给从节点，这样主从之间延迟会变小，但增加了网络带宽的消耗。适用于主从之间的网络环境良好的场景，如同机架或同机房部署。

- 当开启时，主节点会合并较小的TCP数据包从而节省带宽。默认发送时间间隔取决于Linux的内核，一般默认为40毫秒。这种配置节省了带宽但增大主从之间的延迟。适用于主从网络环境复杂或带宽紧张的场景，如跨机房部署。


# 常见问题

## 数据延迟与不一致

a、优化主从节点之间的网络环境（如在同机房部署）；
b、监控主从节点延迟（通过offset）判断，如果从节点延迟过大，通知应用不再通过该从节点读取数据；
c、Redis复制提供了slave-serve-stale-data参数，默认开启状态。如果开启则从节点依然响应所有命令。对于无法容忍不一致的应用场景可以设置no来关闭命令执行，此时从节点除了info和slaveof命令之外所有的命令只返回“SYNC with master in progress”信息。

## 数据过期问题

在主从复制场景下，为了主从节点的数据一致性，从节点不会主动删除数据，而是由主节点控制从节点中过期数据的删除。由于主节点的惰性删除和定期删除策略，都不能保证主节点及时对过期数据执行删除操作，很容易读取到已经过期的数据。Redis 3.2中，从节点在读取数据时，增加了对数据是否过期的判断：如果该数据已过期，则不返回给客户端；将Redis升级到3.2可以解决数据过期问题。

## 故障转移

当主节点或从节点出现问题而发生更改时，需要及时修改应用程序读写Redis数据的连接；连接的切换可以手动进行，或者自己写监控程序进行切换，但前者响应慢、容易出错，后者实现复杂，成本都不算低。可以使用哨兵机制实现自动故障转移。

## 复制超时

注意Redis单机数据量不要过大，另一方面就是适当增大repl-timeout值，具体的大小可以根据bgsave耗时来调整。
主节点会向从节点发送PING命令，频率由repl-ping-slave-period控制；该参数应明显小于repl-timeout值(后者至少是前者的几倍)。否则，如果两个参数相等或接近，网络抖动导致个别PING命令丢失，此时恰巧主节点也没有向从节点发送数据，则从节点很容易判断超时。

## 输出缓冲区溢出

通过client-output-buffer-limit调整输出缓冲区的大小避免复制失败

## 安全重启

如内存碎片过高需要重启主服务器，应使用debug reload避免runid与offset改变而执行全量复制。