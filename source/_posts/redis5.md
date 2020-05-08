---
title: Redis知识归纳(五)持久化机制
date: 2020-04-03 13:32:39
tags: Redis
categories: Redis
---

Redis持久化可以归纳到也可以归纳到高可用的环节里面。

<!-- more -->

### Redis持久化的两种方式

1. RDB: 属于快照模式，即对数据进行周计划的持久化
2. AOF: AOF机制对每条写入命令作为日志，以append-only的模式写入到日志文件中，在redis重启时，可以通过回放AOF日志中的写入命令来重新构建整个数据集。

当同时使用RDB 和 AOF 两种持久化机制，那么在Redis重启时，会使用AOF来重新构建数据，因为AOF中的数据更加完整。

### RDB的优缺点

RDB会生成多个数据文件，每个数据文件都代表了某一时刻中redis的数据，因此适合做为冷备。

RDB对Redis对外提供服务的影响很小，因为redis主进程只需要fork一个子进程，让子进程执行磁盘IO操作来进行RDB持久化，子进程将数据集写入到一个临时RDB文件中，当完成之后，用新的RDB文件替换原来的RDB文件，并删除旧的RDB文件。这种机制可以从 “写时复制”中收益。我们在学习 copyonwriteArrayList时提到过。

相对于AOF持久化机制来说，直接基于RDB文件来恢复和重启redis进程，更加快速。但是当redis出现故障时，尽可能的少丢数据，那么RDB不如AOF，比如RDB每个5分钟持久化一次，那么如果宕机，使用RDB恢复则会丢掉5分钟的数据。

当数据集过大时，redis fork的过程是 非常耗时的。

### AOF的优缺点

AOF可以更好的保护数据不丢失，一般AOF会每个1秒，通过后台线程执行一次fsync操作，因此最多会丢失1秒钟的数据。

AOF日志文件以 append-only模式写入，所以没有任何磁盘寻址开销，写入性能高，而且文件不容易损坏。AOF文件即使过大时，会进行重写操作，不会影响客户端的读写。重写后的新AOF文件包含了恢复当前数据集所需最小命令集合。整个重写操作绝对安全，因为redis在创建新的AOF文件时，会继续将命令 追加到现有的 AOF文件里面，一旦新AOF文件创建完毕，那么redis就会将旧AOF切换到新AOF，并且对新AOF文件进行追加操作。

AOF文件的可读性很强，因此适合作为灾难性的误删除的紧急恢复。比如，如果别人不小心用flushall命令清空了所有数据，那么只要没发生重写操作，就可以立即拷贝AOF文件，将最后一条flushall命令删除，然后将AOF文件放回去，就可以通过恢复机制，自动恢复所有数据。

### RDB和AOF该如何选择？

同时开启两种持久化方式，AOF保证数据不丢失，作为数据恢复的第一选择；RDB来做不同程度的冷备，在AOF文件都丢失或者损坏不可用的时候，可以通过RDB来进行恢复。