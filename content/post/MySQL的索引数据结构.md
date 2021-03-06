---
title: "MySQL的索引"
date: 2021-04-20T17:39:38+08:00
Description: ""
Tags: [MySQL,数据结构]
Categories: [数据库]
DisableComments: false
draft: false
---

## 数据库索引

索引是一种数据结构，可以为不同的场景提供更好的性能。对于数据库来说，索引的功能就是**提高数据查询的效率**，但同时不可避免的也会增加数据写入的消耗，二者之间需要一个平衡，只要当查询带来的好处远远大于写入时的额外工作时，索引才是有意义的。

数据库表建立索引的主要优势：

- 大大减少服务器需要扫描的数据量
- 索引可以帮助服务器避免排序和临时表
- 索引可以将随机IO变为顺序IO

### 常见的索引数据结构

- 哈希表：

  key-value形式的存储，可以O(1)时间复杂度查询，缺点是**只能等值查询**，没办法范围查询。

- 有序数组

  在**等值查询和区间查询**都非常优秀，等值查找用二分法，时间复杂度是O(logN)，区间查找就用二分先找到一个边界，然后向一侧遍历即可。缺点是**插入删除太麻烦**，每插入删除一条数据要动整个数组。所以只适用于**静态存储**，不怎么需要插入删除的场景下比较合适。

- 二叉搜索树

  特点是左子节点的值比右子节点小，所以也是二分法查找，这样查找插入删除都不差，时间复杂度都是O(logN)，缺点是当左子数或右子树中没值时**会退化成链表**，查找效率大大降低，并且也不支持范围查找。

- 二叉平衡树

  解决二叉搜索树可能退化成链表的问题，各方面都比较好了，但是只有二叉，数据大的时候，树的层数会很高，一次查询可能要很**多次访问磁盘**，查找效率依然不令人满意，并且同样不支持范围查找。

- B树

  B树就是多叉平衡树，为了二叉搜索树尽可能减少访问磁盘，把二叉升级为多叉，N值取决于数据块的大小，因为要控制树的深度，所以数据块越大，N值也会越大。B树是很**适合磁盘存储**的数据结构。

- 跳表

  跳表是在链表之上加上多层索引构成的，支持快速插入、查找、删除数据，时间复杂度都是O(logn)，并且调表也支持按照区间快速查找数据，看起来是比较完美的索引数据结构了，Redis采用了这种数据结构。

- B+树

  跳表虽然很好，并且实现也很简单，但是对磁盘IO上没有B树友好，所以Redis这样的内存数据库选用了，而对于大部分需要持久化到磁盘的数据库来说，还是需要更少磁盘IO的B树比较合适，B+树对B树做了两点数据结构上的优化：

  1. 叶子节点之间用链表连接了起来，以优化范围查询。
  2. B+树分成了索引节点和数据节点，索引节点不存数据，只让叶子节点存数据，这样做有两点好处：
     1. 查询效率会更加平衡，MySQL中的B+树一般是2-4层，索引节点不存储数据，只有数据节点存储数据，这就导致所有数据的访问都是2-4次磁盘IO就可以拿到数据了，很稳定。而B树所有节点都存储数据，虽然当数据在高层节点时，访问速度可以很快，但是是非常不平均的，高层的数据就快，底层的就慢，稳定性很差。
     2. 由于索引节点不存数据，所以一次IO可以拿到更多的索引值，而B树中索引节点有存储数据，如果不是我们想要的，就浪费了大量的空间，一次磁盘IO从B+树获取的信息量远大于B树。

这篇文章主要还是聚焦于MySQL的索引实现。

### MySQL中的索引类型

#### B+树索引

MySQL中MyISAM和InnoDB对B+树索引的使用有所不同，后续会详细讲解。

#### 空间数据索引(R-Tree)

MyISAM存储引擎支持空间数据索引，但是MyISAM做的并不完善，GIS做的较好的是PostgreSQL和PostGIS。

#### 哈希索引

