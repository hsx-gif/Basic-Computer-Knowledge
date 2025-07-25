# 概述

Redis是速度非常快的NoSQL键值对数据库，Redis数据库是存放在内存中的。

键的类型只能是字符串，值的类型有：字符串、列表、集合、散列表、有序集合

Redis支持很多特性，将内存的数据持久化到硬盘中，使用复制来扩展读性能，使用分片来扩展写性能

Redis是单线程的模型，一个指令在执行期间，其他指令是不能执行的

Redis只是缓存，而不要将Redis看成是数据库

# 数据类型

| 数据类型 |      可以存储的值      |                             操作                             |
| :------: | :--------------------: | :----------------------------------------------------------: |
|  STRING  | 字符串、整数或者浮点数 | 对整个字符串或者字符串的其中一部分执行操作、 对整数和浮点数执行自增或者自减操作 |
|   LIST   |          列表          | 从两端压入或者弹出元素 、 对单个或者多个元素进行修剪、只保留一个范围内的元素 |
|   SET    |        无序集合        | 添加、获取、移除单个元素、检查一个元素是否存在于集合中、计算交集、并集、差集、 从集合里面随机获取元素 |
|   HASH   | 包含键值对的无序散列表 | 添加、获取、移除单个键值对、 获取所有键值对、检查某个键是否存在 |
|   ZSET   |        有序集合        | 添加、获取、删除元素， 根据分值范围或者成员来获取元素，计算一个键的排名 |



### String

对于过长的字符串建议先序列化或者是压缩后再存储

```html
> set hello world
OK
> get hello
"world"
> del hello
(integer) 1
> get hello
(nil)
```

### List

```html
# 在列表的左边插入abc
LPUSH mylist a b c

# 在列表的右边插入xyz
RPUSH mylist x y z

# 弹出列表头元素
LPOP mylist

# 弹出列表尾元素 
RPOP mylist

# 返回整个列表
LRANGE mylist 0 -1 
LRANGE mylist 1 3    # 返回索引 1 到 3 的元素

# 检查列表的长度
LLEN mylist     


# 设置指定下标的元素新值
LSET mylist 2 foo    # 将索引 2 的元素改为 "foo"
```

### SET

set里面只能是唯一的元素，如果里面已经存在的键，就不能再添加相同名字的键了

```html
sadd 添加key

> sadd set-key item
(integer) 1
> sadd set-key item2
(integer) 1
> sadd set-key item3
(integer) 1
> sadd set-key item
(integer) 0

smember 找出所有的key

> smembers set-key
1) "item"
2) "item2"
3) "item3"

sismember 判断键在不在集合里面

> sismember set-key item4
(integer) 0
> sismember set-key item
(integer) 1

srem 删除键

> srem set-key item2
(integer) 1
> srem set-key item2
(integer) 0
```

### HASH

```bash
HSET：向哈希中添加或更新字段，哈希不止有key，还有field，还有value。只有HASH才有field字段
field使用场景是：可以向一个user用户添加多个属性
例如：
fields := map[string]interface{}{
    "name":    "Alice",
    "age":     30,
    "country": "China",
}
added, err = rdb.HSet(ctx, "user:1000", fields).Result()
-----------
HSET key field value

> hset hash-key sub-key1 value1 
(integer) 1
> hset hash-key sub-key2 value2
(integer) 1
> hset hash-key sub-key1 value1
(integer) 0

HGETALL：获取哈希里所有字段及对应值

> hgetall hash-key
1) "sub-key1"
2) "value1"
3) "sub-key2"
4) "value2"

HDEL：删除一个或多个字段

> hdel hash-key sub-key2
(integer) 1
> hdel hash-key sub-key2
(integer) 0

HGET：获取指定字段的值

> hget hash-key sub-key1
"value1"
```

### ZSET

ZSET中的元素会有对应的分数

**ZADD**：向有序集合中添加（或更新）成员及其分数（score）

```html
ZADD zset-key 728 member1
```

**ZRANGE … WITHSCORES**：按分数从低到高返回指定区间内的成员及其分数

```html
ZRANGE zset-key 0 -1 WITHSCORES  
```

**ZRANGEBYSCORE**：按分数范围返回成员及其分数

```html
ZRANGEBYSCORE zset-key 0 800 WITHSCORES  
```

**ZREM**：从有序集合中删除指定成员

