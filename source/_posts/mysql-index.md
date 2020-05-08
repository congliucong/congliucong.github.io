---
title: MySQL之索引原理及性能优化
date: 2020-04-19 23:42:21
tags: Mysql
categories: Mysql
---

这篇文章主要接上篇文章，对Mysql的索引相关知识进行梳理。

<!-- more -->

### 索引

索引是存储引擎用于快速找到记录的一种数据结构。

### 为什么索引要使用B+Tree?

#### Hash索引

hash表示通过关键字段的哈希值通过哈希函数的转换映射到哈希表对应的位置上，因此查询效率非常高。哈希索引基于hash表实现的。但是hash索引存在着以下问题：

1. 只有精确匹配所有列的查询才有效，比如组合索引（A,B）上建立了哈希索引，如果只查询A，那么是无法使用该索引。
2. 哈希索引不是根据索引值顺序存储的，所以无法使用排序。
3. 哈希索引只能支持等值比较，不支持任何范围查找。
4. 如果大量重复键值的情况下，会存在大量哈希碰撞，哈希索引的效率会很低，

所以hash索引，只是用于特殊的场合。例如InnoDB引擎中的“自适应哈希索引”：如果InnoDB注意到某些索引列被频繁访问，它会在内存基于B+树索引之上再创建一个哈希索引 ，这样就能让B+树也具有哈希索引的优点。

#### B-Tree索引/B+Tree索引

B-Tree是一种多路平衡查询数。B+Tree是B-Tree的优化版本。

那么B-Tree和B+Tree树有什么区别呢？

1. B-Tree每个节点即保存索引，又保存数据。B+Tree的关键字非叶子节点存储的是索引，而非数据，只有叶子节点存放诗句。
2. B+Tree的叶子节点之间构成链表，遍历叶子节点就能获取全部数据。

因此，B+树可以使得树高更矮，从而降低磁盘IO次数，提高查询速度。所以在InnoDB引擎中，索引的数据结构默认是B+树。

#### 局部性原理与磁盘预读

局部性原理和磁盘预读我们在volatile分析时 缓存行那里介绍 过。大概意思是：当一个数据被用到时，其附近的数据也通常会被马上使用到，所以一般会预读长度为一页的整数倍。页是计算机存储的逻辑块，硬件和操作系统通常将主存和磁盘存储块分割为连续大小相等的块，每个存储块称为一页，主存和磁盘以页为单位交换数据。

#### B+Tree索引性能分析

所以有了局部性原理和磁盘预读，mysql利用这个原理，**将一个节点的大小设置为一页，这样每个节点只需要一次IO就可以完全载入**。因为B+树非叶子节点存储的是索引，而非数据，因此每个节点可以存储更多的索引，这样就能整体降低B+树的高度，从而减少磁盘IO次数。这也就是为什么我们要求索引字段尽量小，都是为了在每一个节点尽量存储更多的数据，而降低树高。

### Mysql索引实现

在Mysql中，索引属于存储引擎级别的概念，不同存储引擎对索引的实现是不同的。这里从MyISAM和InnoDB两个存储引擎的索引说起。

#### MyISAM存储引擎

MyISAM引擎使用B+数作为索引结构，叶节点的data域存放的是数据记录的地址。

