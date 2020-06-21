---
title: Redis实用技巧
date: 2020-05-10 16:48:36
tags:
categories: Redis
---

# 慢查询日志

一条客户端命令的生命周期：

1. 命令发送
2. 命令排队
3. 命令执行
4. 返回结果

慢查询日志就是系统在命令执行前后计算每条命令的执行时间，当超过预设阈值，就将这条命令的相关信息（如发生时间、耗时、命令的详细信息）记录下来。慢查询只记录命令执行时间，并不包括命令排队和网络传输时间。因为命令执行排队机制，慢查询会导致其他命令级联阻塞，因此当客户端出现请求超时，需要检查该时间点是否有对应的慢查询。

## 数据结构

每个慢查询日志有4个属性组成，分别是慢查询日志的标识id、发生时间戳、命令耗时、执行命令和参数。

![慢查询日志数据结构](/images/Redis/慢查询日志数据结构.jpg)

## 参数设置

### slowlog-log-slower-than

命令执行时长超过指定阈值时将被记入到慢查询日志中。单位是微妙，默认值10000。0：记录所有命令，<0：对所有命令都不记录。

由于Redis采用单线程相应命令，对于高流量场景，如果命令执行时间在1毫秒以上，那么Redis最多可支撑OPS不到1000。因此对于高OPS场景的Redis建议设置为1毫秒。

### slowlog-max-len

慢查询日志最多存储条数，Redis使用列表存储慢日志。当慢查询日志列表已处于其最大长度，最早插入的一个命令将从列表移除。

由于慢查询日志是一个先进先出的队列，也就是说如果慢查询比较多的情况下，可能会丢失部分慢查询命令。为防止这种情况发生，可以定期执行slow get命令将慢查询日志持久化到其他存储中。

## 相关命令

- slowlog get [n]：获取慢查询日志列表，可指定返回条数
- slowlog len：获取慢查询日志列表当前的长度
- slowlog reset：清空慢查询日志列表


# Redis Shell

| 可执行文件     | 作用     |
| :--- | ---- |
| redis-server     | 启动redis服务     |
| redis-cli     | redis命令行工具 |
| redis-benchmark | 基准测试工具 |
| redis-check-aof | AOF持久化文件检测工具和修复 |
| redis-check-dump | RDB持久化文件检测工具和修复 |
| redis-sentinel | 启动redis-sentinel |

## redis-cli

- -r（repeat）选项代表将命令执行多次
- -i（interval）选项代表每隔几秒执行一次命令，但是-i选项必须和-r选项一起使用
redis-cli -r 100 -i 1 info | grep used_memory_human
- -c（cluster）选项是连接Redis Cluster节点时需要使用的，-c选项可以防止moved和ask异常
- 如果Redis配置了密码，可以用-a（auth）选项，有了这个选项就不需要手动输入auth命令。
- redis-cli.exe --scan --pattern a*，扫描指定pattern格式的key
- --slave选项是把当前客户端模拟成当前Redis节点的从节点，可以用来获取当前Redis节点的更新操作
- --rdb选项会请求Redis实例生成并发送RDB持久化文件，保存在本地。
- --pipe选项用于将命令封装成Redis通信协议定义的数据格式，批量发送给Redis执行
- --bigkeys选项使用scan命令对Redis的键进行采样，从中找到内存占用比较大的键值，这些键可能是系统的瓶颈。
- --eval选项用于执行指定Lua脚本
- --latency可以测试客户端到目标Redis的网络延迟
- --latency的执行结果只有一条，如果想以分时段的形式了解延迟信息，可以使用--latency-history
- --latency-dist该选项会使用统计图表的形式从控制台输出延迟统计信息。
- --stat选项可以实时获取Redis的重要统计信息，虽然info命令中的统计信息更全，但是能实时看到一些增量的数据（例如requests）对于Redis的运维还是有一定帮助的
- --no-raw选项是要求命令的返回结果必须是原始的格式，--raw恰恰相反，返回格式化后的结果

## redis-server

redis-server除了启动Redis外，还有一个--test-memory选项。redis-server --test-memory可以用检测当前操作系统能否稳定地分配指定容量的内存给Redis，通过这种检测可以有效避免因为内存问题造成Redis崩溃，该功能更偏向于调试和测试。
```
redis-server --test-memory 256
```

## redis-benchmark

