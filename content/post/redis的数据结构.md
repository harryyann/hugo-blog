---
title: "Redis数据类型、操作指令和底层实现原理"
date: 2021-09-11T00:51:55+08:00
Description: ""
Tags: [redis, 数据结构]
Categories: [缓存]
DisableComments: false
---

# 字符串

## SDS

字符串在 Redis 中的实现叫做SDS(Symple Dynamic Strings)，简单动态字符串。redis中所有场景中出现的字符串，基本都是由SDS来实现的，包括：

- 所有非数字的key。例如`set msg "hello world"` 中的"msg"这个key；
- 字符串数据类型的值。例如`set msg "hello world"`中的msg的值"hello wolrd"；
- 非字符串数据类型中的“字符串值”。例如`RPUSH fruits "apple" "banana" "cherry"`中的”apple” “banana” “cherry”。

SDS的数据结构如下（3.2版本以前）：

```c
struct sdshdr{
    int len
    int free
    char buf[]
}
```

- len 字段：代表字符串的长度
- free 字段：代表还有多少可用的空间
- buf 数组：存放字符串的一个一个字符、

可以看到，Redis 的字符串实现和 Go 的切片的实现十分相似（可以查看[Go的Slice底层实现](https://yanghairui.life/post/golang%E7%9A%84slice%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0/)这篇文章）。

使用SDS带来的好处：

1. O(1)时间复杂度获取字符串长度；

2. 不会数组越界，通过free字段可以方便的判断是否可以扩充；

3. 减少了修改字符串过程中出现的内存重分配次数，主要是通过**空间预分配**和**惰性删除**实现：

   - 空间预分配：

     每次扩容，扩容后小于1M大小的字符串，都是翻倍扩容，大于1M则是扩充1M。这样的预分配方式减少了后续再次扩容时的内存重分配。

   - 惰性删除：

     在字符串缩容后，不会立刻删除多余空间，而是将多余空间保存在 free 中，以备后续的扩容操作。

     同时 Redis 也提供了 API 可以在需要的时候真正的删除掉不用的内存空间

4. SDS是二进制安全的，存进去的是什么，取出来的就是什么，原因就是底层是用数组存储的一个一个的字符；

5. 除了是字符串的底层实现，SDS还作为 Redis 的各种缓冲区的实现

## SDS的底层编码方式

当我们使用命令 `object encoding key` 来查看对象存储的数据类型时，会发现字符串的键值对包含了三种不同类型： **int**、**enbstr**、**raw**。这三种其实是对应字符串对象的三种编码方式：

- int 类型

  存储的是数字，当设置的value是数字时就会采用这个编码方式，这样才能快速的响应 `incr` 等数字操作命令。

- embstr类型

  存储的是较短的字符串。可以直接分配一块连续的内存来存储，来提升内存释放和查找的速度。非数字的字符串在小于44个字节时，会采用这种方式存储。

- raw类型

  存放较长的字符串，会把这个字符串存放到多块内存区域中，在字符串长度大于44个字节时（不区分数字还是非数字，太大了都要拆分），就会采用这种方式存储，明显这种方式性能上不如embstr和int。

  同时要注意，如果一个数字太大了，占用字节太多导致从int转化为了raw，那么再对这个key执行`incr`时，会报错：

  ```mysql
  127.0.0.1:6379> set kye 1212121212121212121221212121212121212313413412431121
  OK
  127.0.0.1:6379> object encoding kye
  "raw"
  127.0.0.1:6379> incr kye
  (error) ERR value is not an integer or out of range
  ```

  当数字的大小超过2^32或者2^64就会转化为raw(取决于操作系统的位数)。

## 字符串的应用场景

#### Json数据缓存

各种语言的对象都可以在完成 json 的序列化后以字符串的形式存储到 Redis，然后再从 Redis 中取出再反序列化为对象。比如前后端开发中，就常常使用这种方式。

#### 数字计算与统计

由于 Redis 可以通过字符串存储整数和浮点数，并且可以通过命令直接操作数字进行累加或者累减，这样就省去了每次都要取数据、加数据、再存数据的操作，可以用一个命令完成。后续会详细介绍字符串的各种命令

#### 存储session等信息

前后端开发中可能会通过session、token等保存用户的会话状态，这种数据就十分适合存储在缓存中，如果存到数据库则性能就太低了。

## 字符串相关命令

#### 单个键值对操作

1. `set key value [EX seconds|PX milliseconds|KEEPTTL] [NX|XX]`

   添加一个键值对，可以指定过期时间(EX/PX/KEEPTTL)，和是否在键不存在的时候设置键值(NX)。

   EX是设置过期的秒数，PX则可以设置毫秒。

   KEEPTTL 参数会保留原键值对的生存时间，即如果原来已经有这个键了，则修改后的键值对会继承原来的生成时间。

2. `get key`

   获取键的值，当键不存在时，返回nil，当键存在，但是键的类型不是字符串时，返回错误。

3. `append key value`

   给键对应的值追加一个字符串，返回追加后的值的长度。

4. `strlen key`

   查询字符串的长度。

5. `getset key value`

   用于设置键对应值的新值，并且返回旧的值。

6. `setrange key offset value`

   根据偏移量截取字符串的一部分并赋新值，截取的部分大小和新值的大小一样，如果需要截取的位置已经不存在了，则会用空白自负补充。

7. `getrange key start end`

   根据指定的索引范围截取字符串。可以用负数，-1 表示最后一个字符，-2 表示倒数第二个，以此类推。

8. `setex key seconds value`

   设置键值，并指定过期时间。这个和普通的 set 命令加 ex 选项的效果是一样的，后续会被废弃，同样的 `psetex`、`setnx`也都会被废弃。

9. `getdel key`

   这个是 Redis6.2.0 新增的命令，获取一个值后将这个键删除，不存在就返回nil。

10. `getex key EX seconds|PX milliseconds|EXAT timestamp|PXAT milliseconds-timestamp|PERSIST]`

    同样也是 Redis6.2.0 新增的命令，用于获取键的值，并设置或移除键值对的过期时间。

#### 多个键值对的操作

1. `mset key value [key value ...]`

   同时设置多个键值对。**mset 是一个原子操作**，所有的key都会同时设置，不会出现某个key更新了，其他的没更新的情况。同时，如果一个操作失败了，那么其他的也都设置不了。

2. `mget key [key ...]`

   同时获取多个键。返回值的顺序是按照输入 key 的顺序返回的。

#### 数字操作

1. `incr key`

   给key的值加 1，返回加完的值。如果key不存在，则先把key的值初始化为0，再加。

2. `decr key`

   给key的值减一，返回减完的值。如果key不存在，则先把key的值初始化为0，再减。

3. `incrby key increment`

   给key加一个指定的值，同样的，如果key不存在，则先把key的值初始化为0，再加。

4. `decrby key decrement`

   给key减一个指定的值，同样的，如果key不存在，则先把key的值初始化为0，再减。

5. `incrbyfloat key increment`

   给key加一个指定的浮点数，同样的，如果key不存在，则先把key的值初始化为0，再加。

   没有decrbyfloat命令。

# 字典

Redis的字典使用哈希表作为其底层实现之一，除此之外还有很多其他地方也使用到了哈希表的数据结构，比如Redis数据库就是一个大字典，所有key的增删改查都是在操作这个大字典。

除了哈希表之外，如果一个字典中只包含很少的键值对，且每个键值对要么都是整数值，要么是很小的字符串，那么这个dict就会使用 ziplist (压缩列表)作为底层实现：

```mysql
127.0.0.1:6379> hset a a1 1 b1 1
(integer) 2
127.0.0.1:6379> object encoding a
"ziplist"
```

## dict的数据结构

字典本质是由数组和链表组成的，它的结构如下：

```c
typedef struct dict { // dict.h
    dictType *type; // 存储类型的特定函数
    void *privdata; // 私有数据
    dictht ht[2];   // 哈希表，两个数组
    in trehashidx;  // rehash的索引，没有rehash时是-1
} dict;
```

比较关键的就是 ht 属性，该属性包含了两个数组，这两个数组就是真正存储数据的哈希表，一般情况下只会使用ht[0]，ht[1]用于rehash时使用，同时 trehashidx 属性标识了当前rehash操作的进度，这个会在后续说明。

一个ht数组默认长度是4，即它可以存储4个键值对（触发扩容后才会增加），一个普通的字典大概长这样：

![](/images/redis_data/dict2.png)

## 哈希冲突

Redis字典的哈希冲突是通过链表法解决的，即一般情况下是通过数组存储数据，当出现哈希冲突时会在指定的数组位置处向后拉一个链表，大概如下：

![](/images/redis_data/dict.png)

字典存储流程是，先将键值进行哈希计算，得到一个数组的索引，然后把键值存在数组的对应位置，如果这个位置上已经有其他键值，则在这个位置向后拉一个链表存储。并且，每个新的冲突节点，都是排在链表的最前面的（为了插入的时间复杂度O(1)），如下，按照k0、k1、k2的顺序插入，k2和k1冲突时会在链表最前面：

![](/images/redis_data/dict1.png)

## 渐进式rehash

Redis 会在哈希表过满时和过空时触发哈希表的扩容和缩容，扩缩容的过程就是rehash的过程。Redis 采用渐进式的 rehash 来保证高性能运行。

#### 负载因子

通过哈希表的负载因子来评判一个哈希表是否存储了过多的数据，负载因子的计算方式是：

```c
load_factor = ht[0].used / ht[0].size
```

即ht[0]中使用的节点数，除以总共的节点数，以此来评判哈希表装载情况。Redis规定常规情况下当负载因子大于 1 时会触发rehash自动扩容，即如果ht[0]的长度是4，当存储了4个键值对时就需要扩容；当负载因子小于 0.1 时则会触发自动缩容。

如果服务器正在执行 `bgsave` 或者 `bgrewriteaof` 命令时，此时扩容的条件就变成了负载因子大于 5 (这两个命令可以参考文章[Redis的持久化](https://yanghairui.life/post/redis%E7%9A%84%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6/))。

#### 渐进式的扩缩容方式

Redis 对哈希表进行扩容的方式大致如下：

1. 观察到哈希表的负载因子和其他条件满足了扩容和缩容的需求，会触发rehash；
2. 为字典的 ht[1] 号哈希表分配空间，这个空间的大小取决于要执行的是扩容还是缩容，以及当前 ht[0] 的长度。如果是**扩容，那么就是取第一个大于等于ht[0].used这个值的某个2的n次幂再乘以2（翻倍）；如果是缩容，那么就是取第一个大于等于ht[0].used这个值的某个2^n**；
3. 然后将所有 ht[0] 上存储的键值对都rehash到 ht[1] 上；
4. 当所有键值对都迁移完成后，把 ht[1] 设为 ht[0]，把 ht[0] 释放，然后新建一个 ht[1] 为下一次rehash做准备。

如上所说，rehash过程中需要进行大量的数据搬迁操作，为了尽量减少rehash过程对服务器性能造成影响，Redis 会将这个rehash动作分多次、渐进式的完成。渐进式的rehash过程大致如下：

1. 为 ht[1] 分配空间，让这个字典同时拥有 ht[0] 和 ht[1] 两个哈希表；
2. 在字典对象中维护一个rehashidx的属性（如上所说）,这个属性在没有进行rehash时是 -1，当开始rehash时，会把它设为0，意味着将要从哈希表的 0 号位置开始搬迁；
3. 然后在rehash过程中，每次对字典的添加、删除、查找、更新等操作时，Redis 除了完成这些指令，还会顺带进行一些搬迁工作，并更新 rehashidx 属性；
4. 最后当rehash完成，rehashidx 属性会再次被赋值为 -1。

## 字典的使用场景

#### 商品购物车

购物车非常适合用字典存储，使用人员的uid作为key，然后field存商品id，value就是商品数量，金额等信息。

#### 用户的属性信息

很多场景下，需要查询用户的某些属性信息，如果用字典事先存好每个用户的属性和值，那么查询时将非常容易。

#### 页面的某些元信息等

比如文章详情页的一些诸如：文章字数、作者id、阅读时间等键值类型的信息，非常适合用dict存储。

## 字典的操作命令

1. `hset key field value [field value ...]`

   设置一个key，并且指定它的一个或多个 field:value 对。

2. ` hget key field`

   获取某个key的某个field。

3. `hkeys key`

   查询某个key的所有field。

4. `hvals key`

   查询某个key的所有value。

5. `hgetall key`

   获取某个key的所有的 field:value 对。

6. `hlen key`

   查询某个key中的 field:value 对个数。

7. `hexists key field`

   查询某个key是否含有某个field，有就返回 1，没有就返回 0 。

8. `hdel key field [field ...]`

   删除某个key的一个或多个field。

9. `hincrby key field increment`

   当某个key中的某个field的值是数字时，可以直接操作给它加一个整数。

10. `hincrbyfloat key field increment`

    当某个key中的某个field的值是数字时，可以直接操作给它加一个浮点数。

11. `hmset key field value [field value ...]`

    和 hset 一毛一样，后面会被弃用。

12. `hmget key field [field ...]`

    可以同时获取某个key的某些field的值，hget则是只能获取一个。

13. `hsetnx key field value`

    给key的某个不存在的field赋值，如果field存在，则操作不生效。如果没有这个key，则会自动创建这个key。

14. `hscan key cursor [MATCH pattern] [COUNT count]`

    用于遍历哈希表中的键值对。cursor是游标，pattern是匹配的模式，count是返回多少元素。首次迭代时cursor设为0，每次返回会返回一个游标，当返回的游标为0时，证明迭代结束了。如果不为0我们可以继续使用这个命令，使用返回的游标数值继续请求。

# 列表

Redis 的列表使用链表数据结构作为其底层实现方式。元素的插入会按照先后顺序存储到链表中，插入删除的时间复杂度是O(1)，但是查询的时间复杂度是O(n)。

在Redis 3.2以前，列表是使用的 ziplist(压缩列表) 和双向链表组成的，3.2以后改为了 quicklist (快速列表)来实现的。

## 列表的数据结构

Quicklist数据结构：

```c
typedef struct quicklist { // src/quicklist.h
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* ziplist 的个数 */
    unsigned long len;          /* quicklist 的节点数 */
    unsigned int compress : 16; /* LZF 压缩算法深度 */
    //...
} quicklist;
```

节点的数据结构：

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;           /* 对应的 ziplist */
    unsigned int sz;             /* ziplist 字节数 */
    unsigned int count : 16;     /* ziplist 个数 */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* 该节点先前是否被压缩 */
    unsigned int attempted_compress : 1; /* 节点太小无法压缩 */
    //...
} quicklistNode;
```

从以上结构中，可以看出每个 quicklist 的节点都有前后两个指针，所以 quicklist 实际上是一个双向的链表。链表中的每个节点实际上是一个ziplist。大概是如下的结构：

![](/images/redis_data/quicklist.png)

Ziplist(压缩列表)本质上是一个字节数组，是一整块固定大小的顺序型的结构，其结构如下：

![](/images/redis_data/ziplist.png)

## 列表的使用场景

#### 消息队列

列表由于其顺序存储的特性，可以提供先入先出的功能，作为一个队列，常用的异步任务框架比如 Celery 等，都支持使用 Redis 的列表作为其消息队列。

#### 存储列表型的数据

比如文章列表、商品列表等，因为列表的的有序，也可以方便的实现分页的功能。

## 列表的操作命令

1. `lpush key element [element ...]`

   将一个或多个元素推入某个key的列表左端，返回这个key的列表长度。

2. `rpush key element [element ...]`

   将一个或多个元素推入某个key的列表右端，返回这个key的列表长度。

3. `lpushx key element [element ...]`

   将一个或多个元素推入某个key的列表左端，只能在列表key已存在的时候插入，否则操作不会执行并返回 0。

4. `rpushx key element [element ...]`

   和lpushx同理。

5. `lpop key`

   弹出某个列表最左端的一个元素。如果key不存在或者列表被弹空了会返回nil，

6. `rpop key`

   和lpop同理。

7. `rpoplpush source destination`

   操作的是两个列表key，会将source列表最右端的元素推入到destination最左端。返回这个操作的元素。

8. `llen key`

   返回这个列表的元素数量。

9. `linsert key BEFORE|AFTER pivot element`

   在某个值之前/之后添加一个元素。privot是元素的值，不是下标。

10. `lset key index value`

    根据下标修改某个元素。

11. `ltrim key start stop`

    根据下标删除start下标和stop下标之间的一些元素。

12. `lrem key count value`

    从左开始删除指定个数的某个元素，返回影响的元素数目，如果元素不够删，就会全部删掉。

13. `lindex key index`

    获取某个下标上的元素，如果下标不存在，则返回nil。

14. `lrange key start stop`

    返回某个范围内的列表元素。

15. `blpop key [key ...] timeout`

    阻塞式弹出，返回的是key和弹出的元素，如果没有可弹出的就会被阻塞住，直到超时，返回nil和超时时间。

16. `brpop key [key ...] timeout`

    和 blpop 同理。

17. `brpoplpush`

    同理，没有可弹出的就会被阻塞住。

# 集合

集合是唯一无序的类型，这也是集合和列表的区别。列表可以存储一样的值，并且还是按照顺序存储的。集合有它自己的运算方式，如交集、并集、差集等。

## 实现方式

Redis的集合通过两种方式实现，intset(整数集合)和哈希表。

当集合通过哈希表实现时，哈希表的key就是元素值，value是nil。

而当集合中的数都是整数时，且元素数量不多时，Redis 就会使用 intset 结构来存储，通过命令`oobject encoding myset`可以查看：

```mysql
127.0.0.1:6379> sadd myset 1 3 5
(integer) 3
127.0.0.1:6379> object encoding myset
"intset"
127.0.0.1:6379> sadd myset a b c
(integer) 3
127.0.0.1:6379> object encoding myset
"hashtable"
```

#### Intset的数据结构

intset 的数据结构如下：

```c
typedef struct intset{
	uint32_t encoding;
	uint32_t length;
	int8_t contents[];
}
```

其中的contents数组就是整数集合的底层实现，集合中的每个元素都是contents的一个数据项，每个项在contents中是按照值的大小有序的排列。

length属性则记录了intset中包含的元素数量，即contents数组的长度。

encoding属性则是记录contents中存储的数据类型，可能是int16、int32、int64。

每当我们将一个新的元素加入集合中时，并且新元素不能用当前encoding指定的整数类型存储时，就会发生**升级**操作。升级就是根据新元素可以存储的整型来扩充contents数组，并且还要把所有集合中原有的数据都转换成这个更大的整数类型存储。这样做的主要原因，一方面是为了节约内存（尽量用合适的数据类型存储数字），另一方面是因为C语言中contents数组中不能存储不同类型的数据。

需要注意的是，intset并没有降级的操作，即一旦升级，即使那个大的元素被从集合中删除，集合也不会回到原来的长度，所有元素还是用这个更大的数据类型存储。

## 集合的使用场景

微博中关注我的人，和我关注的人就比较适合用集合存储，可以保证不重复。另外抽奖人员的id也可以用集合存储，这样来保证一个人不会重复中奖。

总之，需要保证元素唯一性的场景都比较适合集合存储。

## 集合的操作命令

1. `sadd key member [member ...]`

   给一个集合添加一些元素，如果没有这个key则会新建一个，返回的是新加入的元素个数。

2. `spop key [count]`

   随机返回并移除几个元素。

3. `srandmember key [count]`

   随机返回几个元素，和pop的区别是这个不会移除。

4. `scard key`

   获取集合中的元素个数。

5. `sismember key member`

   判断一个member是否在某个集合中，存在则返回1，不存在则返回0。

6. `smembers key`

   获取集合中的所有元素。

7. `smove source destination member`

   把一个集合中的某个member移动到另一个集合中。

8. `srem key member [member ...]`

   从集合中移除几个member，返回受影响的元素个数。

9. `sinter key [key ...]`

   获取多个集合的交集，返回这些集合中相同的元素。

10. `sunion key [key ...]`

    获取多个集合的并集。

11. `sdiff key [key ...]`

    获取多个集合的差集。

12. `sinterstore destination key [key ...]`

    获取多个集合的交集，并且把结果存到一个其他集合中。

13. `sdiffstore destination key [key ...]`

    和上一个类似，获取多个集合的差集，并且把结果存到一个其他集合中。

14. `sunionstore destination key [key ...]`

    和上两个类似，获取多个集合的并集，并且把结果存到一个其他集合中。

15. `sscan key cursor [MATCH pattern] [COUNT count]`

    遍历集合，和字典的scan一样的使用方式。

# 有序集合

## 实现方式

有序集合相比于集合多了个排序的属性，所以对于zset来说，每个元素相当于由两个值组成，一个是有序集合的元素值，一个是这个元素的排序分值。zset的存储元素值不可重复，但是排序分值是可以重复的，当分值重复时，元素按照自身的大小进行排序。

有序集合底层是通过 ziplist (压缩列表)和 skiplist (跳表)实现的。

当数据量比较小时，有序集合使用的是 ziplist 存储。需要满足以下两个条件：

- 有序集合的元素个数要小于128；
- 有序集合的所有元素的长度要先小于64个字节。

当不满足以上条件时，有序集合就会采用跳表实现。实际上可以通过参数`zset-max-ziplist-entries`（默认128）和参数`zset-max-ziplist-value`（默认64）来指定这两个临界值。

## Skiplist

跳表的实现原理如下，是一个多层的链表，每一层都是有序的，上层可以看成是下层的索引，最下面的层中存储着完整的数据：

![](/images/redis_data/skiplist.png)

假如我们要在以上跳表中查找32，过程会是这样的：

- 从最上层开始找，1比32小，则继续在同一层向后找，7也比32小，而7的后序节点是null，则只好向下一层找；
- 在下一层中继续向后找，18比32小，后一个节点77比32大，则说明一定在这两个节点之间；通过18移动到下层；
- 在第0层的18向后找，就找到了32。

由上述的例子可知，跳表本质上就是二分法的实现，支持平均O(logN)，最坏O(N)的时间复杂度，而且由于顺序存储，还可以批量处理一些顺序的节点。

Redis中每一层节点的提取，并不是均匀的，而是类似快速排序算法那样随机的选择一些pivot，这样的话其实是期望时间复杂度是O(logN)，不过这样比较容易实现。

#### Redis为什么不用红黑树

跳表的性能和红黑树基本相近，第一是因为跳表比红黑树容易实现的多。

第二个点是要考察ZSET提供的API，ZSET需要支持插入、查找、删除、按区间查找，和迭代输出几个API，其中，插入、删除、查找以及迭代输出有序序列这几个操作，红黑树也可以完成，时间复杂度跟跳表是一样的，按照区间来查找数据这个操作，红黑树的效率没有跳表高。跳表可以做到 O(logn) 的时间复杂度定位区间的起点，然后在原始链表中顺序往后遍历就可以了。这样做非常高效。

#### MySQL为什么不用跳表

主要是因为磁盘IO，MySQL需要从磁盘上加载数据，所以必须减少磁盘IO的次数，B+树的叶子节点天生和磁盘块对应，且MySQL的B+树可以将磁盘IO限制在 2~4 次，而跳表基于链表的实现则不适合用磁盘读写，同时，区间查找也会是个问题。还有一个原因是跳表的查询效率不如B+树稳定。

## 有序集合的使用场景

排行榜类的又先后顺序的场景。或者粉丝列表，需要根据关注的先后顺序排序的时候。

## 有序集合的相关命令

1. `zadd key [NX|XX] [CH] [INCR] score member [score member ...]`

   添加一个或多个带分数的成员到集合里。nx参数意味着只能添加新成员，对于已有的成员操作不会生效，xx则是会更新已经存在的成员；ch参数表示返回值是变化的成员总数，否则是添加的成员数量。

2. `zcard key`

   返回某个有序集合中所有元素的个数。

3. `zcount key min max`

   查询某个分数区间内的元素个数。

4. `zincrby key increment member`

   累加一定分数给某个元素。

5. `zrange key start stop [WITHSCORES]`

   查询某个索引范围内的数据，两端都是闭区间。withscores参数将会同时返回元素的分数。

6. `zrevrange key start stop [WITHSCORES]`

   查询某个索引范围内的数据，但是是按照分数倒序返回。

7. `zrangebyscore key min max [WITHSCORES] [LIMIT offset count]`

   按照某个范围内的分数进行排序，并返回成员。

8. `zrevrangebyscore key max min [WITHSCORES] [LIMIT offset count]`

   按照某个范围内的分数进行排序，并返回成员。但是是倒序返回。

9. `zrank key member`

   查看某个成员的排名，0就是第一，1就是第二。按照分数从小到大的顺序排。

10. `zrevrank  key member`

    查看某个成员的排名，不过是倒着的，是按照分数从大到小排。

11. `zrem key member [member ...]`

    删除一个或多个成员。

12. ``

13. `zremrangebyrank key start stop`

    按照排名删除成员，start和stop都是按分数从小到大排的索引。

14. `zremrangebyscore key min max`

    按照分数排名删除成员，min和max是分数。

15. `zremrangebylex key min max`

    当集合中所有成员分数都一样时，会按照字典顺序排序，这里就是根据字典顺序删除，min和max也是这种排序后的索引。

16. `zscore key member`

    获取指定成员的分数

17. `zinterstore destination numkeys key [key ...] [WEIGHTS weight] [AGGREGATE SUM|MIN|MAX]`

    和set的命令类似，同样是两个有序集合求交集，并把结果保存到一个新有序集合中。weights参数可以指定两个集合的权重，aggragate则表示交集的聚合方式，默认是sum，就是交集中元素的score数相加。

18. `zunionstore destination numkeys key [key ...] [WEIGHTS weight] [AGGREGATE SUM|MIN|MAX]`

    和上述类似的，求并集。

19. `zscan key cursor [MATCH pattern] [COUNT count]`

    同样的，和scan命令、hscan命令、sscan命令类似。

# Bitmap

Bitmap就是位图，用一段连续的二进制数组（0和1），可以通过偏移量offset来定位元素，bitmap通过最小的单位bit表示0和1，时间复杂度O(1)，特别适合一些大数据量下的，**使用二值统计的场景**。

Redis的bitmap是基于String类型的位操作，而redis字符串的最大长度是512M，所以bitmap的offset值也是有上限的，即`8 * 1024 * 1024 * 512`个bit，即`2^32`次方个比特位。

## 常用命令

- `setbit key offset value`：

  给某个key的某个offset处置为value，value只能是0或者1。注意的是，如果第一次创建，offset特别大，就要创建一个特别大的字节数组，这么大块内存分配可能会导致Redis阻塞。创建完成后，如果不扩容，那么所有的操作都是O(1)的时间复杂度。

- `getbit key offset`

  获取指定key指定offset上的位的值。

  一个bitmap被初始化时，除了被赋了1的位，其他位上都默认是0。由于Bitmap底层是String，所有还可以直接设置一个String，然后这个String依然可以使用bitmap读，这适合需要将很多位初始化为1的情况。比如这样：

  ```mysql
  127.0.0.1:6379> set bitkey 42
  OK
  127.0.0.1:6379> getbit bitkey 2
  (integer) 1
  127.0.0.1:6379> getbit bitkey 5
  (integer) 1
  ```

- `bitcount key [start end]`

  统计给定字符串中，比特位为1的数量，默认统计整个字符串，也可以指定start和end，satrt和end可以是负数，-1就是最后一个字节，-2就是倒数第二个字节。

  注意start和end都是字节，不是位。

- `bitpos key bit [start [end]]`

  返回字符串中，从左到右，第一个比特值为指定的bit的offset值。

  不指定start和end就是整个字符串从0位开始，同样，这里的start和end都是字节，不是位。

- `bitop operation destkey key [key ...]`

  对多个字符串进行位操作，并将结果保存到destkey中。operation可以是AND、OR、XOR、NOT。

## 使用场景

#### 快速判断一个大数据集中，某个元素是否存在

可以开一个bitmap，先遍历这个大数据集，按照数据把对应的位置为1，然后就可以O(1)的时间复杂度获取某个元素是否存在了。你需要事先直到数据集中的元素的范围，才能创建一个合适的bitmap。

#### 统计用户登录情况

可以每个用户id为key，设置一个bitmap，如果要记录一年，那就是365个bit的bitmap，每一天登录了，就把对应的位置为1即可。可以用bitcount命令获取这个用户一年中登录的天数。

#### 群里已读未读消息的判断

可以用消息id作为bitmap的key，然后一个用户读了，就把这个用户id对应的比特位置为1，未读就是0。依然用bitcount命令可以很方便的统计出已读和未读的成员数量，同时也可以知道具体是哪个用户读了，哪个用户未读。



# 其他数据类型

目前位置，Redis共有9种数据类型，除了以上6种以外，还有下列3种：

- geohash：地理位置坐标
- hyperloglog：基数统计类型
- BloomFilter：布隆过滤器

剩下的后续有机会再进行研究。

# 关于Key的一些常用指令

1. `keys pattern`

   根据pattern的正则表达式查询一些key。

2. `scan cursor [MATCH pattern] [COUNT count] [TYPE type]`

   渐进式遍历，可以有效解决keys命令造成的阻塞问题，scan每次的复杂度都是O(1)。用法和之前的hscan、sscan、zscan类似，多了一个type参数，相当于这三个命令的集合。

3. `del key [key ...]`

   删除多个key，不存在就忽略。

4. `unlink key [key ...]`

   同样删除多个key，不存在就忽略，但是和del不同，它是非阻塞的，它会在其他线程中回收内存，顾名思义，它是断开key和value的连接，真正的删除操作会在后续异步完成。

5. ` expire key seconds`

   给key重新设置一个过期时间。

6. `exists key [key ...]`

   检测一些key是否存在，返回的是输入的key中存在的个数。

7. `type key`

   判断key的数据类型。

8. `ttl key`

   查看key的过期时间，-1就是没有过期时间。

9. `pttl key`

   一样，就是返回的是毫秒。

10. `rename key newkey`

   给某个key设置一个新key名。

11. `object subcommand [arguments [arguments ...]]`

    object允许我们从内部查看一个key的底层对象。比如我们之前一直使用的`object encoding key`来查看一个key的底层编码格式。

12. `randomkey`

    随机返回一个key。

13. `dump`和`restore`

    用于rdb序列化，和数据恢复。

14. `migrate`

    dump、restore、del三个命令的集合。

15. `select index`

    选择数据库。

16. `flushdb` 或 `flushall`

    清空数据库，flushdb只清空当前数据库，flushall清除所有数据库的数据。