![https://blog.codinglabs.org/articles/theory-of-mysql-index.html](mysql_index/9.png)

如上图所示，MyISAM的索引文件仅仅保存数据记录的地址。所以,MyISAM的索引文件和数据文件是分开的。在MyISAM中，主索引和辅助索引在结构上是没有区别的。

MyISAM的索引方式叫做"非聚集"的，这样称呼是与InnoDB的聚集索引区分开来。

#### InnoDB存储引擎

InnoDB也是使用B+树作为索引结构的，但是实现上面跟MyISAN不一样。

InnoDB中，索引文件和数据文件是同一份文件。表数据文件本身就是按照B+树组织的一个索引结构，这颗树的叶子节点data域保存了完整的数据记录。这个索引的key是数据表的主键，因此InnoDB表数据文件本身就是主索引。
![https://blog.codinglabs.org/articles/theory-of-mysql-index.html](mysql_index/10.png)

可以看出来InnoDB主索引的示意图，可以看出叶节点包含了完整数据，这种索引叫做聚集索引。InnoDB数据文件本身要按照主键聚集，所以InnoDB要求表必须要有主键，如果没有显示指定，那么Mysql会自动选择一个可以唯一表示你数据的列作为主键，如果不存在这样的列，那么InnoDB会自动生成一个隐含字段作为主键。

InnoDB的二级索引，叶子节点data域记录的是相应记录的主键值而非数据。
![https://blog.codinglabs.org/articles/theory-of-mysql-index.html](mysql_index/11.png)

所以，通过二级索引查询记录，要通过二级索引B+树查询到主键值，再通过主键聚集索引定位到行记录，一共需要两遍查询，这个就是叫做 **回表查询**。

所以，我们要求索引字段尽量小原因一是，主键聚集索引每个节点可以容纳更多的索引值，降低树高，又因为二级索引的叶子节点记录的是主键，主键过长也会导致二级索引的树高变高，从而影响查询效率。

### 索引使用策略及优化

#### 索引覆盖

索引覆盖指的是一个查询语句的执行，只需要从索引中就可以获取到，不必从数据表中值读取，就叫做索引覆盖。

应用场景：
1. 全表count查询。
2. 列查询回表优化。

#### 索引下推

索引下推是Mysql5.7推出的一项优化策略。**索引下推只作用于二级缓存**。意思是：在没有索引下推的情况下进行查询，存储引擎通过索引检索到数据，然后返回给Mysql服务器，服务器然后判断数据是否符合条件；在使用索引下推之后，判断条件直接会在存储引擎完成判断，将符合条件的返回，这样就能有效减少回表次数，大大提升查询效率。

#### 正确使用索引

1. 最左前缀匹配
在Mysql建立联合索引时会遵循最左前缀匹配原则，即最左优先，在检索数据时，从联合索引的最左边开始匹配。所以当创建联合索引时，如（k1,k2,k3），相当于创建了（k1）（k1,k2）（k1,k2,k3）三个索引。因此，如果查询条件是k3，则使用不到索引，如果查询条件是k1，k2，那么只能用到k1索引。

2. 全列匹配
索引中所有列都精确匹配，索引都可以被用到。理论上索引对顺序是敏感的，但是Mysql查询优化器会自动调整where子句的条件顺序来使用合适的索引。

3. 范围查询
范围列可以用到索引（必须是最左匹配），但是范围列后的列无法用到索引，并且索引最多用于一个范围列。即Mysql会一直向右匹配，直到遇到范围查询就停止匹配。

4. 查询条件中包含函数或者表达式
当条件中存在函数或者计算表达式，那么索引会失效。

5. 正确选择索引
索引文件本身也要消耗存储空间，同时索引会加重插入、删除和修改记录时的负担，因此索引也不是越多越好。所以索引的选择要选择那些区分度高的列作为索引。区分度计算 count(distinct clo)/count(*)。选择性越高的索引价值就越大。

并且对于主键索引的选择也很重要。如果不是特殊要求，要尽量使用自增字段作为主键。如果使用UUID或者一些没有顺序的字段作为主键，为了保证索引有序，Mysql每次插入的数据都要放在合适的位置，那么当一个快满或者已满的数据页中插入数据时，新插入的数据会将数据页写满，Mysql就要申请新的数据页，这样就导致了 **页分裂**，需要把上个数据页中的部分数据挪到新的数据页上，这个大量移动的过程非常影响插入效率。

### Mysql性能分析及优化

1. 查看Mysql服务器运行的状态值

**show status**: 一般关注 Queries、Threadsconnected、Threadsrunning的值，及查询次数、线程连接数。线程运行数，观察是由于访问量高导致数据库速度变慢还是其他原因。如果系统的并发请求并不高，且查询速度慢，那么可以直接记性SQL语句调优步骤。

2. 获取需要优化的SQL语句
   1. 查询运行的线程
   **show processlist**
   从返回结果中，我们可以了解到线程执行了什么命令/SQL语句以及执行的时间。其中state的值使我们判断性能好坏的关键，当出现如下内容，则代表这该行记录的SQL语句需要优化：
    Converting HEAP to MyISAM 查询结果太大，把结果放到磁盘里面。
    Create tmp table 创建临时表，严重
    Copying to tmp table on disk 把内存临时表复制到磁盘，严重
    locked 被其他查询锁住，严重
    loggin slow quert 记录慢查询
    sorting result 排序

   2. 开启慢查询日志
     在配置文件my.cnf中【mysqld】一行下面添加两个参数：

      slow_query_log = 1
      slow_query_log_file=/var/lib/mysql/slow-query.log
      long_query_time = 2
      log_queries_not_using_indexes = 1

    当有慢sql出现后，我们可以用Mysql提供的 mysqldumpslow 来查询是哪些语句执行时间长而导致慢查询的出现。

3. 分析SQL语句

上面的步骤可以筛选出有问题的sql之后，我们就可以分析sql语句。通过 **explain**命令来查看sql的执行计划情况。

```sql
mysql> explain select * from category;
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | category | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

一般关注 **type**:

system：表只有一行记录，相当于系统表
const：通过索引一次就找到，只匹配一行数据
eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常用于主键或唯一索引扫描
ref：非唯一性索引扫描，返回匹配某个单独值的所有行。用于=、< 或 > 操作符带索引的列
range：只检索给定范围的行，使用一个索引来选择行。一般使用between、>、<情况
index：只遍历索引树
ALL：全表扫描，性能最差

**key** ： 显示MySQL实际使用的索引，如果为null，说明没有使用索引查询。

**rows** : 大致估算出找到所需记录需要扫描的行，数值越小越好。

**extra** : 其他信息，
当出现 using filesort说明Mysql对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。
当出现 using temporary 说明使用了临时表保存中间结果。Mysql在对查询结果排序时使用临时表，常见于排序Order by 和 分组查询 group by。
当出现这两种情况的时候，就应该去优化sql。

4. 优化SQL语句
  1. 避免select * ,需要什么数据就查询对应的数据

  2. 小表驱动大表。当使用in时，先执行in中的子查询，然后执行外面的主查询；当使用exits时，先执行主查询，然后子查询作为内层循环。因此有A B两表，通过id字段关联。当A表数据大于B表时，应该使用 select * from A where id in (select id from B)；当A表数据小于B表时，应该使用select *  from A where exists(select 1 from B where B.id = A.id)。总之要保证小表驱动大表。

  3. 一般情况下，使用连接代替子查询，因为使用join,Mysql不会在内存中创建临时表。
  4. 适当冗余字段，减少表关联
  5. 合理使用索引值。比如为排序 分组字段创建索引，避免filesort出现等。（上面已经提到过）

3. 硬件优化
4. 分库分表






> 参考列表
> 1. https://blog.codinglabs.org/articles/theory-of-mysql-index.html