```html
ZREM zset-key member1
```

# 数据结构

### 字典

<mark>在redis里面，如果发生哈希冲突的话，那么就会去访问桶下面所有的链表的元素，直到找到对应的元素</mark>

字典的好处：

> 1. 对于任意的key，可以在常数时间内完成查找／插入／删除
> 2. 可以快速判断元素在不在，例如插入一个元素的话，可以判断是新增还是更新当前元素

下面的代码是Redis对字典实现的源码。

```c
typedef struct dictht {
    dictEntry **table;      # 是一块连续的内存区域，里面存放的是许多桶中每个桶对应的头指针
    unsigned long size;     # 是当前桶的总数，一般来说是2的幂次方
    unsigned long sizemask; # 用来通过按位与快速计算桶索引（不是通过取余的方式计算得到索引的，按位与比较快）sizemask = size-1
    unsigned long used;     # 已插入的键值对总数
} dictht;
```

> 当桶的个数是2的幂次方的时候，可以用与的操作来代替求余的操作
>
> 就是size和sizemask求与之后会得到对应的桶索引

**字典项**

表示表中的一个实际元素（一个键值对）。每次往哈希表插入一个键值对，就会 new 一个 `dictEntry` 节点。

```c
typedef struct dictEntry {
    void *key;                // 指向“键”的指针
    union {                   // “值”可以有多种类型，你根据需要选一个来用
        void   *val; 		 // 通用指针
        uint64_t u64;   	 // 无符号 64 位整数
        int64_t  s64;   	 // 有符号 64 位整数
        double   d;     	 // 浮点数
    } v;
    struct dictEntry *next;  // 同一桶里下一个 entry（拉链法）
} dictEntry;
```

**rehash**

用来管理整个哈希表及其增量重哈希过程，就是对旧的哈希表进行扩容/缩容

```c
typedef struct dict {
    dictType *type;          // 指向一组函数指针，定义了如何对 key、value 做哈希、比较、释放等操作
    void *privdata;          // 私有数据指针，会传给 type 里的函数做上下文
    dictht ht[2];            // 两张哈希表：ht[0] 是当前表，ht[1] 用于增量扩容/缩容时的目标表
    long rehashidx;          // 如果正在重哈希，则表示下一个要迁移的桶索引；等于 -1 表示未在重哈希
    unsigned long iterators; // 当前正在运行的迭代器数量，用于 safe iterator 期间避免重哈希
} dict;
```

> 增量的过程：不是特意去将旧的数据移动到新的哈希表中的，是在每一次用户的查询过程中将数据移动到新的哈希表中
>
> - 那么如何确定是去新的哈希表还是旧的哈希表中进行查找的呢？（会根据rehashidx进行判断）
>
> 1. 如果数据所在的桶小于rehashidx，那么就会去新的表里进行查找
>
> 2. 如果数据所在的桶大于rehashidx，那么就会去旧的桶里查找，因为这个表示数据还没移动到新的哈希表中
>
> - 如果你今天查找的是8号桶的数据，那么是先移动0号桶的数据，还是移动8号桶的数据呢？（先移动再查找）
>
> 1. 先将0号桶的数据移动到新的哈希表中（不会移动8号桶，会先移动0号桶）
> 2. 然后再去执行查找8号桶的操作
>
> - 数据移动到新的表之后，那么旧的表会进行什么操作呢？（旧的表会将移动过的数据进行删除）
>
> 1. 新旧表都会存储一部分的数据。数据移动到新的表格之后，旧表格就会删除。
> 2. 如果有新的数据到来的话，那么会根据rehashidx判断要插入新表还是旧的表格。假设现在新表存储的是0-20，旧的是21-40的话。那么当有数据要插入0号桶的话，就会查看0号桶是不是移动到新表，是的话就直接插入新表，不是的话就将数据插入到旧表。

采用渐进式 rehash 会导致字典中的数据分散在两个 dictht 上，因此对字典的查找操作也需要到对应的 dictht 去执行。

### 跳跃链表

跳跃表是有序集合的实现之一，跳跃表是基于多指针有序链表实现的，可以看成多个有序链表。

下图所示就是一个跳跃链表，其中最底层BL里面存储的是所有的链表数据，上面的层只会存储部分的数据。

下图是一个查找的过程：

