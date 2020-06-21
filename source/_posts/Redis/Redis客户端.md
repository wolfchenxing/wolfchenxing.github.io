---
title: Redis客户端
date: 2020-05-20 21:24:49
tags:
categories: Redis
---

# Jedis

## 版本
```xml
<!-- https://mvnrepository.com/artifact/redis.clients/jedis -->
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
```

## 连接池JedisPool

![JedisPool](/images/Redis/JedisPool.jpg)

Jedis提供了JedisPool这个类作为对Jedis的连接池，同时使用了Apache的通用对象池工具common-pool作为资源的管理工具。

## Jedis直连与连接池对比

|    | 优点    | 缺点        |
| -- | ------ | :---------- |
| 直连 | 简单方便，适用于少量长期连接的场景 | 1）存在每次新建/关闭TCP连接开销<br>2）资源无法控制，极端情况会出现连接泄漏<br>3）Jedis对象线程不安全 |
| 连接池 | 1）无需每次连接都生成Jedis对象，降低开销<br>2）使用连接池的形式保护和控制资源的使用 | 相对于直连，使用相对麻烦，尤其在资源的管理上需要很多参数来保证，一旦规划不合理也会出现问题 |

## 问题

**（1）场景一：从连接池中取出Jedis，操作完成后未及时还回连接池**

连接池中的资源时有限的，默认连接数只有8个，如果处理完成后不及时归还，并且有大量的请求时，连接资源会很快被消耗完。当超过maxTotal值后调用者所在线程将会阻塞直到有连接还回连接池。

**解决方案：**
- 良好的编码习惯，用完即调用Jedis.close()归还连接。
- 请求量过大导致调用者所在线程阻塞，可以通过设置blockWhenExhausted=true并且设置maxWaitMillis指定最大等待时间，超过该值后将解除阻塞。

**（2）场景二：连接空闲时间过长导致连接被关闭**

连接空闲时间大于客户端设置的timeout时间后，Redis服务器会强制关闭客户端的连接，但是连接池无法获知消息删除无效的连接。如果再从连接池中取出连接进行处理时会抛出异常。

**解决方案：**
- 设置testOnBorrow为true，每次从连接池取连接时会做连接有效性的测试（ping），无效连接会被移除，每次取连接时多执行一次ping命令。
- 通过设置minEvictableIdleTimeMillis、timeBetweenEvictionRunsMillis定期清理失效连接。minEvictableIdleTimeMillis=1000、timeBetweenEvictionRunsMillis=2000 表示后台线程每2s会执行一次清理任务，将空闲时间>1s的连接移除。

## 连接池的配置

![JedisPool配置](/images/Redis/JedisPool配置.jpg)


# 客户端属性

Redis服务器与客户端连接成功后，都会创建一个client对象保存道歉客户端的信息。可通过输入`client list`命令查看：
```shell
127.0.0.1:6379> client list
id=3 addr=127.0.0.1:45820 fd=7 name= age=16 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=26 qbuf-free=32742 obl=0 oll=0 omem=0 events=r cmd=client
```
输出每行命令都是代表一个客户端信息，每行中都有十几个属性，它们是客户端的执行状态，理解这些属性对于Redis的开发和运维非常有帮助。
- addr ： 客户端的地址和端口
- fd ： 套接字所使用的文件描述符，普通客户端的fd大于-1。伪客户端的fd是-1，目前Redis服务器会在两个地方用到伪客户端，一个用于载入AOF文件还原数据，另一个用于执行Lua脚本中包含的Redis命令
- age ： 以秒计算的已连接时长
- idle ： 以秒计算的空闲时长
- flags ： 客户端 flag
- db ： 该客户端正在使用的数据库 ID
- sub ： 已订阅频道的数量
- psub ： 已订阅模式的数量
- multi ： 在事务中被执行的命令数量
- qbuf ： 查询缓冲区的长度（字节为单位， 0 表示没有分配查询缓冲区）
- qbuf-free ： 查询缓冲区剩余空间的长度（字节为单位， 0 表示没有剩余空间）
- obl ： 输出缓冲区的长度（字节为单位， 0 表示没有分配输出缓冲区）
- oll ： 输出列表包含的对象数量（当输出缓冲区没有剩余空间时，命令回复会以字符串对象的形式被入队到这个队列里）
- omem ： 输出缓冲区和输出列表占用的内存总量，单位byte
- events ： 文件描述符事件，r/w分别代表客户端套接字可读和科协
- cmd ： 最近一次执行的命令，不含参数

## 输入缓冲区：qbuf、qbuf-free

Redis为每个客户端分配了输入缓冲区，作用是将客户端发送的命令临时保存，同时会从输入缓冲区读取命令并执行，它为Redis服务器提供了缓冲功能。

Redis无法配置输入缓冲区的大小，输入缓冲区会根据输入内容大小的不同动态调整，但是总体大小不能超过1G。

**输入缓存区使用不当产生的问题:**
- 客户端的输入缓冲区超过1G，客户端将会被强制关闭
- 输入缓冲区不受maxmemory控制，如果redis实例的maxmemory设置了1G，已经存储800M数据， 如果此时输入缓冲区使用了500M，将总内存将超过maxmemory，可能会产生数据丢失、键值淘汰、OOM等情况

