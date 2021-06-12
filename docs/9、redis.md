[TOC]

### C语言基础

```
#include <stdio.h>就是一条预处理命令，它的作用是通知C语言编译系统在对C程序进行正式编译之前需做一些预处理工作
一个C程序有且只有一个主函数，即main函数。主函数就是C语言中的唯一入口

```

### 源码阅读步骤

#### 1、阅读数据结构的实现

| 文件                                                         | 内容                        |
| :----------------------------------------------------------- | :-------------------------- |
| `sds.h` 和 `sds.c`                                           | Redis 的动态字符串实现。    |
| `adlist.h` 和 `adlist.c`                                     | Redis 的双端链表实现。      |
| `dict.h` 和 `dict.c`                                         | Redis 的字典实现。          |
| `redis.h` 中的 `zskiplist` 结构和 `zskiplistNode` 结构，<br /> 以及 `t_zset.c` 中所有以 `zsl` 开头的函数， 比如 `zslCreate` 、 `zslInsert` 、 `zslDeleteNode` ，等等。 | Redis 的跳跃表实现。        |
| `hyperloglog.c` 中的 `hllhdr` 结构， 以及所有以 `hll` 开头的函数。 | Redis 的 HyperLogLog 实现。 |

#### 2、阅读内存编码结构实现

这些结构是redis为了节约内存而专门开发而来的，但基本上都和内存、指针操作、位操作这些底层的东西相关

各个内存编码数据结构的实现文件：

| 文件                       | 内容                           |
| :------------------------- | :----------------------------- |
| `intset.h` 和 `intset.c`   | 整数集合（intset）数据结构。   |
| `ziplist.h` 和 `ziplist.c` | 压缩列表（zip list）数据结构。 |

#### 3、阅读数据类型的实现

以上两步介绍的redis六种不同类型的键的所有底层实现的结构，这一步是为了了解Redis是如何通过以上提到的数据结构来实现不同的键，来阅读实现数据类型的键的文件，以及Redis的对象系统文件

| 文件                                             | 内容                           |
| :----------------------------------------------- | :----------------------------- |
| `object.c`                                       | Redis 的对象（类型）系统实现。 |
| `t_string.c`                                     | 字符串键的实现。               |
| `t_list.c`                                       | 列表键的实现。                 |
| `t_hash.c`                                       | 散列键的实现。                 |
| `t_set.c`                                        | 集合键的实现。                 |
| `t_zset.c` 中除 `zsl` 开头的函数之外的所有函数。 | 有序集合键的实现。             |
| `hyperloglog.c` 中所有以 `pf` 开头的函数。       | HyperLogLog 键的实现。         |

#### 4、阅读数据库相关实现源码

| 文件                                                   | 内容                             |
| :----------------------------------------------------- | :------------------------------- |
| `redis.h` 文件中的 `redisDb` 结构， 以及 `db.c` 文件。 | Redis 的数据库实现。             |
| `notify.c`                                             | Redis 的数据库通知功能实现代码。 |
| `rdb.h` 和 `rdb.c`                                     | Redis 的 RDB 持久化实现代码。    |
| `aof.c`                                                | Redis 的 AOF 持久化实现代码。    |

选读

Redis 有一些独立的功能模块， 这些模块可以在完成第 4 步之后阅读， 它们包括：

| 文件                                                         | 内容                                            |
| :----------------------------------------------------------- | :---------------------------------------------- |
| `redis.h` 文件的 `pubsubPattern` 结构，以及 `pubsub.c` 文件。 | 发布与订阅功能的实现。                          |
| `redis.h` 文件的 `multiState` 结构以及 `multiCmd` 结构， `multi.c` 文件。 | 事务功能的实现。                                |
| `sort.c`                                                     | `SORT` 命令的实现。                             |
| `bitops.c`                                                   | `GETBIT` 、 `SETBIT` 等二进制位操作命令的实现。 |

#### 5、阅读客户端与服务器的相关代码

| 文件                                                         | 内容                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `ae.c` ，以及任意一个 `ae_*.c` 文件（取决于你所使用的多路复用库）。 | Redis 的事件处理器实现（基于 Reactor 模式）。                |
| `networking.c`                                               | Redis 的网络连接库，负责发送命令回复和接受命令请求， 同时也负责创建/销毁客户端， 以及通信协议分析等工作。 |
| `redis.h` 和 `redis.c` 中和单机 Redis 服务器有关的部分。     | 单机 Redis 服务器的实现。                                    |