> 1. 先在L3查找，没有的话就到L2查找，发现25>22就继续到下一层
> 2. 到达L1之后，就会接着查找，找到15则继续，发现25>22，就继续到下一层BL查找
> 3. 最终在BL层找到节点22

![img](../images/0ea37ee2-c224-4c79-b895-e131c6805c40.png)

跳跃链表的优点：

> 1. 插入方便，红黑树插入的时候还需要翻转才可以实现排序
> 2. 自平衡树（红黑树）的实现逻辑复杂，代码量大，跳跃链表就比较容易实现，只需要维护每一层节点的next指针即可
> 3. 在多线程并发场景里，对红黑树做插入／删除通常需要对整个树加写锁。但是跳跃链表可以不用。因为跳跃链表采用的是硬件原子 CAS（CAS是只有当内存中的值等于预期值时，才把它修改为新值，否则不做任何改变，并告诉调用者操作是否成功），所以整个操作是不可分的，只有全部做完或者全部不做。

# Redis使用场景

## 计数器

可以对 String 进行自增自减运算，从而实现计数器功能。Redis是内存型数据库，适合存储频繁读写的计数量。

## 缓存

将热点数据放在内存中，设置内存最大使用量以及淘汰策略来保持缓存的命中率。缓存的内容可能会失效

## 查找表

像DNS的记录就可以用Redis来存储，查找表的内容不能失效，但是缓存的内容会失效

## 消息队列

List的数据类型其实是一个双向链表，可以通过 lpush 和 rpop 写入和读取消息。最好还是使用 Kafka、RabbitMQ 等消息中间件。

## 会话缓存

服务器不会存储某个对话对应的cookie了而是将所有的cookie存储到Redis数据库当中。因为如果存储的话，那么该请求将会永远到达这个服务器了（粘性）。浏览器会保存Cookie，浏览器发送用户的请求之后就会带上这个Cookie，然后中间件会将这个请求发送到后端的服务器而不是固定的服务器。然后后端的服务器再去Redis数据库中找到用户所对应的Cookie，并恢复出用户上下文。

## 分布式锁实现

是一种跨多台机器、在分布式环境中保证互斥访问共享资源的协调机制

实现分布式锁的方法：

### SETNX

SETNX (SET if Not eXists)

SETNX是如果不存在就写入并返回成功

在分布式环境中用 `SETNX`（或 `SET … NX PX ttl`）来做锁，其核心就在于那条命令的**原子性**和**排它性**：

```latex
这里的A和B是客户端唯一标识，表示当前是哪个客户端在访问共享资源


# 客户端 A
if SET mylock "A" NX PX 5000 then # NX 表示的是只有当mylock不存在时才设置成功——保证互斥，PX 给锁设置超时时间，防止持锁者崩溃后锁永不释放。
  └── 成功，A 获得锁，开始访问共享资源
else
  └── 失败，A 等待后重试或直接返回获取失败

...

# 判断当前的锁是不是A，是的话就删掉A锁，因为A执行完了
if GET mylock == "A" then
  DEL mylock
end

# 客户端 B 在此期间
if SET mylock "B" NX PX 5000 then
  └── 只有等到 A 释放（或锁过期）后，B 才能成功拿到锁
```

优缺点如下：

- 优点：

> 1. 原子性强：在键不存在的时候才能写入成功（因为客户端A执行完之后会删除mylock键，如果没有这个键说明当前没有客户端在执行），保证了并发时只有一个客户端可以拿到锁
>
> 2. 实现简单：只需要一条命令即可
> 3. 性能高：Redis 本身是内存数据库，单命令执行开销极低，加锁／解锁都能在微秒级完成。
> 4. 集成方便：任何支持 Redis 的语言或框架几乎都内建了对 `SETNX` 的调用方式

- 缺点

> 1. 单点故障风险：所有的操作都依赖于一个Redis实例，一旦这个出问题就会导致功能不可用
> 2. 死锁风险：如果忘记给锁设过期（TTL），持锁者崩溃后锁永不释放。或者是锁设置时间太短，会导致其他人误拿锁。
> 3. 手动释放锁：解除锁需要先检查自己是否持锁，需要调用Lua脚本
> 4. 无阻塞延迟：SETNX 失败后客户端只能轮询（自旋）或定时重试
> 5. 功能单一：只是一把非可重入的互斥锁，不支持读写锁、可重入锁、锁续期、排队公平性等高级特性，难以应对复杂分布式场景。