**产生的原因：**
- redis处理速度跟不上输入缓冲区的输入速度，例如存在bigkey，慢查询等原因导致命令执行的时间变长。
- 写入命令量非常大，但此时redis服务器在执行持久化导致阻塞无法处理命令，导致命令大量积压在输入缓存区中。

**解决方案：**
- `client list`查看qbuf、qbuf-free来定位存在问题的客户端，分析原因加以处理
- `info clients`定期监控client_biggest_input_buf，设置预警阀值超过时发送报警邮件或短信 

## 命令参数

在服务器将客户端发送deep命令请求保存到输入缓冲区后，服务器对命令请求的内容进行解析得出命令参数以及命令参数的个数分别保存到客户端的argv和argc中。

**client对象属性：**
- argv是一个数组，数组中的每项都是一个字符串对象，其中argv[0]是命令，之后的其他项是传给命令的参数。
- argc负责记录argv数组的长度。
- cmd命令表字典结构，key是命令的名字，值是redisCommand对象。redisCommand中有命令实现方法、命令接收的参数个数、命令的总执行次数和总消耗时长等统计信息。命令表的查找操作不区分输入字母的大小写。

![client对象属性](/images/Redis/client对象属性.jpg)

## 输出缓冲区：obl、oll、omem

Redis为每个客户端分配了输出缓冲区，作用是保存命令执行的结果返回给客户端，为Redis与客户端交互返回结果提供了缓冲功能。

输出缓冲区的容量可以通过参数`client-output-buffer-limit`来进行设置。输出缓冲区不受maxmemory控制，如果redis使用内存总量+输出缓冲区的容量大于maxmemory时，会产生数据丢失、键值淘汰、OOM等情况。

**配置：**
```shell
client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```
- <class>：客户端类型，分为三种。a）normal：普通客户端；b）slave：slave客户端，用于复制；c）pubsub：发布订阅客户端。
- <hard limit>：如果客户端使用的输出缓冲区大于<hard limit>，客户端会被立即关闭。
- <soft limit>和<soft seconds>：如果客户端使用的输出缓冲区超过了<soft limit>并且持续了<soft limit>秒，客户端会被立即关闭。

**输出缓冲区按客户端的不同分为三种：普通客户端、发布订阅客户端、slave客户端。**

![三种客户端输出缓冲区](/images/Redis/三种客户端输出缓冲区.jpg)

**输出缓冲区有两部分组成：固定缓冲区（16KB）和动态缓冲区**
- 固定缓冲区用于保存那些长度比较小的回复，比如OK、简短的字符串值、整数值、错误回复等。
- 可变大小的缓冲区用于保存那些长度比较大的回复，比如一个非常长的字符串值，一个有很多元素组成的集合或列表。

固定缓冲区使用的是字节数组，动态缓冲区使用的是链表。当固定缓冲区存满后会将Redis新的返回结果存放在动态缓冲区的队列中，队列中的每个对象就是每个返回结果。

![client对象结构](/images/Redis/client对象结构.jpg)

![输出缓冲区](/images/Redis/输出缓冲区.jpg)

**解决方案：**
- 通过定期执行client list命令，收集obl、oll、omem找到异常的连接记录并分析，最终找到可能出问题的客户端。
- info clients定期监控client_longest_output_list代表输出缓冲区列表最大对象数，设置预警阀值超过时发送报警邮件或短信 。
- 合理配置普通客户端输出缓冲区的大小。
- 如果master节点写入较大，适当增大slave的输出缓冲区的，slave客户端的输出缓冲区可能会比较大，一旦slave客户端连接因为输出缓冲区溢出被kill，会造成复制重连。
- 限制容易让输出缓冲区增大的命令，例如，高并发下的monitor命令就是一个危险的命令。
- 及时监控内存，一旦发现内存抖动频繁，可能就是输出缓冲区过大。

## 客户端的存活状态

client list中的age和idle分别代表当前客户端已经连接的时间和最近一次的空闲时间。当age等于idle时，说明连接一直处于空闲状态。

## 客户端的限制maxclients和timeout

maxclients参数来限制最大客户端连接数，一旦连接数超过maxclients，新的连接将被拒绝。
maxclients默认值是10000，可以通过info clients来查询当前Redis的连接数。

某些情况由于业务方使用不当（例如没有主动关闭连接）可能存在大量idle连接，因此Redis提供了timeout（单位为秒）参数来限制连接的最大空闲时间合理使用有限的资源，一旦客户端连接的idle时间超过了timeout，连接将会被关闭。
Redis的默认配置给出的timeout=0，客户端不会因超时而关闭。

## 客户端类型

![客户端类型](/images/Redis/客户端类型.jpg)

## 客户端监控

```shell
127.0.0.1:6379> info clients
# Clients
connected_clients:1
client_recent_max_input_buffer:2
client_recent_max_output_buffer:0
blocked_clients:0
```

- connected_clients：代表当前Redis节点的客户端连接数，需要重点监控，一旦超过maxclients，新的客户端连接将被拒绝。
- client_longest_output_list：当前所有输出缓冲区中队列对象个数的最大值。
- client_biggest_input_buf：当前所有输入缓冲区中占用的最大容量。
- blocked_clients：正在执行阻塞命令（例如blpop、brpop、brpoplpush）的客户端个数。
- total_connections_received(info stats)：Redis自启动以来处理的客户端连接数总数。
- rejected_connections：Redis自启动以来拒绝的客户端连接数，需要重点监控。