MySQL中只有Memory存储引擎显式支持哈希索引，InnoDB也有自适应哈希索引的存在（具体可查看[InnoDB存储引擎特性](https://yanghairui.life/posts/innodb%E7%89%B9%E6%80%A7/)）。哈希索引的底层实现就是上述的哈希表，只能进行等值查询。

#### 全文索引

MyISAM和InnoDB都支持全文索引。全文索引一般通过**倒排索引**来实现，倒排索引也是一种数据结构，它在辅助表中存储了单词与单词自身在一个或多个文档中所在位置的映射，通常利用关联数组实现。

#### 分形树索引(Fractal tree)

第三方的存储引擎TokuDB支持这种索引，它有B+树的很多优点，也避免了一些缺点，它可以优化写路径；大部分二级索引的写操作是异步的。 分形树是一种写优化的磁盘索引数据结构，写操作性能非常好，同时读操作还能近似于B+树（略低）；并且这种数据结构还天然支持可以在写操作的同时修改表结构。

在写多读少的场景下这种存储引擎比较合适，缺点是资料较少。

## B+树

![](/images/index/B+tree.png)

如上图所示是B+树的结构。可以看到是以页为单位存储数据的，MySQL中的页大小默认是16k。高层索引节点中只存索引的key，低层数据节点(或者称为叶子节点)中才会存储value，并且数据页之间是以链表的形式连接起来的，包括不同索引页的叶子节点。并且链表的顺序是按照索引key的顺序存储的，当然这是为了方便按索引范围取数据。

#### B+树的插入操作

B+树的插入操作必须保证插入后叶子节点中的记录依然有序，插入B+树会有三种情况，可能会导致不同的算法：

1. 当索引节点和叶子节点都没有满时，可以直接将数据记录插入到叶子节点上
2. 当索引节点没满，但是叶子节点满了时，需要拆分叶子节点，拆分叶子节点时会创建出一个新的叶子节点，将旧页中中间的记录的索引值放入上层的索引节点中，然后小于该索引的记录放在旧的叶子节点，大于该索引的记录放在新的叶子节点中。
3. 当索引节点满了，叶子节点也满了，那么索引节点和叶子节点都要进行拆分。拆分过程和以上类似，都是创建一个新页，然后把原本的索引和记录对半分。

以上由于叶子节点满了导致页的拆分，称为**页分裂**。频繁的页分裂会有大量的数据移动十分影响性能，并且页分裂都是对半拆分的，会导致页中大量的空位造成空间浪费。

**旋转优化**：

由于页分裂的性能和空间浪费问题，B+树同样提供了类似二叉平衡树的旋转功能，来尽量较少页分裂。当叶子节点满了时，会去检查它兄弟节点是否也满了，一般左兄弟会被优先检查（尽量还是保证树是完全的），如果有空位，就会将此节点上的数据挪动一些到兄弟节点上，不用进行页分裂了。

**顺序插入优化**

总是从页的中间进行分裂会造成很多空间的浪费，如果我们的索引是顺序递增的，更常见的情况是顺序写满一个页后才开始进行页分裂，如果是对半分，而后续又继续在最后一页向后追加插入，那么原本的页就是又一半空间永远都不会被插入数据，造成空间浪费。

为了解决这个问题，InnoDB会在插入时根据情况决定是向左还是向右进行分裂，以及分裂点在哪个位置。

当页在插入新行会导致页写满时，InnoDB会检查要插入的这一行的后面原本有没有超过三条记录，如果有，就会取三条记录后作为页分裂点，如下图：

![](/images/index/right_splite.png)

而如果新插入的记录是这个页的最后一条记录的话，则会以自己作为分裂点，如下图：

![](/images/index/no_splite.png)

即如果**完全顺序插入的话，写满一个页后就会新创建一个页继续写，不会在页从中间进行拆分**。而如果是随机插入则会遵循对半分的页分裂策略。

#### B+树的删除操作

B+树使用**填充因子**来控制树的删除变化，50%是填充因子可以设置的最小值。B+树需要保证删除后叶子节点的记录依然有序，和插入类似，删除操作也会有三种情况：

1. 删除后叶子节点和索引节点都不小于填充因子时，可以直接将该记录从叶子节点删掉，如果在索引节点上就从索引节点也删掉。
2. 删除后叶子节点小于填充因子，而索引节点不小于填充因子时，需要在删除后将该叶子节点和它的兄弟节点合并，以保证叶子节点的填充因子满足要求。
3. 删除后叶子节点和索引节点都小于填充因子，那么就需要先合并叶子节点，然后更新索引节点，索引节点也满了，那么在合并索引节点。

以上由于叶子节点的填充因子小于预设值时导致的**页合并**，页合并同样也需要挪动大量数据，造成性能浪费。



## InnoDB对B+树索引的使用

InnoDB存储引擎支持主键索引、唯一索引、普通索引和全文索引。

#### InnoDB存储引擎对表的组织

InnoDB是**索引组织表**。InnoDB存储引擎的每张表都会有一个主键，如果没有指定，在创建表时，InnoDB会生成一个名叫row_id的隐藏字段作为该表的主键。然后**InnoDB会根据主键生成一颗B+树，并在这颗B+树种存放表中的所有数据**。这颗B+树也是主键索引的B+树，一个表其实就是一颗主键索引的B+树，它的索引节点上存储的是主键，**叶子节点上存储的是主键和数据行**，因此主键索引也叫**聚簇索引**。

而对于表的每一个其他的索引，也都会创建出一颗B+树，可以说，**一个表有几个索引，那么它就有几颗B+树存储在磁盘上**。因为主键索引上存储着完整的数据行，所以其他索引就没必要存储完整的数据行了，其他索引B+树的索引节点上存储的是索引列的数据，**叶子节点上存储的是索引列和主键值**。

基于以上差别，在通过主键索引和非主键索引进行查询整行数据时，比如`select * from T;`，**使用主键索引只需要搜索一次B+树**，而**非主键索引则需要搜索两次B+树**，第一次根据索引列找到主键值，然后再根据主键拿到数据行（后面根据主键值再查一次的操作被称为**回表**），才能获取到整行数据。

#### 覆盖索引

当可以通过直接扫描k索引树就能得到结果，而不需要回表时，这就称之为**覆盖索引**，覆盖索引可以大幅提升性能，是一种常用的优化手段。

如我们前面所说，非主键索引的B+树叶子节点中存放的的是索引值和主键，那么我们要通过这个索引查询到某个非主键的其他字段时，就需要根据得到的主键值，再查一次主键索引，从主键索引中拿到完整的数据行，再获取想要的字段，我们说过，一颗B+树大概2-4层，访问一次需要2-4次磁盘IO，我们访问了两颗B+树的话，实际上就需要4-8次磁盘IO，这样性能是很差的。所以如果要查的字段通过非主键索引就能拿到的话，那么性能就大大提升了。

#### 自增主键的好处

如上所说，B+树为了保持有序性，在插入新值时要做维护。如果插入的行在最后的话，就只需要在树的末尾添加，但如果是在树的中间插值，就需要**挪动后面的数据**空出位置。如果又恰好此时这一个数据页满了，就还要进行**页分裂**操作，并且随机插入不能用到B+树的**顺序插入优化**，性能会受到影响，同时空间也会有大量浪费。

删除数据时，如果页的利用率很低（可能因为页分裂造成了很多空位，也可能是删除操作后没有重新**optimize table**），会将数据**页合并**，同样性能也会受影响。

基于页分裂的特性，InnoDB在插入时，**尽可能的在最后按顺序插入**，这样既不需要在插入时挪动数据，也会减少页分裂的发生，这也是**自增主键**的好处。同时自增主键的占用空间上也要比UUID之类要小，还不用担心重复的问题。自增主键还可以进行方便的进行范围查询。

#### Cardinality

一般我们都会有经验，像性别，类型等字段我们都不会给它建立索引，因为他们的取值范围很小，只有固定的几个值，称为**低选择性**。这种情况下B+树索引的优化几乎等于没有，因为建立起来的索引就没几个节点，相反，如果一个字段的取值范围很广，少有重复的值，那它就是高选择性，这时给它建立一个B+树索引就能提升查询效率。

通过`show index from  table`命令结果中的Cardinality字段可以观察索引的选择性，Cardinality是一个预估值，并不是准确值，基本上我们期望Cardinality/总行数趋近于1比较好，如果太小，那么就需要考虑是否有必要创建这个字段的索引。

```mysql
Mysql> show index from titles;
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table  | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| titles |          0 | PRIMARY  |            1 | emp_no      | A         |      442367 |     NULL | NULL   |      | BTREE      |         |               |
| titles |          0 | PRIMARY  |            2 | title       | A         |      442367 |     NULL | NULL   |      | BTREE      |         |               |
| titles |          0 | PRIMARY  |            3 | from_date   | A         |      442367 |     NULL | NULL   |      | BTREE      |         |               |
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
3 rows in set (0.000 sec)
```

Cardinality值的统计是通过采样来判断的，这也是这个值可能不准确的原因。InnoDB会在一定的时机触发执行Cardinality的统计，通过参数`innodb_stats_sample_pages`可以设置采样的页数，默认是8，显然值越大统计的越准，但也会增加消耗。

#### 前缀索引

InnoDB允许给一个字段建立前缀索引，即可以取字段前缀的一部分作为索引，类似于BLOB、TEXT或者很长的VARCHAR类型的列，必须要使用前缀索引。创建前缀索引的关键就是要尽量选择短一点的前缀时，并且还要有比较高的选择性，让前缀索引的选择性尽可能趋近于索引完整列。

比如类似邮箱类型的字段，后缀都是固定的字符串，就非常适合前缀索引；其实对于这样的字段，还可以在数据库中直接只存储前缀，然后给完整的列创建一个索引。

#### 联合索引

InnoDB不仅允许一个字段建一个索引，还可以多个字段一起建立一个索引，节省空间。联合索引就是多个字段一起建一个索引。联合索引也是一颗B+树，只不过存储的key不是一个字段，而是多个字段，比如我们对字段name和字段age建立一个联合索引，那么这颗B+树大概就是长这样：

![](/images/index/coopration.png)

这个普通索引B+树的key就变成了两个字段，并且是按照创建索引的顺序排列的。因此，对于联合索引的查询，需要满足**最左前缀法则**。比如上面这个例子，我可以通过name查询，或者通过（name，age）查询时都可以用到这个索引，但是如果通过age来查询，则用不到这个索引。如果非要查询age，那就只能再给age字段也加一个索引。

最左前缀不仅可以查询完整字段的最左前缀，如果字段是字符串，还可以指定字段一部分的最前缀。比如对(first_name, last_name)建立联合索引，那么当我们查询条件时first_name="Harry"，last_name LIKE "Y%"时，也符合最左前缀法则。

#### ICP优化

MySQL5.6版本以后增加了**索引下推优化**（Index Condition Pushdown，ICP），用于在仅能利用最左前缀索的场景”下（而不是能利用全部联合索引），对不在最左前缀索引中的其他联合索引字段加以利用——在遍历索引时，就用这些其他字段进行过滤。

比如如果我要查name字段第一个字是"张"，并且age=10的行，那么按照最左前缀法则，这个查询用不到联合索引，只能用name字段先查到以"张"开头的行，再回表根据age字段进行过滤。而索引下推就可以在遍历的过程中，也能检查并过滤不满足条件的age的值。

通过explain执行计划中的Extra字段中有"Using index condition"，就代表这个查询会用到索引下推。

#### MMR优化

MySQL5.6版本以后还支持了Multi-Range Read(MMR)优化，MMR优化的主要目的就是为了减少磁盘的随机访问，将随机访问尽量转换为偏顺序的访问。MMR优化适合于range、ref、eq_red类型的查询。

MMR的原理：

比如一个表中有(id, age, name)三个字段，id是主键索引，age是普通索引，当我们想查询某个范围的age对应数据行的name字段时，由于不是覆盖索引，需要先通过age字段的B+树，找到id字段，然后通过id字段查询主键索引的B+树，即回表，拿到所有行信息后即可拿到name字段。而在访问age字段的B+树时，age是一个顺序，可以用到range，并且得到的(age, id)数据集是以age进行排序的，然后再逐行根据id去回表，而id往往此时不是顺序的，即回表时会有很多的随机IO操作。MMR优化就是当获取到的(age, id)先根据id进行排序，再去主键索引上查询，这样随机IO就变成较为顺序的IO操作了，并且主键有连续的还可以根据主键批量的查询，效率就得到了提升。

总结就是，对于使用普通索引范围查询和join查询，并且需要回表时：

1. 查询普通索引得到数据集，此时是根据普通索引排序的，
2. 将数据集根据主键进行排序，
3. 然后可以顺序IO和批量操作完成查询。

## MyISAM的B+树索引

MyISAM的主键索引和InnoDB的主键索引虽然都使用了B+树，但是二者的数据分布不同。InnoDB是索引组织表，即一个表其实就是一颗索引树，MyISAM存储表数据不是这样的，MyISAM的表数据是按顺序存储在磁盘上的，如下图：

![](/images/index/myisam_table.png)

可以看到，类似我们从MySQL客户端查询时看到的表一样，数据是一行一行存储的。这种方式类似于前面所说的数组索引，可以通过跳过固定的字节数找到需要的行（MyISAM并不总是使用行号，而是根据字段定长还是变长的行使用不同的策略）。

MyISAM的索引也是一颗B+树，但和InnoDB不同的是，它的**叶子节点上存储的是表实际存储位置的行号**（或者理解为行实际存储位置的指针），如下图：

![](/images/index/myisam_index.png)

MyISAM存储引擎的所有索引都一样，即所有的索引叶子节点上存储的都不是整行数据，而是”行号“。主键索引和普通索引唯一的不同只是它是唯一且非空的而已。