### RedLock

在 N 个相互独立的 Redis 实例（建议 N≥5）上并行尝试获取同一把锁，并以 多数派（>N/2）获得锁作为加锁成功的判据。

<mark>下面是个例子：</mark>

**部署**：至少 5 个相互独立的 Redis 实例。

**加锁**：客户端生成唯一 ID，依次向每个实例执行

```
SET resource_name unique_id NX PX ttl  
```

统计返回 OK 的实例数，若大于半数且耗时 < ttl，则视为加锁成功；否则在已成功的实例上撤销并返回失败。

**解锁**：在所有实例上运行原子 Lua 脚本

```lua
if redis.call("GET",KEYS[1]) == ARGV[1] then 
  return redis.call("DEL",KEYS[1]) 
else 
  return 0 
end
```

确保只有持锁者能释放。

**特点**：多数派可用保证高容错，TTL 防死锁，无需客户端时钟同步。

## 其它

Set 可以实现交集、并集等操作，从而实现共同好友等功能。

ZSet 可以实现有序性操作，从而实现排行榜等功能。

## 限流



## 朋友圈点赞

## 抽奖







# Redis和Memcached

Redis和Memcached都是非关系型数据库

> 关系型数据库：关系数据库是基于关系模型创建的数据库，数据以表的形式存放，在使用前需要定义表结构，非关系型得到数据库都可以通过 SQL 语句来查询，通过SQL和事务保证数据一致性和完整性（支持ACID事务）。适合于复杂的关系强、事务多、报表／分析复杂的传统业务系统
>
>
> 非关系型数据库：非关系型数据库可以不定义结构，每种数据库的API/查询语言都不一样。适用于实时写入、缓存、日志、社交、推荐等场景

### 数据类型不同

Memcached只支持字符串，但是Redis支持五种不同的数据类型

### 数据持久化

数据持久化的意思是：将内存中的数据定期或实时地保存到磁盘上，以便在Redis重启或者是崩溃后能够恢复原有地数据状态

Redis有两种持久化地策略：RDB快照以及AOF日志，但是Memcached不支持数据持久化

### 分布式

**Memcached的分布式是采用一致性哈希（Consistent Hashing）的方式实现的，一致性哈希是一种哈希算法**

> 一致性哈希
>
> 1. 将所有的Memcached节点放在一个哈希环上做虚拟节点映射
> 2. 如果服务器有新的节点加入，只需要在客户端添加（可以采用手动添加，或者是动态配置等）即可，不需要服务器之间的相互通信。
> 3. 当有节点发生故障的时候，只需要顺着哈希环将故障节点的数据移动到新的节点上即可，只会影响部分范围的数据。如果是直接哈希或是除法哈希的话，当系统中服务器的数量增加或减少（例如添加或移除服务器）时，使用传统哈希算法会导致大多数键需要重新映射到新的服务器上，这会引起数据的大规模迁移，效率很低。
> 4. 每次get/set的时候，客户端会先对key做哈希，找到环上顺时针的第一个对应的机器。
> 5. 向那台机器发送请求

一致性哈希的例子

- 假设有 3 台服务器 A、B、C，它们的哈希值分别位于虚拟哈希环上的不同位置。比如，A 的哈希值是 3，B 的哈希值是 7，C 的哈希值是 11（这里简化了哈希值的范围，实际应用中哈希值范围会更大）。
- 当要存储一个对象 X，计算对象 X 的哈希值为 5。在哈希环上，从对象 X 的哈希值 5 位置开始顺时针查找，找到的第一个服务器节点是 B（7），所以对象 X 就会被存储到服务器 B 上。
- 如果此时添加一个新的服务器 D，其哈希值为 9。那么对于对象 X 来说，它的存储位置仍然是服务器 B，因为从 X 的哈希值 5 顺时针查找，最近的服务器仍然是 B。但是，对于哈希值在 7 - 9 之间的对象，它们的存储位置就会从服务器 B 变为服务器 D，这样就只会影响一部分对象的映射关系。

<mark>Memcached的优点：</mark>

实现简单，不需要进行额外的集群管理

<mark>Memcached的缺点：</mark>

