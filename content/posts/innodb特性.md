---
title: "Innodb存储引擎的特性"
date: 2021-05-07T14:59:14+08:00
Description: ""
Tags: [MySQL]
Categories: [数据库]
DisableComments: false
draft: false
---

## InnoDB存储引擎的体系架构

InnoDB存储引擎的体系架构大致如下图所示，可以看到主要包含了一系列后台线程，和一个大的缓冲池。最底层则是文件系统。

![](/images/innodb/structure.png)

## InnoDB的线程模型



InnoDB的线程主要分为Master线程、IO线程、Purge线程和Page Cleaner线程。

#### Master 线程 

是核心的后台线程，主要负责调度其他线程，以及脏页的刷新，undo页的回收，redo log的刷新，合并缓冲区等。（版本不同做的工作不一样）

#### IO线程

innodb中大量使用了AIO（异步IO）来处理写请求，以提高性能，IO线程用于处理这些IO操作的回调。

通过show engine innodb status命令可以查看到当前各个IO线程的状态，以下可以看到各个线程的功能和当前状态：

```mysql
mysql> show engine innodb status\G
......
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
1433 OS file reads, 15194 OS file writes, 1197 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
-------------------------------------
......
```

可以看到，IO线程一共有四种：

- write线程：4个，负责写操作，把脏页刷到磁盘
- read线程：4个，负责读操作，把数据从磁盘加载到buffer pool
- insert buffer线程：insert buffer专用的线程，负责把insert buffer刷到磁盘，只有一个
- log线程：负责将redo log、binlog、undo log等日志从缓冲区刷到磁盘，只有一个

#### Purge线程

事务提交后，这个事务的undo log可能需要进行回收（能不能回收要看还有没有事务需要用到相关undo log的版本链，实际上只有update undo log需要被回收，insert undo log在事务结束后就可以删除了）。

在InnoDB1.1版本可以通过在配置文件上加上这行开启，并且值只能设为1：

```mysql
[mysqld]
innodb_purge_threads=1
```

1.2版本以后可以支持多个purge thread，加快undo log的回收：

```mysql
mysql> show variables like 'innodb_purge_threads';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| innodb_purge_threads | 4     |
+----------------------+-------+
1 row in set (0.001 sec)
```

#### Page Cleaner线程

在innodb1.2版本引入，用于将脏页刷新的操作都放到这个线程完成，调用IO线程的write线程完成刷盘处理，减轻master线程对于用户查询的阻塞。

## InnoDB的缓冲池

