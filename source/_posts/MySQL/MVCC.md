---
title: MVCC
categories: 
- MySQL
---

**原文：** https://mp.weixin.qq.com/s/jxM7n_4Or52_-MlB4aK9Bw

我们知道，根据MySQL的锁机制，写锁和读锁是冲突的，所以MySQl通过MVCC（多版本并发控制）方式来处理读写冲突，提高数据库高并发场景下的吞吐性能

最早的数据库系统，只有读读之间可以并发，读写、写读、写写都要阻塞。引入MVCC后，只有写写之间相互阻塞，其他三种操作都可以并行，这样大幅度提高了InnoDB的并发度。

在内部实现中，InnoDB通过undo log保存每条数据的多个版本，并且能够找回数据的历史版本提供给用户读，每个事务读到的数据版本可能是不一样的

# 为什么需要MVCC

InnoDB相比MyISAM有两大特点，一是支持事务，二是支持行级锁

**「但并发事务处理也会带来一些问题，主要包括以下几种情况：」**

更新丢失（ Lost Update ）：当两个或多个事务选择同一行，然后基于最初选定的值更新该行时，由于每个事务都不知道其他事务的存在，就会发生丢失更新问题 ，最后的更新覆盖了其他事务所做的更新。

> ❝
>
> 如何避免这个问题呢，最好在一个事务对数据进行更改但还未提交时，其他事务不能访问修改同一个数据
>
> ❞

脏读（ Dirty Reads ）：一个事务正在对一条记录做修改，在这个事务并提交前，这条记录的数据就处于不一致状态；这时，另一个事务也来读取同一条记录，如果不加控制，第二个事务读取了这些尚未提交的脏数据

> ❝
>
> 官网对脏读定义的地址为 https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_read
>
> ❞

不可重复读（ Non-Repeatable Reads ）：一个事务在读取某些数据已经发生了改变、或某些记录已经被删除了

> ❝
>
> 官网对不可重复读定义的地址为 https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_non_repeatable_read
>
> ❞

幻读（ Phantom Reads ）：一个事务按相同的查询条件重新读取以前检索过的数据，却发现其他事务插入了满足其查询条件的新数据

> ❝
>
> 官网对幻读定义的地址为 https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_phantom
>
> ❞

以上是并发事务过程中会存在的问题，解决更新丢失可以交给应用，但是后三者需要数据库提供事务间的隔离机制来解决

**「实现隔离机制的方法主要有两种 ：」**

1.加读写锁，效率低

2.一致性快照读，即 MVCC

**「注意：」**

> ❝
>
> MySQL中 InnoDB 引擎支持 MVCC; 应对高并发事务, MVCC 比单纯的加行锁更有效, 开销更小; MVCC 在读已提交（Read Committed）和可重复读（Repeatable Read）隔离级别下起作用（下面介绍）
>
> ❞

# MVCC原理

总体上来讲MVCC的实现是基于ReadView版本链以及Undo日志实现的

首先对于使用InnoDB存储引擎的表来说，它的聚簇索引记录中都包含两个必要的隐藏列

- trx_id：每次一个事务对某条聚簇索引记录进行改动时，都会把该事务的事务id赋值给trx_id隐藏列。
- roll_pointer：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到undo日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息

> ❝
>
> undo log主要存储的也是逻辑日志，比如我们要insert一条数据了，那undo log会记录的一条对应的delete日志。我们要update一条记录时，它会记录一条对应相反的update记录
>
> ❞

> ❝
>
> 聚簇索引简单说就是表中记录的物理顺序与键值的索引顺序相同，一个表只能有一个聚集索引，这类索引是和数据存在一起的
>
> ❞

现在比方说我们的表test现在只包含一条记录

```sql
CREATE TABLE t_test (
    id INT,
    name VARCHAR(100),
    PRIMARY KEY (id)
) Engine=InnoDB CHARSET=utf8;
mysql> SELECT * FROM t_test;
+--------+--------+
| id   | name   |
+--------+--------+
|  1   | 月伴飞鱼 |
+--------+--------+
1 row in set (0.07 sec)
```

假设插入该记录的事务id为10，那么此刻该条记录的示意图如下所示： ![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a43bb9f098f546aeb1758a7c2988b52b~tplv-k3u1fbpfcp-zoom-1.image)

假设之后事务id分别为20和30对这条记录进行UPDATE操作

```sql
update t_test set name = '周星驰' where id = 1;
update t_test set name = '周星星' where id = 1;
```

每次对记录进行改动，都会记录一条undo日志，每条undo日志也都有一个`roll_pointer`属性（INSERT操作对应的undo日志没有该属性，因为该记录并没有更早的版本），可以将这些undo日志都连起来，串成一个链表，像下图一样：

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db6ed73e353443b082682dae3edfc930~tplv-k3u1fbpfcp-zoom-1.image)