1. 客户端需要自己持有和更新服务器节点列表以及一致性哈希算法
2. 不支持跨节点复制、故障转移：这是因为Memcached每个节点存储的数据都是不一样的，不会将当前节点存储的数据复制备份到另一个节点，所以节点宕机的话就会导致这个节点的数据不能使用。所以Memcached适用用于高速缓存，因为缓存的数据是允许丢失的，如果节点发生故障，那么可以用一个新的节点从数据库或者源头加载数据即可。

**Redis Cluster的分布式**

Redis Cluster的分布式是在服务器内实现的，在服务器内部建立分片和路由，流程如下：

1. 将整个Key空间划分为16384的槽（Slot），每个节点负责一部分的Key（槽）
2. 节点之间通过gossip协议互相发现、选主、故障转移
3. 客户端如果发送一个Key到不是负责该Key的节点，节点就会返回MOVED或ASK，客户端或者驱动就会自动重试到正确的节点
4. 某个主节点挂掉后，集群可以自动将从节点升级为主节点，并更新所有节点 / 客户端的拓扑信息。

### 内存管理机制

Redis和Memcached的数据会一直存放在内存中

Memcached将内存分割为特定长度的块来存储数据，但是这样会导致内存的利用率不高

Redis按照不同的对象动态分配内存，利用 jemalloc 的 bin、arena 机制减少系统调用和碎片，同时通过 SDS、ziplist/quicklist 等策略兼顾性能和空间利用。

> bin：jemclloc是Redis默认的内存分配器，jemalloc会把不同大小的内存划分到不同的桶里，每次申请的时候就会直接从对应桶中拿出预先分配好的空闲块，这样就不需要每次都向操作系统申请了
>
>
> arena：jemalloc会为每个线程维护自己的内存池，这样线程就不会频繁竞争同一个内存池，减少锁冲突
>
> 
> ziplist：一种紧凑的、连续内存的压缩列表结构，适合存放少量或短字符串的列表／哈希／有序集合，能显著减少指针开销。
>
> quicklist：Redis 4.0 引入的列表底层实现，将多个 ziplist 串成双向链表，每个节点是一个 ziplist。

# 键的过期时间

Redis可以为每一个键设置一个过期时间，当时间到的时候Redis会自动地删除这个键

```bash
EXPIRE key seconds         # 以秒为单位设置过期时间
PEXPIRE key milliseconds   # 以毫秒为单位设置过期时间

# 也可以使用下面这种方式
SET key value [EX seconds] [PX milliseconds] [NX|XX]  # NX表示当键不存在的话才执行这次set，XX是键存在才执行set
例如： SET foo "bar" EX 30 NX
如果foo不存在的时候，就将它的值设置成为bar，设置30秒的过期时间，但是如果foo存在，则返回nil
例如： SET foo "bar" EX 30 XX
如果foo存在的话，将其值设置成为bar，设置30s的过期时间，但是如果foo不存在，则返回nil
```

# 使用Go操作Redis

对于过期的键，Redis的处理方式是这样的

> 1. 惰性删除： 当对某个Key执行读/写或者是其他操作的时候，Redis会去检查TTL，如果已经过期的话，那么Redis会将其删除，然后返回nil来处理命令
> 2. 定期删除： Redis会定期抽样检查部分带有过期时间的键（有过期的时间的键Redis会将其存入到字典当中，并定时在字典里面随机抽样进行检查，如果发现过期则会将其删除），然后将这些键全部删除。
>
> 所以只有在访问键的时候，那些过期的键才会被马上删除
>
> 并且在go-redis v8中，所有命令都返回一个 `*redis.XxxCmd` 对象，你需要调用它的 `.Result()`（或者只关心错误的 `.Err()`）来拿到最终结果和错误。

