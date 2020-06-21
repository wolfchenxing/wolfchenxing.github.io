---
title: Redis数据结构
date: 2020-04-26 23:48:10
tags: 
categories: Redis
---

# RedisObject

## 结构

```C
typedef struct redisObject {
    // 对象的数据结构，占4bits，共5种类型
    unsigned type:4;
    // 对象的编码类型，占4bits，共10种类型
    unsigned encoding:4;
    // 对象最后一次被访问的时间
    // 使用LRU算法计算相对server.lruclock的LRU时间，占24bits
    unsigned lru:LRU_BITS;
    // 引用计数，占4bytes
    int refcount;
    // 指向底层数据实现的指针，64位系统占8bytes
    void *ptr;
} robj;
```
![RedisObject结构](/images/Redis/RedisObject.jpg)

## 类型与编码

通过设置encoding属性来设定对象所使用的编码，而不是为特定对象关联一种固定的编码，极大提升了Redis的灵活性和效率。因为Redis可以根据不同的使用场景来为一个对象设置不同的编码，从而优化对象在某一个场景下的效率。

|    类型    |          编码          |            对象            |
| :--------: | :--------------------: | :------------------------: |
| OBJ_STRING |    OBJ_ENCODING_INT    |   整数值类型的字符串对象   |
| OBJ_STRING |  OBJ_ENCODING_EMBSTR   | EMBSTR字符串对象，少量数据 |
| OBJ_STRING |    OBJ_ENCODING_RAW    |       动态字符串对象       |
|  OBJ_LIST  | OBJ_ENCODING_QUICKLIST |   快速列表实现的列表对象   |
|  OBJ_HASH  |  OBJ_ENCODING_ZIPLIST  |    压缩表实现的哈希对象    |
|  OBJ_HASH  |    OBJ_ENCODING_HT     |     字典实现的哈希对象     |
|  OBJ_SET   |  OBJ_ENCODING_INTSET   |   整数集合实现的集合对象   |
|  OBJ_SET   |    OBJ_ENCODING_HT     |     字典实现的集合对象     |
|  OBJ_ZSET  |  OBJ_ENCODING_ZIPLIST  |  压缩表实现的有序集合对象  |
|  OBJ_ZSET  | OBJ_ENCODING_SKIPLIST  |  跳跃表实现的有序集合对象  |

<br/>

# 五种数据类型
Redis使用对象来表示数据库中的键值，每当在redis的数据库中创建一个键值对时，将会创建两个对象，即键对象和值对象。

## 字符串（String）
```C
struct sds {
    // buf中已占用空间的长度
    unsigned int len;
    // 内存大小
    unsigned int alloc;
    // 特殊标志位
    unsigned char flags;
    // 初始化sds分配的数据空间
    char buf[];
};
```
![String三种编码格式](/images/Redis/String.jpg)

如果一个字符串保存的是整数值，并且这个值可以用long类型表示，那么字符串对象会将整数值保存在字符串对象结构的ptr属性里。并将字符串编码encoding设置为int，1-9999之间的数值字符串在内存中可以复用。

embstr 存储形式是这样一种存储形式，它将 RedisObject 对象头和 SDS 对象连续存在一起，使用 malloc 方法一次分配。而 raw 存储形式不一样，它需要两次 malloc，两个对象头在内存地址上一般是不连续的。

在字符串比较小时，SDS 对象头的大小是capacity+3——SDS结构体的内存大小至少是 3。意味着分配一个字符串的最小空间占用为 19 字节 (16+3)。如果总体超出了 64 字节，Redis 认为它是一个大字符串，不再使用 emdstr 形式存储，而该用 raw 形式。而64-19-结尾的\0，所以empstr只能容纳44字节。

> 扩容策略：
字符串在长度小于 SDS_MAX_PREALLOC 之前，扩容空间采用加倍策略，也就是保留 100% 的冗余空间。当长度超过 SDS_MAX_PREALLOC 之后，为了避免加倍后的冗余空间过大而导致浪费，每次扩容只会多分配 SDS_MAX_PREALLOC 大小的冗余空间。SDS_MAX_PREALLOC 容量大小定义在sds.sh文件中，默认为1024*1024，即1M。

## 列表（List）

quicklist是在Redis 3.2之后出现的一种Redis底层数据结构用于List结构的具体实现。quickList 是 zipList 和 linkedList 的混合体，它将 linkedList 按段切分，每一段使用 zipList 来紧凑存储，多个 zipList 之间使用双向指针串接起来。

![quicklist结构](/images/Redis/quicklist.jpg)

### ziplist

ziplist是由一系列特殊编码的连续内存块组成的顺序存储结构，类似于数组，ziplist在内存中是连续存储的，但是不同于数组，为了节省内存 ziplist的每个元素所占的内存大小可以不同，每个节点可以用来存储一个整数或者一个字符串。

![ziplist结构](/images/Redis/ziplist.jpg)

