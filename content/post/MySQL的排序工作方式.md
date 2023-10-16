---
title: "MySQL的ORDER BY语句的工作方式"
date: 2022-02-20T13:26:00+08:00
Description: ""
Tags: [MySQL]
Categories: [数据库]
DisableComments: false
---

在日常开发应用时，经常会碰到根据某个字段排序后显示结果，那么 MySQL 底层对这种需要排序的查询到底是怎么处理的呢？

首先我们构造一张表：

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;
```

这个表有 `id`、`city`、`name`、`age`等几个字段，主键是 `id`，`city` 字段创建了普通索引。

假设一个查询语句是这样的：

```mysql
select city,name,age from t where city='杭州' order by name limit 1000;
```

查询所有 city="杭州" 的city 、name和 age字段，然后按照 name 字段排序，并且限制1000条。



首先我们通过 `explain` 语句查看这条语句的执行计划：

![](/images/orderby/explain.webp)

观察到 extra 字段中说明 `Using filesort`，这意味着该语句需要在MySQL排序后才能输出结果。MySQL对数据集的排序是使用一块专用的内存，叫做 sort buffer，sort buffer是位于MySQL server层的。

**sort buffer**：每个查询线程，MySQL都会给它分配一个固定大小的sort buffer用于排序，sort buffer的大小通过参数 `sort_buffer_size` 控制，默认是 256kb。

#### 全字段排序

使用sort buffer后，上述这条语句的最常规的执行过程大概如下：

1. 首先初始化这个线程的 sort buffer，确定需要查询name、city、age 三个字段；
2. 查询where条件是 `city="杭州"`，city 字段可以查询索引，通过访问二级索引找到所有 city="杭州" 的行的主键 id ；
3. 然后用主键回表，查询到对应的完整行，取出name、city 和 age 三个字段，存入到 sort buffer 中；
4. 一行一行的用主键查询出所有需要的行，然后在sort buffer中按照name做排序；
5. 排好序后，取前1000行数据返回给客户端。

上述的这个过程，就叫做**全字段排序**，就是说所有字段都加载到了sort buffer然后做统一的排序后返回。

其中，第4步，按照name排序的从操作，可能是完全在内存中完成的，也可能要利用外部的磁盘文件协助排序，主要是根据需要排序的数据量的大小，如果数据量 sort buffer放的下，则就可以直接用内存完成排序，如果数据量过大导致 sort buffer 放不下，则需要借助外部的磁盘文件。借助磁盘文件排序使用的是归并排序算法。

#### row_id 排序

在上述的全字段排序中，如果我们select要查询的行数太多，导致 sort buffer 严重不足，就会使用大量的磁盘文件排序，导致大量的磁盘IO，性能会很差。

MySQL中规定了一个参数 `max_length_for_sort_data`，这个参数指定了 sort buffer 中一行数据的大小，如果一行太大超过了这个参数，MySQL就会放弃使用全字段排序的方式，改为使用**raw_id 排序**。

raw_id 排序的过程大致如下：

1. 首先初始化sort buffer，只确定放入两个字段 id 和 name；
2. 然后查询 city 的索引，找出满足条件的行 id；
3. 然后回表，通过 id 拿到整行数据，但是每行只取出 id 和 name 两个字段存入sort buffer，不取city 和 age 字段，使得取出的一行数据小于 `max_length_for_sort_data`的大小；
4. 根据 id 一行一行取完后，然后在 sort buffer 进行根据name的排序；
5. 最后按照排好的顺序，根据 id 再次回表查询到 city 和 age 两个字段；
6. 组合后将 city、name 和 age 三个字段返回客户端。 

整个过程会访问一次 city 的二级索引，然后两次回表操作。

---

很明显，上述两种 order by 的实现方式，raw_id排序相比全字段排序，多回了一次表，性能肯定会受影响。所以MySQL只有在 sort buffer 可能不够的情况下才会使用 raw_id 这种排序方式，还是会优先使用全字段排序。

 #### 优化

一、sort buffer的优化

不管是全字段排序，还是 raw_id 排序，都需要在sort buffer里进行排序，这都是因为从存储引擎取出的数据本身是乱序的，如果我们存储时就是有序的，那么查询时就不需要用到sort buffer了，性能就能大大优化。

那么上面那个例子，怎么才能让根据 city 查询出来的行，天然就是按照 name 排序呢，答案就是建索引，我们给(city, name)两个字段建立一个联合索引，那么当我们访问这个联合索引取出`city="杭州"`的数据时，name字段就是天然有序的了(联合索引相关内容可以查看[MySQL的索引](https://yanghairui.life/post/mysql%E7%9A%84%E7%B4%A2%E5%BC%95%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)这篇文章)。

那么整个过程就变成：

1. 通过(city, name)这个联合索引找出满足`city="杭州"`这个条件的记录，联合索引可以把city、name和id查出来，并且name是排好序的；
2. 然后根据id进行回表，直接取出name、city、age三个字段的值，可以一边回表一遍返回，因为name是已经排好序的了。

二、回表的优化

上述的过程还是需要回表，如果我们可以直接通过二级索引就拿到需要的city、name、age的值不是更好，那么解决方式也很容易想到，把(city, name)联合索引改成(city, name, age)三个字段的联合索引，则我们就可以通过这个联合索引，拿到所有想要的字段，不需要再回表了。这就是**覆盖索引**优化。

当然了，如果这种类型的查询并不那么多的话，三个字段的联合索引还是比较占空间和插入效率的，所以还是需要根据实际情况酌情考虑，选择一个最有性价比的方案。