```go
package main

import (
	"context"
	"fmt"
	"github.com/go-redis/redis/v8"
	"log"
	"time"
)

func main() {
	// 1. 创建一个 context（go-redis v8+ 都需要显式传 context）
	ctx := context.Background()
	myKey := "myKey"

	// 2. 初始化客户端
	rdb := redis.NewClient(&redis.Options{
		Addr:     "120.92.20.105:26356",            // Redis 服务地址
		Password: "C2025xTkSbRi%xQtFkEX98BQR3yQ==", // Redis 密码（没有可留空）
		DB:       0,                                // 选择 DB 0
		// （可选）连接池参数：
		// PoolSize:     10,                        // 同一个客户端最大可以同时打开的TCP连接数
		// MinIdleConns: 3,							// 池里最少要保留多少空闲连接。
		// DialTimeout:  5 * time.Second,			// 建立新 TCP 连接时的超时时间。
	})

	// 3. 测试连接：发送 PING
	pong, err := rdb.Ping(ctx).Result()
	if err != nil {
		log.Fatalf("连接 Redis 失败: %v", err)
	}
	fmt.Println("Redis PING:", pong) // 应该打印 "PONG"

	err = rdb.Set(ctx, "mykey", "hust", 10*time.Second).Err()
	if err != nil {
		log.Fatal(err)
	}
	val, err := rdb.Get(ctx, "mykey").Result()
	if err != nil {
		log.Fatal(err)
	} else {
		fmt.Printf("mykey=%s\n", val)
	}
	time.Sleep(10 * time.Second)
	fmt.Println("Waiting for 15 seconds...")
	val, err = rdb.Get(ctx, "mykey").Result()
	if err != nil {
		fmt.Println(err)
	} else {
		fmt.Printf("%s=%s\n", myKey, val)
	}
	err = rdb.HSet(ctx, "user001", "name", "hsx").Err()
	if err != nil {
		fmt.Println(err)
	} else {
		fmt.Printf("Set success!\n")
	}
	err = rdb.HSet(ctx, "user001", "add", "east 10").Err()
	if err != nil {
		fmt.Println(err)
	} else {
		fmt.Printf("Set success!\n")
	}
	val, err = rdb.HGet(ctx, "user001", "name").Result()
	if err != nil {
		fmt.Println(err)
	} else {
		fmt.Printf("%s\n", val)
	}
	val, err = rdb.HGet(ctx, "user001", "add").Result()
	if err != nil {
		fmt.Println(err)
	} else {
		fmt.Printf("%s\n", val)
	}
	// 删除多个字段
	removed, err := rdb.HDel(ctx, "user001", "name", "add").Result()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("删除了 %d 个字段（field1, field3）\n", removed)

}
```

# Redia和Lua脚本结合

可以用EVAL执行Lua脚本

可以用SCRIPT LOAD将脚本存在Redis里面，并返回其SHA1校验值，后面就可以用这个校验值来代表这个脚本，运行这个脚本。

```bash
1. 把脚本加载进服务器，拿到 sha1
127.0.0.1:6379> SCRIPT LOAD "return redis.call('INCR', KEYS[1])"
"f3c3e2f6d324c3ae5f2c1e6e1234567890abcdef"

2. 用 EVALSHA 执行脚本
127.0.0.1:6379> EVALSHA f3c3e2f6d324c3ae5f2c1e6e1234567890abcdef 1 mycounter
(integer) 1
127.0.0.1:6379> EVALSHA f3c3e2f6d324c3ae5f2c1e6e1234567890abcdef 1 mycounter
(integer) 2
```

# 缓存穿透

## 基本概念

缓存穿透指的是缓存没发挥作用，数据不在缓存里面也不在数据库里面。（因为是恶意的攻击，所以反复的去访问已知不存在的Key）

## 解决办法

1. 最好：防止非法请求打到缓存
2. 退而求此次：请求打到缓存就好，缓存扛住，保护后端数据库
3. 做好入参校验：限制非法参数，比如手机号，身份证等等
4. 布隆过滤（真实比较少，因为布隆过滤真实占用的存储空间比较大）

# 缓存击穿

## 基本概念

缓存击穿指的是数据不在缓存里面，但是<mark>数据是在数据库里面的</mark>。

原因是：热点Key过期或者是还没有加载到缓存中，大量并发请求同时访问热点Key

## 解决办法

1. 热点数据永久缓存，不过期（要根据实际情况来判断）
2. 互斥锁：同一个时刻只有一个线程查询数据库（对缓存更新操作进行加锁，保证只有一个线程可以进行缓存更新）
3. 缓存预热：提前加载缓存（什么时候预加载：）
4. 定时异步刷新

# 缓存雪崩

## 基本概念

缓存中的数据在相近时间内大批量过期（即缓存失效）或缓存系统本身故障，而导致数据库响应时间慢或数据库宕机，最终造成整个系统崩溃

## 解决办法

1. 改善集中到期的时间
2. 限流熔断，降级
3. 定时异步刷新