- -c（clients）选项代表客户端的并发数量（默认是50）。
- -n（num）选项代表客户端请求总量（默认是100000）。
- -q选项仅仅显示redis-benchmark的requests per second信息。
- 用-r（random）选项，可以向Redis插入随机的键。-r选项会在key、counter键上加一个12位的后缀，-r10000代表只对后四位做随机处理（-r不是随机数的个数）。
- -t选项可以对指定命令进行基准测试。
- --csv选项会将结果按照csv格式输出，便于后续处理，如导出到Excel等。
```
redis-benchmark -c 100 -n 20000 -q
redis-benchmark -c 100 -n 20000 -r 10000
redis-benchmark -t get,set -q
```


# Pipeline

RTT(Round-Trip Time): 往返时间。在计算机网络中它是一个重要的性能指标，表示从发送端发送数据开始，到发送端收到来自接收端的确认（接收端收到数据后便立即发送确认），总共经历的时间。RTT在不同网络环境下会有不同，例如同机房和同机器会比较快，跨机房跨地区会比较慢。

Pipeline（流水线）机制能改善上面这类问题，它能将一组Redis命令进行组装，通过一次RTT传输给Redis，再将这组Redis命令的执行结果按顺序返回给客户端。

![Pipeline命令模型](/images/Redis/Pipeline命令模型.jpg)

**原生批量命令（mget/mset/hmget/hmset） VS Pipeline：**

- 原生批量命令是原子的，Pipeline是非原子的。
- 原生批量命令是一个命令对应多个key，Pipeline支持多个命令。
- 原生批量命令是Redis服务端支持实现的，而Pipeline需要服务端和客户端的共同实现。

**使用须知:**
Pipeline虽然好用，但是每次Pipeline组装的命令个数不能没有节制，否则一次组装Pipeline数据量过大，一方面会增加客户端的等待时间，另一方面会造成一定的网络阻塞，可以将一次包含大量命令的Pipeline拆分成多次较小的Pipeline来完成。


# 事务

Redis提供了简单的事务功能，将一组需要一起执行的命令放到multi和exec两个命令之间。multi命令代表事务开始，exec命令代表事务结束。它们之间的命令是原子顺序执行的。

## Redis事务的三个阶段

1. **事务开始**
MULTI 命令标志着事务的开始，通过将在客户端状态的flags属性中打开CLIENT_MULTI标识来完成从非事务状态切换至事务状态。

2. **命令入队**
客户端发送exec、discard、watch、multi四个命令时，服务器立即执行。
客户端发送的除此4个命令之外的其他命令时，服务器将命令放入事务队列（FIFO），然后向客户端返回queued。

3. **事务执行**
处于事务状态的客户端向服务器发送exec命令时，这个命令将被服务器立即执行。服务器首先会遍历这个客户端的事务队列，执行队列中保存的所有命令，最后将执行命令所得的结果全部返回客户端。

![Redis事务](/images/Redis/Redis事务.jpg)

## 错误处理

- **入队错误**
如果一个事务在入队命令的过程中出现了命令不存在或格式不正确等情况，那么Redis将拒绝执行这个事务，直接退出事务。

- **执行错误**
执行命令过程中发生的错误都是一些不能在入队时被服务器发现的错误，这些错误只会在命令执行时被触发。
执行命令过程中发生了错误，服务器也不会中断事务的执行，它会继续执行事务中余下的命令，并且已执行的命令不会被出错的命令影响。

## Watch命令

watch命令是一个乐观锁，它可以在exec命令执行之前，监视任意数量的数据库键，并在exec命令执行时，检查被监视的键是否至少有一个已经被修改过，如果是的话，服务器将拒绝执行事务，并向客户端返回代表事务执行失败的空回复。

每个Redis数据库都有一个watched_keys字典保存被监视的键，字典的key为被监视的键，字典的值是一个链表，记录了所有监视相应数据库键的客户端。

![Watch数据结构](/images/Redis/Watch数据结构.jpg)

![Watch流程图](/images/Redis/Watch流程图.jpg)

1. watched_keys字典监视的键被修改后，服务器会将对应客户端的标识位置为CLIENT_DIRTY_CAS。
2. 服务端收到exec执行命令，如果当前客户端的标识位是CLIENT_DIRTY_CAS，则拒绝执行，否则提交事务

> Redis不支持事务中的回滚，是因为Redis作者认为这种复杂的功能和Redis最求的见到高效的设计主旨不符，并且Redis事务的执行时错误通常都是编程错误产生的，这种错误通常只会出现在开发环境中，而很少在实际的生产环境中出现，所以没有必要为Redis开发事务回滚功能。

# Bitmaps

Bitmaps本身不是一种数据结构，实际上它就是字符串，但是它可以对字符串的位进行操作。Redis提供setbit、getbit、bitcount、bitop、bitpos五个个命令处理二进制数组位，数组的下标在Bitmaps中叫做偏移量。

![Bitmaps](/images/Redis/Bitmaps.jpg)