- zlbytes: ziplist的长度（单位: 字节)，是一个32位无符号整数
- zltail: ziplist最后一个节点的偏移量，反向遍历ziplist或者pop尾部节点的时候有用。
- zllen: ziplist的节点（entry）个数
- entry: 节点
- zlend: 值为0xFF，用于标记ziplist的结尾

### entry

![ziplist-entry结构](/images/Redis/ziplist-entry.jpg)

- prevlengh: 记录上一个节点的长度，为了方便反向遍历ziplist
- encoding: 当前节点的编码规则
- data: 当前节点的值，可以是数字或字符串

> 连锁更新：
如果我们将一个长度大于等于 254 字节的新节点 new 设置为压缩列表的表头节点，由于previous entry length大小不够用(1->5B)，后面所有的节点可能都要重新分配内存大小。因为连锁更新在最坏情况下需要对压缩列表执行N 次空间重分配操作，而每次空间重分配的最坏复杂度为O(N)，所以连锁更新的最坏复杂度为O(N^2)。

### quicklist参数

- **list-max-ziplist-size**：取正值时表示quicklist节点ziplist包含的数据项，取负值表示按照占用字节来限定quicklist节点ziplist的长度
    - -5：每个quicklist节点上的ziplist大小不能超过64kb
    - -4：每个quicklist节点上的ziplist大小不能超过32kb
    - -3：每个quicklist节点上的ziplist大小不能超过16kb
    - -2：每个quicklist节点上的ziplist大小不能超过8kb（默认值）
    - -1：每个quicklist节点上的ziplist大小不能超过3kb
- **list-compress-depth**：list设计最容易被访问的是列表两端的数据，中间的访问频率很低。为此可配置对中间节点进行压缩（采用LZF-一种无损压缩算法），进一步节省内存。
    - 0：是个特殊值，表示都不压缩（默认值）
    - 1：表示quicklist两端各有1个节点不压缩，中间的节点压缩
    - 2：表示quicklist两端各有2个节点不压缩，中间的节点压缩

## 哈希（Hash）

### hashtable编码

```C
// 字典
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    // rehash索引；当rehash不在进行时，值为-1
    int rehashidx;
    // 目前正在运行的安全迭代器
    int iterators;
} dict;
//哈希表
typedef struct dict {
    //哈希表数组，数组的每个项是entry链表的头结点（链地址法解决哈希冲突）
    dictEntry **table;
    //哈希表大小
    unsigned long size;
    //哈希表大小掩码，用于计算索引值，总是等于size-1
    unsigned long sizemask;
    //该哈希表已有节点的数量
    unsigned long used;
} dictht;
// 哈希表节点
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    //指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```
![字典结构](/images/Redis/dict.jpg)

![hash-hashtable结构](/images/Redis/hash-hashtable.jpg)

Hashtable采用渐进式rehash，负载因子=已使用节点数量/哈希表大小，在两种情况下对哈希表进行扩展
- 当服务器未执行BGSAVE或BGRWRITEAOF，并且负载因子>1
- 当服务器执行BGSAVE或BGRWRITEAOF，并且负载因子>5

当负载因子<0.1对哈希表进行收缩

### ziplist编码

当哈希对象同时满足以下两个条件时，哈希对象使用ziplist编码：
- 哈希对象保存的所有键值元素的长度都小于hash-max-ziplist-value配置（默认64字节）
- 哈希对象保存的键值对数量小于hash-max-ziplist-entries配置（默认512个）

![hash-ziplist结构](/images/Redis/hash-ziplist.jpg)

## 集合（Set）

集合对象的编码可以是intset或hashtable

```C
typedef struct intset {
    // 每个整数的类型 int16/int32/int64
    uint32_t encoding;
    // intset长度
    uint32_t length;
    // 整数数组
    int8_t contents[];
} intset;
```

![set结构](/images/Redis/set.jpg)

当集合对象同时满足以下两个条件时，集合对象使用intset编码：
- 集合对象保存的所有元素都是整数值
- 哈希对象保存的键值对数量不超过set-max-intset-entries配置（默认512个）

## 有序集合（Sorted Set）

### ziplist编码

![zset-ziplist结构](/images/Redis/zset-ziplist.jpg)

当有序集合对象同时满足以下两个条件时，有序集合对象使用ziplist编码：
- 有序集合对象保存的所有成员的长度都小于zset-max-ziplist-value配置（默认64字节）
- 有序集合对象保存的所有元素数量小于zset-max-ziplist-entries配置（默认128个）

### skiplist编码

![zset-skiplist结构](/images/Redis/zset-skiplist.jpg)

跳表具有如下性质：
- 由很多层结构组成
- 每个节点的层数随机生成（Redis5.0版本ZSKIPLIST_MAXLEVEL为64）
- 每一层都是一个有序的链表
- 最底层(Level 1)的链表包含所有元素
- 如果一个元素出现在 Level i 的链表中，则它在 Level i 之下的链表也都会出现。
- 每个节点包含两个指针，一个指向同一链表中的下一个元素，一个指向下面一层的元素。

![skiplist结构](/images/Redis/skiplist.jpg)