#### 6、阅读多机功能的实现

| 文件            | 内容                        |
| :-------------- | :-------------------------- |
| `replication.c` | 复制功能的实现代码。        |
| `sentinel.c`    | Redis Sentinel 的实现代码。 |
| `cluster.c`     | Redis 集群的实现代码。      |

### 数据结构与对象

#### 1、简单动态字符串

```c
/*
 * 保存字符串对象的结构
 */
struct sdshdr {
    
    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};
```

```c
Redis只会使用C字符串作为字面量，在大多数情况下，Redis使用SDS（Simple Dynamic String，简单动态字符串）作为字符串表示。
·比起C字符串，SDS具有以下优点：

1）常数复杂度获取字符串长度
	通过访问SDS的len属性即可，复杂度仅为O(1)。

2）杜绝缓冲区溢出。
	SDS的API需要对SDS进行修改时，API会先检查SDS空间是否满足，不满足的话，自动将SDS的空间拓展到需要的大小
	
3）减少修改字符串长度时所需的内存重分配次数。
	通过未使用空间，SDS实现了空间预分配和惰性空间释放两种优化策略
		【空间预分配】：空间拓展的时候，不仅会为SDS分配修改所需要的空间，还会为SDS分配额外的未使用空间
		公式如下：
			如果SDS修改之后，长度将小于1MB，那么程序分配和Len属性同样大小的未使用空间
			如果修改之后，将变成13，那么SDS的buf数组实际长度将变成13+13+1（空字符串）=27字节
			
			如果>=1MB,那将多分配1MB的未使用空间：
			如果修改之后，将变成2MB，那么SDS的buf数组长度将变成 2MB+1MB+1byte
			
		【惰性空间释放】
			用于优化SDS的字符串缩短操作
			当SDS的API需要缩短SDS的字符串长度的时候，程序不会立即回收多出来的字节，而是用free来讲这些字节的数量记录起来，待将来使用
			
			注意：
			SDS也提供了响应的API，在需要的时候，真正的释放SDS未使用的空间，这样不懂担心内存的浪费
	
4）二进制安全。
	C的字符串必须符合某种编码，只能尾部有空字符，其他地方不能有，否则的会被程序误认为的字符串的结尾，所有c字符串只能保持文本，不能保持视频、图片这样的二进制文件
	Redis为了使用各种不同的使用场景，SDS的API都是二进制（binary-safe）安全的,SDS API都会以处理二进制的方式来处理SDS存储在buf数组里的数据，程序不会对其中的数据做任务限制、过滤、或者假设，保持读写字符串时，不损害其内容，数据写入的时候和读取的时候是一样的。
    
	这也是SDS的buf属性是数组的原因，因为Redis不是用这个数据来保持字符，而是来保存一系列的二进制数据
	
5）兼容部分C字符串函数。
	SDS遵循c字符串以空字符结尾的惯例，保存空字符串的1字节空间不计算在SDS的len属性里面，并且为空字符分配额外的1字节空间，添加到字符串的尾部这些操作都是SDS函数自动完成的，对使用来说就是完全同名的。
	为什么？
	SDS可以重用一部分C字符串函数库中的函数
```

#### 2、链表（双端链表）

```c
/*
 * 双端链表节点
 */
typedef struct listNode {

    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;

} listNode;

/*
 * 双端链表迭代器
 */
typedef struct listIter {

    // 当前迭代到的节点
    listNode *next;

    // 迭代的方向
    int direction;

} listIter;

/*
 * 双端链表结构
 */
typedef struct list {

    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);

    // 链表所包含的节点数量
    unsigned long len;

} list;
```

- 链表被广泛用于实现Redis的各种功能：列表键、发布与订阅、慢查询、监视器等。
- 每个链表节点是由listNode来表示，由前置和后置节点的指针，所以redis的链表实现是个双端链表
- 每个链表使用一个list结构来表示，由表头、表尾指针以及链表长度
- 每个链表表头节点的前置指针、表尾节点的后置节点都执行null，所以redis的链表实现是个无环链表
- 通过为链表设置不同的类型特定函数，redis的链表可以保存各种不同类型的值

#### 3、字典

```c
/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
/*
 * 哈希表
 *
 * 每个字典都使用两个哈希表，从而实现渐进式 rehash 。
 */
typedef struct dictht {
    
    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;
    
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht;

/*
 * 哈希表节点
 */
typedef struct dictEntry {
    
    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;

} dictEntry;

/*
 * 字典
 */
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

    // 目前正在运行的安全迭代器的数量
    int iterators; /* number of iterators currently running */

} dict;
```