对该记录每次更新后，都会将旧值放到一条undo日志中，就算是该记录的一个旧版本，随着更新次数的增多，所有的版本都会被`roll_pointer`属性连接成一个链表，我们把这个链表称之为版本链，版本链的头节点就是当前记录最新的值

## ReadView

对于使用READ COMMITTED和REPEATABLE READ隔离级别的事务来说，都必须保证读到已经提交了的事务修改过的记录，也就是说假如另一个事务已经修改了记录但是尚未提交，是不能直接读取最新版本的记录的

**「核心问题就是：需要判断一下版本链中的哪个版本是当前事务可见的」**

因此，为了解决READ COMMITED和REPEATABLE READ级别下读取数据的问题，INNODB的设计者提出了READVIEW的概念，READVIEW中包含以下几个参数：

- m_ids：表示在生成READVIEW时当前系统中活跃的读写事务的事务id列表，活跃的是指当前系统中那些尚未提交的事务；
- min_trx_id：表示在生成READVIEW时当前系统中活跃的读写事务中最小的事务id，也就是m_ids中的最小值；
- max_trx_id：表示生成READVIEW时系统中应该分配给下一个事务的事务id值，由于事务id一般是递增分配的，所以max_trx_id就是m_ids中最大的那个id再加上1；
- creator_trx_id：表示生成该READVIEW的事务id，由于只有在对表中记录做改动（增删改）时才会为事务分配事务id，所以在一个读取数据的事务中的事务id默认为0；

有了这个READVIEW，就可以在访问某条记录时，按照如下的规则进行判断就可以确定版本链中哪个版本对当前读事务是否可见：

1. 版本的`trx_id==READVIEW中的creator_trx_id`，表示当前读事务正在读取被自己修改过的记录，该版本可以被当前事务访问；
2. 版本`trx_id < min_trx_id`，表明生成该版本的事务在当前事务生成READVIEW前已经提交了，所以该版本可以被当前事务访问；
3. 版本的`trx_id > max_trx_id`，表明生成该版本的事务在当前事务生成READVIEW后才开启的，该版本不可被当前事务访问；
4. 版本的`trx_id在READVIEW的min_trx_id和max_trx_id之间`，那就需要判断一下trx_id属性值是不是在m_ids中。如果在这个范围内，说明创建READVIEW时该事务还处于活跃状态，该版本不可以被当前事务访问；如果不在，说明创建READVIEW时生成该版本的事务已经被提交，该版本可以被当前事务访问；

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e8ebd4307944ca685f6f60486eb39db~tplv-k3u1fbpfcp-zoom-1.image)

如果某个版本的数据对当前事务不可见的话，那么就顺着版本链找到下一个版本的数据，继续按照上面的规则继续进行判断，以此类推，若是到了最后一个版本，该版本的数据仍对当前事务不可见，那么就表明该条记录对该事务完全不可见，查询结果就不会包含该条记录。

**「下面说一下，READ COMMITED和REPEATABLE READ在生成READVIEW时的区别：」**

RC在每一次 SELECT 语句前都会生成一个 ReadView，事务期间会更新，因此在其他事务提交前后所得到的 m_ids 列表可能发生变化，使得先前不可见的版本后续又突然可见了。

而 RR 只在事务的第一个 SELECT 语句时生成一个 ReadView，事务操作期间不更新

# MVCC能否完全解决幻读

假设有如下场景：

```sql
# 事务T1，REPEATABLE READ隔离级别下
mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM t_test WHERE id = 2;
Empty set (0.01 sec)

# 此时事务T2执行了：INSERT INTO t_test VALUES(2, '呵呵'); 并提交

mysql> UPDATE t_test SET name = '哈哈' WHERE id = 2;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> SELECT * FROM t_test WHERE id = 2;
+--------+---------+
| id  |  name    |
+--------+---------+
|     2 |   哈哈   |
+--------+---------
1 row in set (0.01 sec)
```

在REPEATABLE READ隔离级别下，T1第一次执行普通的SELECT语句时生成了一个ReadView，之后T2向表中新插入了一条记录便提交了，ReadView并不能阻止T1执行UPDATE或者DELETE语句来对改动这个新插入的记录（因为T2已经提交，改动该记录并不会造成阻塞），但是这样一来这条新记录的`trx_id`隐藏列就变成了T1的事务id

之后T1中再使用普通的SELECT语句去查询这条记录时就可以看到这条记录了，也就把这条记录返回给客户端了。因为这个特殊现象的存在，可以认为InnoDB中的MVCC并不能完完全全的禁止幻读

# 总结

MVCC就是在使用READ COMMITTD、REPEATABLE READ这两种隔离级别的事务在执行普通的SELECT操作时访问记录的版本链的过程，这样可以使不同事务的读-写、写-读操作并发执行，从而提升系统性能；

参考：

https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html

[MySQL是怎样运行的:从根儿上理解 MySQL](https://juejin.im/book/6844733769996304392)

公众号，欢迎关注！！！！！！

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/636823cf57af4a49bc5836178b22cc8d~tplv-k3u1fbpfcp-zoom-1.image)