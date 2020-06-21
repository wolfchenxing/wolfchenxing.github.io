---
title: Redis持久化
date: 2020-04-27 22:21:55
tags:
categories: Redis
---

# RDB持久化

## 触发方式

### 手动触发

- save：阻塞式，内存较大的实例在执行过程中会造成长时间的阻塞，影响主进程上的正常服务请求
- bgsave：fork子进程，RDB持久化的过程在子进程中进行，完成后自动结束进程。阻塞发生在fork阶段，时间较短。

### 自动触发

满足RDB持久化条件后会自动执行持久化过程。

**触发条件**

1. 使用save相关配置
    配置格式：save <seconds> <changes>
    表示<seconds>秒内数据集存在<changes>次修改时，自动触发bgsave
> 实现原理：
serverCron服务定时器每100ms执行一次检查，满足以下两个条件进行bgsave
（1）now() - rdb_last_save_time < m(指定秒数)
（2）rdb_changes_since_last_save > n(修改次数))

2. 主从复制场景下，如果从节点执行全量复制操作，则主节点会执行bgsave命令，并将rdb文件发送给从节点
3. debug reload命令
4. 执行shutdown命令时，如果没有开启AOF自动执行bgsave

## RDB持久化步骤

1. 执行bgsave命令，Redis父进程判断当前是否存在正在执行的子进程，如RDB/AOF子进程，如果存在bgsave命令直接返回
2. 父进程执行fork操作创建子进程，fork操作过程中父进程会阻塞，通过info stats命令查看latest_fork_usec选项，可以获取最近一个fork操作的耗时，单位为微秒。
3.  父进程fork完成后，bgsave命令返回“Background saving started”信息并不再阻塞父进程，可以继续响应其他命令。
4.  子进程创建RDB文件，根据父进程内存生成临时快照文件，完成后对原有文件进行原子替换。执行lastsave命令可以获取最后一次生成RDB的时间，对应info统计的rdb_last_save_time选项。
5.  进程发送信号给父进程表示完成，父进程更新统计信息，具体见info Persistence下的rdb_*相关选项。

## redis.conf中相关配置

```
// 默认打开RDB持久化，bgsave自动触发的条件
save 900 1
save 300 10
save 60 10000
// fork子进程内存不足或RDB文件所在的文件夹没有写入权限，Redis是否停止执行写命令
stop-writes-on-bgsave-error yes
// 是否开启RDB文件压缩，默认采用LZF算法对RDB文件进行压缩。压缩针对数据库中的字符串进行的，且只有在字符串达到一定长度(20字节)时才会进行
rdbcompression yes
// RDB文件名
dbfilename dump.rdb
// 是否开启RDB文件的校验，在写入文件和读取文件时都起作用；关闭checksum在写入文件和启动文件时大约能带来10%的性能提升，但是数据损坏时无法发现
rdbchecksum yes
// RDB和AOF文件所在目录
dir ./ 
```

## RDB文件

RDB文件是经过压缩的二进制文件。不同版本的RDB文件格式不同，故无法利用不同版本的RDB文件恢复数据。

```
----------------------------#
52 45 44 49 53              # 魔数"REDIS"
30 30 30 37                 # RDB数据库版本号，3.0服务器数据库为6，3.2服务器数据库为7. "0007" = 7
----------------------------
FA                          # 辅助信息
$string-encoded-key         # 包括Redis服务器版本，系统32或64位,创建时间，可用内存
$string-encoded-value       #                               
----------------------------
FE 00                       # 数据库编号默认16个库. db number = 00
FB                          # 数据库键值对总数量
$length-encoded-int         # 键值对总数
$length-encoded-int         # 过期键值对总数
----------------------------# Key-Value pair starts
FD $unsigned-int            # 过期时间单位秒, 四个字节长的无符号int表示
$value-type                 # 1个字节，表示编码方式
$string-encoded-key         # key值，字符串类型
$encoded-value              # value值,编码方式
----------------------------
FC $unsigned long           # 过期时间单位毫秒, 八个字节长的无符号int表示
$value-type                 # 1个字节，表示编码方式：string,list,set,inset等共14种
$string-encoded-key         # key值，字符串类型
$encoded-value              # value值,编码方式
----------------------------# 无过期时间的键值对      
$value-type                 # 1个字节，表示编码方式
$string-encoded-key         # key值，字符串类型
$encoded-value              # value值,编码方式
----------------------------
FE $length-encoding         # 当前数据库结束，下一个数据库开始（多个数据库场景）.
----------------------------

FF                          ## RDB文件结束
8-byte-checksum             ## 环冗余校验码，Redis采用crc-64-jones算法，初始值为0.
```

![RDB二进制文件结构](/images/Redis/rdb-file-structure.jpg)

1. REDIS：常量，保存着“REDIS”5个字符。 
2. db_version：RDB文件的版本号，注意不是Redis的版本号。 
3. SELECTDB 0 pairs：表示一个完整的数据库(0号数据库)，同理SELECTDB 3 pairs表示完整的3号数据库；只有当数据库中有键值对时，RDB文件中才会有该数据库的信息(上图所示的Redis中只有0号和3号数据库有键值对)；如果Redis中所有的数据库都没有键值对，则这一部分直接省略。其中：SELECTDB是一个常量，代表后面跟着的是数据库号码；0和3是数据库号码；pairs则存储了具体的键值对信息，包括key、value值，及其数据类型、内部编码、过期时间、压缩信息等等。 
4. EOF：常量，标志RDB文件正文内容结束。 
5. check_sum：前面所有内容的校验和；Redis在载入RBD文件时，会计算前面的校验和并与check_sum值比较，判断文件是否损坏。 