## 命令
（1）getbit key <offset>
1. 计算byte = offset / 8,该值记录了offset偏移量指定的二进制位保存在位数组的哪个字节。
2. 计算bit = (offset mod 8)+1,bit值记录了offset偏移量指定的二进制位是byte字节的第几个二进制位。
3. 根据byte与bit返回位数组中指定的二进制位的值。

（2）setbit key <offset> <value>
1. 计算所需位数组长度，len=(offset / 8)+1，len记录了保存offset指定的二进制位至少需要多少字节
2. 检查key保存的位数组长度是否小于len，小于则扩展字符串长度位len字节，并将扩展的二进制位的值设置位0
3. 计算byte = offset / 8,该值记录了offset偏移量指定的二进制位保存在位数组的哪个字节
4. 计算bit = (offset mod 8)+1,bit值记录了offset偏移量指定的二进制位是byte字节的第几个二进制位。
5. 根据byte与bit设置位数组中指定的二进制位的值。
6. 向客户端返回原值

（3）bitcount key
统计字符串二进制码中，有多少个1
bitcount key [start][end]

（4）bitop
bitop op destkey key [key....]

bitop是一个复合操作，它可以做多个Bitmaps的and（交集）、or（并集）、not（非）、xor（异或）操作并将结果保存在destkey中。

（5）bitpos
bitpos key targetBit [start] [end]
bitpos有两个选项[start]和[end]，分别代表起始字节和结束字节，返回字符串里面第一个被设置为1或者0的bit位。

## 应用场景

Bitmaps适用于进行快速、简单、实时统计，包括独立访客，日活用户的统计，并且Bitmaps极其节省空间。

**独立访客和日活用户统计方案：**
- 设置值，setbit key offset value，每个独立用户是否访问过网站存放在Bitmaps中，将访问的用户记做1，没有访问的用户默认为0，用偏移量作为用户的id
- 获取值，getbit key offset，获取bitmaps当前偏移量的值，判断用户是否已访问
- 获取指定范围值为1的个数，bitcount [start] [end]，对bitmaps中所有偏移量值为1的个数统计已访问用户总量
- bitmaps间运算，bitop op destkey key，如通过and操作符统计连续n天访问网站的用户数量

假设网站有1亿用户，每天独立访问的用户有5千万，如果每天用集合类型和Bitmaps分别存储活跃用户，一天存储量对比如下

| 数据类型 | 每个用户id占用空间      | 需要存储的用户量 | 消耗内存 |
| -------- | ----------------------- | ---------------- | -------- |
| Set集合  | 64位（Long类型占8字节） | 50,000,000       | 约400M   |
| Bitmaps  | 1位                     | 100,000,000      | 约12.5M  |


# HyperLogLog

HyperLogLog是一种基数算法，实际类型为字符串，通过HyperLogLog可以利用极小的内存空间完成总数的统计。HyperLogLog提供不精确的去重技术方案，标准误差是0.81%。

在有些情况下， 我们只想要知道在线用户的人数， 而不需要知道具体的在线用户名单， 这时bitmap和集合储存的信息就会显得多余了。在需要尽可能地节约内存并且只需要知道在线用户数量的情况下， 可以使用 HyperLogLog 来对在线用户进行统计： HyperLogLog 是一个概率算法， 它可以对元素的基数进行估算， 并且每个 HyperLogLog 只需要耗费 12 KB 内存， 对于用户数量非常多但是内存却非常紧张的系统， 这一方案无疑是最佳之选。

因此在选用HyperLogLog 时需要明确两点：
1. 只为了计算独立总数，不需要获取单条数据。
2. 可以容忍一定误差率，毕竟HyperLogLog在内存的占用量上有很大的优势。

**命令**
1. 添加
pfadd key element [element … ]，添加成功返1，失败返回0

2. 统计数量
pfcount key [key …]，返回数量

3. 合并
pfmerge destkey sourcekey [sourcekey ...]


# 发布订阅

Redis提供了基于“发布/订阅”模式的消息机制，此种模式下，消息发布者和订阅者不进行直接通信，发布者客户端向指定的频道（channel）发布消息，订阅该频道的每个客户端都可以收到该消息。实现系统之间的解耦。若无消费者订阅，则消息丢失。

## 命令
- 发布消息
publish channel message
- 订阅消息
subscribe channel [channel ...]
客户端在执行订阅命令之后进入了订阅状态，只能接收subscribe、psubscribe、unsubscribe、punsubscribe的四个命令。
新开启的订阅客户端，无法收到该频道之前的消息，因为Redis不会对发布的消息进行持久化。
- 取消订阅
unsubscribe [channel [channel ...]]
- 按照模式订阅和取消订阅
psubscribe pattern [pattern...]
punsubscribe [pattern [pattern ...]]
- 查询订阅
查看活跃的频道，pubsub channels [pattern]
查看频道订阅数，pubsub numsub [channel ...]
查看模式订阅数，pubsub numpat