由于内存和磁盘读写的访问速度鸿沟，Innodb使用缓冲池buffer pool来进行数据存取的优化。需要和MySQL的查询缓存区分开，那个是Server层的，这个是InnoDB的，关于MySQL的架构和查询缓存的内容可以查看这篇文章：[MySQL的架构](https://yanghairui.life/posts/mysql%E7%9A%84%E6%9E%B6%E6%9E%84%E5%92%8C%E8%AF%AD%E5%8F%A5%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B/)。

buffer pool的主要作用，就是作为磁盘的缓冲。当读数据时，如果数据所在的页在缓冲池，则直接从缓冲池读取并返回；当写入数据时，如果该数据所在的页在缓冲池，则先修改缓冲池中的数据页并返回，后续在一定时机统一将缓冲池中的数据页统一刷到磁盘。

通过参数`innodb_buffer_pool_size`可以设置buffer pool的大小：

```mysql
Mysql> show variables like "innodb_buffer_pool_size";
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| innodb_buffer_pool_size | 134217728 |
+-------------------------+-----------+
1 row in set (0.001 sec)
```

从Innodb1.0开始，可以设置多个缓冲池，然后各个数据页可以均匀分配到多个缓冲池中，提高并发效率。通过参数`Innodb_buffer_pool_instances`可以设置

```mysql
Mysql> show variables like "innodb_buffer_pool_instances";
+------------------------------+-------+
| Variable_name                | Value |
+------------------------------+-------+
| innodb_buffer_pool_instances | 1     |
+------------------------------+-------+
1 row in set (0.001 sec)
```



**buffer pool的组成**：

![](/images/innodb/buffer_pool.png)

如上图，buffer pool主要包含以下几块：

- 数据页(即B+树的数据页)
- 索引页(即B+树的索引页)
- insert buffer(插入缓冲)
- 自适应哈希索引
- redo log buffer(重做日志缓冲)
- 额外内存池
- 锁信息
- 数据字典信息

接下来我们将通过buffer pool的管理方式了解这些块的作用和运作方式。

### 数据页和索引页的管理：LRU算法

这部分存储的就是磁盘中的实际表数据，InnoDB的表是通过聚簇索引以B+树形式存储的（索引的相关内容可以查看[MySQL的索引](https://yanghairui.life/posts/mysql%E7%9A%84%E7%B4%A2%E5%BC%95%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)这篇文章）。所以buffer pool中存的数据页和索引页也是一个个的B+树，InnoDB的一个页的大小默认是16kb，数据页和索引页在buffer pool中通过**LRU**算法（最近最少访问）进行管理。即通过一个链表把数据页和索引页串起来，最近使用的页就插入到LRU链表的前端，当数据页和索引页占用的空间大小超过了缓冲池大小时，LRU链表的尾部的数据页会被淘汰掉。

InnoDB对传统的LRU算法进行了优化，新页会被查到LRU链表的中间的midpoint位置（5/8处），而不是最前端。midpoint的位置可以通过参数innodb_old_blocks_pct设置，默认是37，即后面37%的位置。如果预估热点数据可能很多，也可以尽量调小这个值。

```mysql
Mysql> show variables like "innodb_old_blocks_pct";
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_old_blocks_pct | 37    |
+-----------------------+-------+
1 row in set (0.001 sec)
```

还有另一个可调整的参数用于LRU优化，`innodb_old_blocks_time`，用于表示页读取到mid位置后需要等待多久才会被加入到LRU列表的热端，默认是1000，可以通过命令`set global innodb_old_blocks_time=0;`设置不移到前面，以防止真正的热点数据后续还是会被刷出。

```mysql
Mysql> show variables like "innodb_old_blocks_time";
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_old_blocks_time | 1000  |
+------------------------+-------+
1 row in set (0.001 sec)
```

为什么要对LRU进行优化呢？比如当批量查询数据时，需要访问表中的许多页甚至全部页，而这些页可能只用这一次，这大量的页如果占据了LRU链表当前最前端的页，就会导致原本被频繁访问的LRU的头部的页被刷出，后续再访问这些真正热点的页时就得访问磁盘。

除此之外，innodb1.0版本开始支持压缩页的功能，原本16kb的页可以压缩到几kb，非16kb的页是通过unzip_LRU列表进行管理的，命令`show engine innodb status\G`可以看到。

```mysql
LRU len: 5669, unzip_LRU len: 0
```

### 脏页的刷盘处理

Buffer pool中LRU链表上的数据页和索引页，如果被修改了的话，这个页就被称为脏页，即buffer pool中的数据，和磁盘上存储的数据产生了不一致。脏页会被InnoDB保存到**Flush链表**，然后在一定时机通过**checkpoint**机制将脏页刷到磁盘。通过show engine innodb status\G可以看到当前的脏页数量。

```mysql
Modified db pages  0
```

**Flush列表**：即脏页列表，LRU链表中被修改的页会拷贝一份到FLush列表，即脏页同时存在于LRU列表和Flush列表中，两个列表的功能不一样，互不影响。

#### checkpoint刷盘机制

除了Page Cleaner或者Master线程会以1s或者10s一次定期的将Flush列表中的脏页刷盘，还有几种情况下会触发checkpoint机制进行脏页刷盘，比如redo log满了，LRU链表满了等。

- **redo log满了**

  redo log是大小固定的文件，用于MySQL宕机时的灾难恢复，可以保证MySQL宕机重启时不会丢数据。redo log可以理解为一个环形的文件，当redo log写满时需要擦除redo log前面的内容才能继续写，此时就需要进行checkpoint刷盘。

- **LRU链表满了**

  当LRU链表中尾部需要淘汰的数据页是脏页时，就要强制触发checkpoint刷盘。LRU链表一般需要保证有100个左右的空闲页可用，当不足时，就会从列表尾端淘汰一些。

- **脏页数量太多**

  参数`innodb_max_dirty_pages_pct`可以控制这个阈值，当超过这个阈值时，就会触发刷盘操作。

#### 脏页刷盘的策略

checkpoint有两种：Sharp checkpoint和Fuzzy checkpoint。

- **sharp checkpoint**

  发生在数据库关闭时，会将所有的脏页都刷到磁盘。

- **fuzzy checkpoint**

  其余的场景，都是从Flush列表中只刷一部分脏页到磁盘。

#### 刷新邻接页

当某些场景触发了checkpoint，需要刷脏页时，InnoDB会自动检测该页所在区的所有页，如果也是脏页，那么就会一起刷了，多个脏页的刷新合并成一个IO操作，以提高效率，对机械磁盘比较有效。

但是刷新邻接页也存在问题：

1. 邻接页可能是一个不怎么脏的页，刷了这个页后该页又很快变成了脏页。
2. 固态硬盘有较高的IOPS（每秒输入输出量），可能并不需要这个。

针对以上问题，从1.2版本开始提供参数`innodb_flush_neighbors`，可以选择开启或者不开启这个特性，默认是开启状态

```mysql
Mysql> show variables like "innodb_flush_neighbors";
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_flush_neighbors | 1     |
+------------------------+-------+
1 row in set (0.001 sec)
```



### Redo log buffer

Buffer pool中除了有数据页和索引页之外，还有redo log的buffer，在事务中每次写入redo log实际上也是先写入redo log buffer，然后根据redo log的刷盘策略将redo log刷到磁盘。通过参数`innodb_flush_log_at_trx_commit`设置，它有0、1、2三个可选参数：

- 0：延迟写：每次事务提交先写道redo log buffer，然后每隔1s将redo log写入os buffer，并立刻调用fsync()刷到磁盘，这样如果系统崩溃的话最多会丢失1s的数据。
- 1：实时写，每次事务提交，立刻刷到磁盘，这种方式不会丢数据，但是因为每次都访问磁盘，所以性能稍差，但是redo log的写入是顺序的，相比与内存的随机IO可能还差不多，为了安全期间，大部分时候还是设为1的。
- 2：实时写，延迟刷，每次提交事务，都先写到redo log buffer，并写到os buffer，然后间隔1s调用fsync()刷到磁盘，这种是个折中的方案。和0的差别是如果MySQL挂了但是机器没挂可以保证不丢数据，如果机器挂了，那么也最多会丢失1s的数据。

Redo log buffer的大小和已通过参数`innodb_log_buffer_size`设置，默认是8M，这个参数需要设置一个合理的值，太小就会频繁的满然后频繁的触发checkpoint，太大会让MySQL重启时数据恢复时间变长。



### 额外内存池

注意到buffer pool组成的图里面，还有一个额外内存池，这个内存池就是InnoDB对它运行过程中本身的一些数据结构对象进行存储的。

### Insert buffer/Change buffer

insert buffer是为了优化二级索引（非唯一）刷盘时的插入效率，它也存在buffer pool，并且会持久化。通过`show engine innodb status`可以查看到相关信息：

```mysql
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
insert 0, delete mark 0, delete 0
discarded operations:
insert 0, delete mark 0, delete 0
```

#### 刷盘时二级索引的插入问题

当插入一个新行时，如果这个行要插入的页不在buffer pool，就需要先把这个页从磁盘取出来。对于主键索引，由于它一般都是递增的，所以直接从磁盘取出最后一个页，把新插入的行往B+树的最后按顺序加即可。而对于二级索引来说，新加的这行的索引字段往往不是最后面的，不能顺序插入，就需要随机IO磁盘，拿到这行数据应该插入的页，然后保存这个页到buffer pool，然后插入到buffer pool中这个页的B+树上，这里就有随机的磁盘IO了，导致效率较低。插入缓存就是为了解决这个问题，**提高二级索引的插入效率**，它既存在buffer pool中，也存在磁盘上，重启后，也可以从磁盘中加载回来还能用。

#### Change buffer的使用要求

1. 索引不是聚簇索引，因为聚簇索引不需要优化。
2. 索引不是唯一索引，因为要保证唯一的话，要去遍历二级索引判断是否重复，无论怎样都要遍历，优化没意义。

#### Change buffer的使用过程

1. 当有一行数据要插入/更新到某个表时，数据需要插入到这个表的主键索引和所有二级索引上，首先检查这行数据要写入的页在不在buffer pool的LRU链表上。
2. 如果主键索引页和二级索引页都在LRU链表上，则可以直接操作内存直接更新。
3. 如果主键索引不在LRU链表上，则直接顺序IO磁盘，拿到主键索引的最后一个数据页放到LRU链表，然后插入即可
4. 如果二级索引页不在LRU链表上，则把这个插入操作先放到insert buffer中。（insert buffer在事务提交后也会持久化到共享表空间的ibdata文件，并且也会写入redo log中），并不去随机IO磁盘，然后之后根据一定频率和情况进行**merge**，merge的过程就是InnoDB再进行随机磁盘IO，把这个二级索引的数据页从磁盘加载到buffer pool的LRU链表上，然后再把insert buffer和buffer pool中的这个页进行merge操作，同时这个页变成脏页。

#### Change buffer是什么数据结构

Change buffer是一颗B+树，在mysql4.1版本前一张表有一颗，存储的是这张表的所有二级索引，后续改为了全局只有一颗B+树，负责所有的二级索引

#### Merge发生的时机

1. 如果change buffer中的记录操作的二级索引页刚好被从磁盘读入buffer pool。
2. Change buffer的空间用光了，不能再写了，必须先从磁盘随机IO取出二级索引页先进行merge，给change buffer腾出空间。
3. Master线程会周期性的进行merge。

#### Change buffer的问题

在写频繁的情况，如果buffer pool中没有需要操作的二级索引页，就都需要写insert buffer，导致insert buffer过大，占了buffer pool的大部分。（mysql5.5之前的叫做insert buffer，InnoDB1.0Z版本后改为**change buffer**，顾名思义，不仅是插入，修改和删除操作也做了优化），1.2版本开始可以通过参数`innodb_change_buffer_max_size`来控制change buffer的最大使用内存，最大值是50，默认是25，表示最多是25%的buffer pool的空间。

如果mysql某时刻宕机了，就可能磁盘中真实存储的二级索引页还没和insert buffer合并，这时用ibd文件恢复的话会不成功，需要进行repair table来重建表的二级索引。

#### Change buffer不适用于哪些场景

因为 merge 的时候是真正进行数据更新的时刻，而 change buffer 的主要目的就是将记录的变更动作缓存下来，所以在一个数据页做 merge 之前，change buffer 记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大。对于写多读少的业务来说，页面在写完以后马上被访问到的概率比较小，此时 change buffer 的使用效果最好。这种业务模型常见的就是账单类、日志类的系统。假设一个业务的更新模式是写入之后马上会做查询，那么即使满足了条件，将更新先记录在 change buffer，但之后由于马上要访问这个数据页，会立即触发 merge 过程。这样随机访问 IO 的次数不会减少，反而增加了 change buffer 的维护代价。

#### change buffer和redo log的区别

**redo log优化的是向磁盘随机写的操作(改为顺序写)，而change buffer解决的是从磁盘随机读二级索引的问题（先记录，后merge）。**

总体而言redo log和change buffer的执行顺序是：、

1. 当执行写入操作时，如果要操作的页在内存中，就直接更新内存即可。
2. 如果页不在内存，则需要从磁盘加载这个页，如果要加载的页是二级索引的页，则可以使用change buffer先记录下来这次修改。如果是唯一索引，则不能使用change buffer，只能老老实实的加载数据页到内存，然后还要遍历唯一约束。
3. 然后就可以将上述操作记录在redo log中（如果用了change buffer，那么在redo log中也会有记录）。
4. 然后就是redo log的两阶段提交了。

### 自适应哈希索引

Buffer pool中还包含了一块叫做自适应哈希索引的区域。故名思意，就是存储哈希索引。哈希索引在等值查询时，改用hashmap提高B+树的访问速度。通过参数`innodb_adaptive_hash_index`设置是否开启，默认都是开启状态。

MySQL中只有memory引擎支持和哈希索引，innodb是自己加了个自适应的哈希索引。

哈希索引的创建规则是：相同的查询，并且都是等值查询的情况使用索引访问了17次，相同模式的查找访问了超过了100次，这个数据就会成为一个热点数据，那么就会给这个查询建立一个哈希索引，下一次的相同查询可以直接从哈希索引中取得，就不用再去buffer pool中去访问B+树和磁盘上的数据页了。

## Double write

### 部分写失效的问题

buffer pool中的脏页在触发checkpoint刷新到磁盘时，都是**以页为单位**刷的，一个页是16kb，而**文件系统是以4kb为单位**写入的，**机械磁盘是以扇区（512字节）为单位**写入的，这就导致16kb的脏页被刷到磁盘上**不是一个原子操作**。可能会发生宕机时，一个页只有4kb写入了磁盘，剩下的没写入。

**redo log的写入单位和磁盘的扇区大小一样**是512字节，所以redo log的写入是原子的，不会产生坏页。脏页刷盘时如果一个页没刷完，理论上是可以通过redo log的checkpoint点来继续刷的，但是由于**redo log是物理逻辑日志，不是纯物理日志**，不能用redo log进行恢复。

```
物理日志: 记录的是一个页中发生改变的字节，这样重复多次执行也不会导致数据不一致，它是幂等的，但是日志量会比较大
逻辑日志：像binlog，记录的就是一条条操作语句，它不是幂等的，但是日志量比物理日志小的多
物理逻辑日志：redo log,对应的页是物理的，但是页内部的操作是逻辑的，物理逻辑日志不是完全幂等的，
如果页本身发生了损坏，那用redo log是恢复不了的
```

### 两次写

部分写问题的解决方式：**当一个页发生部分写失败时，需要还原这个页到之前的状态，再通过redo log进行重做**。两次写就是InnoDB解决这个问题的方式。

#### Double write的组成

内存部分doublewrite buffer：2Mb。

磁盘上的共享表空间ibdata：2Mb，128个页，120个页用于批量刷新脏页，8个页用于单页刷新。

![](/images/innodb/double_write.png)

#### 刷脏页时double write的执行过程

1. 先通过memcpy函数将脏页复制到内存的double write buffer。
2. 之后double write会分两次，每次1Mb的顺序的把页写入磁盘上的ibdata文件，这个过程是直接调用fsync函数立刻写入磁盘的。因为是顺序写，所以开销并不是很大。
3. 完成磁盘上的double write写入后，再把double write buffer中的脏页写入磁盘上的各个表空间文件。

#### 刷脏页时MySQL崩溃的恢复过程

1. InnoDB会先去共享表空间找到double write中该页的副本
2. **如果该副本不完整，则说明上次刷盘时，double write写入磁盘时就失败了，还没到后面真正刷脏页的过程**，真正的数据页没有写了一部分的问题，它是完整的，那么就可以丢弃这个副本，直接用redo log恢复即可。
3. **如果副本完整，则说明上次刷盘时，已经过了double write写磁盘的阶段**，表空间中页就有可能是不完整的（当然，也有可能是完整的），不管它完不完整都可以通过完整的double write副本直接恢复这个页，然后再应用redo log看看需不需要恢复即可（redo log可以和double write中的数据副本对应上的，看看一不一致，不一致就用redo log恢复，一致就跳过即可）。

可以看到，就是因为redo log写磁盘一定会成功，无法通过redo log判断刷脏页时是否

通过命令`show global status like "innodb_dblwr%"\G`可以观察double write的运行情况：如下一共写了14383个页，写的次数是155。

```
Mysql> show global status like "innodb_dblwr%";
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Innodb_dblwr_pages_written | 14383 |
| Innodb_dblwr_writes        | 155   |
+----------------------------+-------+
2 rows in set (0.001 sec)
```