## 优缺点

**优点**
- RDB是一个非常紧凑（有压缩）的文件，它保存了某个时间点的数据，非常适用于数据的备份，可以方便传送到另一个远端数据中心，适用于灾难恢复
- 保存RDB文件时父进程唯一需要做的就是fork出一个子进程，接下来的工作全部交由子进程来做，父进程不需要再做其他IO操作，可以最大化Redis性能
- 与AOF相比，在恢复大的数据集时，RDB方式会更快一些

**缺点**
- Redis意外宕机时，会丢失部分数据
- 当Redis数据量比较大时，fork的过程是非常耗时的，fork子进程时是会阻塞的，在这期间Redis不能响应客户端的请求
- RDB文件的致命缺点在于其数据快照的持久化方式决定了必然做不到实时持久化，数据的大量丢失很多时候是无法接受
- RDB文件需要满足特定格式，兼容性差

# AOF持久化

AOF持久化是通过保存Redis服务器所执行的写命令来记录数据库状态，也就是每当Redis执行一个改变数据集的命令，这个命令就会被追加到AOF文件的末尾。

## 触发方式

### 手动触发

使用bgrewriteaof命令：Redis主进程fork子进程来执行AOF重写，这个子进程创建新的AOF文件来存储重写结果，防止影响旧文件。**因为fork采用了写时复制机制，子进程不能访问在其被创建出来之后产生的新数据。**Redis使用“**AOF重写缓冲区**”保存这部分新数据，最后父进程将AOF重写缓冲区的数据写入新的AOF文件中然后使用新AOF文件替换老文件。

### 自动触发

redis.conf中相关配置如下：
- appendonly：是否打开AOF持久化功能，appendonly默认是 no, 改成appendonly yes
- appendfilename：AOF文件名称
- appendfsync：同步频率
- auto-aof-rewrite-min-size：如果文件大小小于此值不会触发AOF，默认64MB
- auto-aof-rewrite-percentage：Redis记录最近的一次AOF操作的文件大小，如果当前AOF文件大小增长超过这个百分比则触发一次重写，默认100

## AOF工作流程

1. **命令追加（append）**：所有的写入命令会追加到aof_buf（缓冲区）中。
2. **文件写入（write）和文件同步（sync）**：AOF缓冲区根据对应的策略向硬盘做同步操作。
3. **文件重写（rewrite）：**随着AOF文件越来越大，需要定期对AOF文件进行重写，达到压缩的目的。
4. 当Redis服务器重启时，可以加载AOF文件进行数据恢复。

**AOF文件采用文本协议格式的好处**
- 文本协议具有很好的兼容性。
- 文本协议具有可读性，方便直接修改和处理。
- 开启AOF后，所有写入命令都包含追加操作，直接采用协议格式，避免了二次处理开销。

## AOF重写机制

### 重写必要性
重写后的AOF文件缩小文件体积有如下原因：
1） 进程内已经唾弃的数据不再写入文件
2） 旧的AOF文件含有无效命令，如del key1、hdel key2、srem keys、set a 111、set a222等。重写使用      进程内数据直接生成，这样新的AOF文件只保留最终数据的写入命令。
3） 多条写命令可以合并为一个，如：lpush list a、lpush list b、lpush list c可以转化为：lpush list a b c。为了防止单条命令过大造成客户端缓冲区溢出，对于list、set、hash、zset等类型操作，以64个元素为界拆分为多条。

AOF重写降低了文件占用空间，除此之外，另一个目的是：更小的AOF文件可以更快地被Redis加载。

### 重写触发机制
1）、手动触发，直接调用bgrewriteaof命令。
2）、自动触发，根据auto-aof-rewrite-min-size和auto-aof-rewrite-percentage参数确定自动触发时机。
- auto-aof-rewrite-min-size：表示运行AOF重写时文件最小体积，默认为64MB。
- auto-aof-rewrite-percentage：代表当前AOF文件空间（aof_current_size）和上一次重写后AOF文件空间（aof_base_size）的比值。
        

自动触发的两个条件:
- aof_current_size>auto-aof-rewrite-minsize
- (aof_current_sizeaof_base_size)/aof_base_size>=auto-aof-rewritepercentage

### 重写工作流程
1） 执行AOF重写请求。如果当前进程正在执行AOF重写则不执行重写操作，如果进程在执行bgsave则等待执行完毕后再执行。

2） 父进程执行fork创建子进程，开销等同于bgsave过程。

3.1）主进程fork操作完成后，继续响应其他命令。所有修改命令依然写入AOF缓冲区并根据appendfsync策略同步到硬盘，保证原有AOF机制正确性。
        
3.2）由于fork操作运用写时复制技术，子进程只能共享fork操作时的内存数据。由于父进程依然响应命令，Redis使用“AOF重写缓冲区”保存这部分新数据，防止新AOF文件生成期间丢失这部分数据。

4）子进程根据内存快照，按照命令合并规则写入到新的AOF文件。每次批量写入硬盘数据量由配置aof-rewrite-incremental-fsync控制，默认为32MB，防止单次刷盘数据过多造成硬盘阻塞。

5.1）新AOF文件写入完成后，子进程发送信号给父进程，父进程更新统计信息，具体见info persistence下的aof_*相关统计。
        
5.2）父进程把AOF重写缓冲区的数据写入到新的AOF文件。

5.3）使用新AOF文件替换老文件，完成AOF重写。

## 优缺点

**优点**
- 支持秒级持久化
- 可用于不同版本的Redis服务器，兼容性强
- AOF文件可读性高，易于分析

**缺点**
- 文件大、恢复速度慢、对性能影响大