## 订阅实现

Redis将所有的订阅关系都保存在服务器的名为pubsub_channels的字典表内，字典表的键为频道名，值为链表，表中保存客户端信息。

![普通订阅实现](/images/Redis/普通订阅实现.jpg)

模式订阅与与普通订阅原理一致，字典表为pubsub_patterns

![模式订阅实现](/images/Redis/模式订阅实现.jpg)

- 频道没有订阅者，则先在字典中为新的频道创建一个键，并将这个键的值设置为空链表，然后在将客户端添加到链表中，成为链表的第一个元素。
- 频道已经有订阅者，那么它在pubsub_channels字典中必然有相应的订阅者链表，只要将客户端加到订阅者链表的末尾。

## 退订实现

client-10086退订 “news.sport”,"news.movie"

![退订实现](/images/Redis/退订实现.jpg)

- 程序会根据被退订频道的名字， 在 pubsub_channels 字典中找到频道对应的订阅者链表， 然后从订阅者链表中删除退订客户端的信息。
- 如果删除退订客户端之后， 频道的订阅者链表变成了空链表， 那么说明这个频道已经没有任何订阅者了， 程序将从 pubsub_channels字典中删除频道对应的键。

## 发布实现

在 pubsub_channels字典里找到对应的频道，将消息发送给所有的订阅者。


# GEO

Redis3.2版本提供了GEO（地理信息定位）功能，支持存储地理位置信息用来实现诸如附近位置、摇一摇这类依赖于地理位置信息的功能。

GEO的实现主要包含以下两项技术：
- 使用geohash存储地理位置的坐标
- 使用有序集合（zset）存储地理位置的集合

## geohash计算步骤
1) 首先将经度范围(-180, 180)平分成两个区间(-180,0)、(0, 180)，如果目标经度位于前一个区间，则编码为0，否则编码为1。经度的编码长度为26位。
2) 用同样的方法将纬度范围(-85.05112878,85.05112878)平分成两个区间(-85.05112878,0)、(0, 85.05112878)。如果目标纬度位于前一个区间，则编码为0，否则编码为1。纬度的编码长度为26位。
3) 接下来将经度和纬度的编码合并，奇数位是纬度，偶数位是经度，得出52位的二进制经纬度值
4) 将二进制经纬度转换成的整形数值保存到zset有序集合中，score为geohash的52整形值，member为命令中的成员。

## geohash特点
- GEO的数据类型为zset，Redis将所有地理位置信息的geohash存放在zset中。
- 字符串越长表示的位置更精确。Redis使用的geohash编码长度为26位。可以精确到0.59m的精度。
- 两个字符串越相似，它们之间的距离越近，Redis利用字符串前缀匹配算法实现相关的命令。
- geohash编码和经纬度是可以相互转换的。

## 命令
（1）增加地理位置信息
geoadd key longitude latitude member [longitude latitude member ...]
- 有效的经度是[-180,180]
- 有效的纬度是[-85.05112878,85.05112878]

（2）获取地理位置信息
geopos key member [member ...]

（3）获取两个地理位置的距离
geodist key member1 member2 [unit]

其中unit代表返回结果的单位，包含以下四种：
- m（meters）代表米。
- km（kilometers）代表公里。
- mi（miles）代表英里。
- ft（feet）代表尺。

（4）获取指定位置范围内的地理信息位置集合
georadius key longitude latitude radiusm|km|ft|mi [withcoord] [withdist]
[withhash] [COUNT count] [asc|desc] [store key] [storedist key]

georadiusbymember key member radiusm|km|ft|mi [withcoord] [withdist]
[withhash] [COUNT count] [asc|desc] [store key] [storedist key]

- withcoord：返回结果中包含经纬度。
- withdist：返回结果中包含离中心节点位置的距离。
- withhash：返回结果中包含geohash，有关geohash后面介绍。
- COUNT count：指定返回结果的数量。
- asc|desc：返回结果按照离中心节点的距离做升序或者降序。
- store key：将返回结果的地理位置信息保存到指定键。
- storedist key：将返回结果离中心节点的距离保存到指定键。

（5）获取geohash
geohash key member [member ...]

从指定member中读取geohash整形值转为52位二进制数据，然后进行base32编码。

![geohash_base32](/images/Redis/geohash_base32.jpg)

该命令最终将返回11个字符的Geohash字符串。

（6）删除地理位置信息
zrem key member