**注：**

**dict中的ht属性是一个包含两个项的数组，字典只使用ht[0]哈希表，ht[1]哈希表只有在rehash时使用**



##### **rehash**

拓展和收缩哈希表的工作可以通过rehash操作来完成，

- 为ht[1]哈希表分配空间，大小取决于ht[0]的键值对数量
  - 如果是拓展，ht[1]的大小为第一个大于等于ht[0].used*2的2n次幂
  - 如果是收缩，ht[1]的大小为第一个大于等于ht[0].used的2n次幂

- 将保持在ht[0]上的所有键值对rehash到ht[1]上面，rehash指的是重新计算键的hash值和索引值，然后将键值对放置在ht[1]哈希表指定的位置上
- 将ht[0]上所有的键值对都迁移到ht[1]，释放ht[0]，将ht[1]设置为ht[0]，ht[1]设置成空白，为下一次做rehash做准备

###### 什么时候会rehash

1. 服务器没有在执行BGSAVE或者BgRewriteAOF命令，并且hash表的负载因子大于等于1

2. 正在进行BgSave或者BgRewriteAOF命令，并且hash表的负载因子大于等于5

   ```c
   负载因子 = 哈希表已保持节点数量 / 哈希表大小
   load_factor = ht[0].used / ht[0].size
   
   在执行BgSave或者BgRewriteAOF命令时，Redis需要创建当前服务器进程的子进程，而大部分操作系统都采用写时复制（copy-on-write）技术优化子进程的使用效率。
   负载因子在执行上面两个命令时，会自动提高hash表的负载因子，为了避免在子进程存在期间进行rehash操作，避免不必要的内存写入，最大限度的节约内存
       
   当负载因子小于0.1的时候，程序自动对hash表进行rehash。
   ```

##### **渐进式rehash**

为了解决hash表键值数量比较多的时候，不能一次性全部rehash到ht[1]，这样会导致服务器在一段时间停止服务，所以采用分多次、渐进式地将ht[0]里面的键值对慢慢地rehash到ht[1]，**分而治之**。

1. 为ht[1]分配空间
2. rehashidx rehash索引置为 0，代表rehash开始
3. 每rehash完成以后，rehashidx += 1
4. 直到rehashidx完成，rehashidx = -1

注：

​	rehash过程中，字典会同时使用ht[0]和ht[1]两个hash表，比如：查找一个键的话，先在ht[0]上找，找不到，继续到ht[1]上去找，新增的键值对一律添加到ht[1]上，这样就保证了ht[0]只减不增，随着rehash操作的执行最终变成空表。

```
字典被广泛用于实现Redis的各种功能，其中包括数据库和哈希键。
Redis中的字典使用哈希表作为底层实现，每个字典带有两个哈希表，一个平时使用，另一个仅在进行rehash时使用。
当字典被用作数据库的底层实现，或者哈希键的底层实现时，Redis使用MurmurHash2算法来计算键的哈希值。
哈希表使用链地址法来解决键冲突，被分配到同一个索引上的多个键值对会连接成一个单向链表。
在对哈希表进行扩展或者收缩操作时，程序需要将现有哈希表包含的所有键值对rehash到新哈希表里面，并且这个rehash过程并不是一次性地完成的，而是渐进式地完成的。 
```

#### 4、跳跃表

```
/* ZSETs use a specialized version of Skiplists */
/*
 * 跳跃表节点
 */
typedef struct zskiplistNode {

    // 成员对象
    robj *obj;

    // 分值
    double score;

    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 跨度
        unsigned int span;

    } level[];

} zskiplistNode;

/*
 * 跳跃表
 */
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;

} zskiplist;
```

跳跃表是有序集合的底层实现之一。
Redis的跳跃表实现由zskiplist和zskiplistNode两个结构组成，其中zskiplist用于保存跳跃表信息（比如表头节点、表尾节点、长度），而zskiplistNode则用于表示跳跃表节点。
每个跳跃表节点的层高都是1至32之间的随机数。
在同一个跳跃表中，多个节点可以包含相同的分值，但每个节点的成员对象必须是唯一的。
跳跃表中的节点按照分值大小进行排序，当分值相同时，节点按照成员对象的大小进行排序。

**跳跃表就是一个可以实现二分查找的有序链表**
