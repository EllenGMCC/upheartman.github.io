---
title: Join语句
categories: 
- MySQL
---

文章首发在公众号！！

# 前言

最近在读《MySQL性能调优与架构设计》，看到一个关于join的优化原则，如下：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea916952590e4f82917a64c0d6724c0c~tplv-k3u1fbpfcp-zoom-1.image)

**大白话解释下：**

因为驱动结果集越大，意味着需要循环的次数越多，也就是说在被驱动结果集上面所 需要执行的查询检索次数会越多。

比如，当两个表（表 A 和 表 B） Join 的时候，如果表 A 通过 WHERE 条件过滤后有 10 条记录，而表 B 有 20 条记录。如果我们选择表 A 作为驱动表，也就是被驱动表的结果集为 20，那么我们通过 Join 条件对被驱动表（表 B）的比较过滤就会有 10 次。反之，如果我们选择表 B 作为驱动表，则需要有 20 次对表 A 的比较过滤。

> “
>
> 小贴士1：驱动表的定义：当进行多表连接查询时，1.指定了联接条件时，满足查询条件的记录行数少的表为驱动表，2.未指定联接条件时，行数少的表为驱动表
>
> ”

> “
>
> 小贴士2：关联查询的概念：MySQL 表关联的算法是 Nest Loop Join，是通过驱动表的结果集作为循环基础数据，然后一条一条地通过该结果集中的数据作为过滤条件到下一个表中查询数据，然后合并结果
>
> ”

所以本文就从这个地方开始，学习下mysql join的相关知识

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9349dc0af0cc428aae82757677789f7f~tplv-k3u1fbpfcp-zoom-1.image)

# 基本介绍

**left join、right join、inner join的区别**

相信大家都知道这个，简单介绍下

- left join(左连接)：返回包括左表中的所有记录和右表中联结字段相等的记录
- right join(右连接)：返回包括右表中的所有记录和左表中联结字段相等的记录
- inner join(等值连接)：只返回两个表中联结字段相等的行

一张大图， 清楚明了：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3c3bc06e1e94ceab7546cc553e24590~tplv-k3u1fbpfcp-zoom-1.image)

那我们看看在join连接时哪个表是驱动表，哪个表是被驱动表：

- 1.当使用left join时，左表是驱动表，右表是被驱动表
- 2.当使用right join时，右表是驱动表，左表是被驱动表
- 3.当使用inner join时，mysql会选择数据量比较小的表作为驱动表，大表作为被驱动表

注意：当连接查询有where条件时，带where条件的表是驱动表，否则是被驱动表

具体情况大家可以用Explain执行计划验证下

Explain使用可以参考我之前的文章：[最完整的Explain总结，SQL优化不再困难](https://mp.weixin.qq.com/s/twTghH8wTA_0uZghOdawkw)

**举个例子：**

假如有两张表：A是小表，B是大表

使用left join 时，则应该这样写

```
select * from A a left join B b on a.id=b.id;
```

此时A表时驱动表，B表是被驱动表

测试：假设A表140多条数据，B表20万左右的数据量

```
select * from A a left join B b on a.id=b.id;
```

执行时间：8s

```
select * from B b left join A a on a.id=b.id;
```

执行时间：19s

所以记住：小表驱动大表优于大表驱动小表

**一个注意点**

join查询在有索引条件下

- 驱动表有索引不会使用到索引
- 被驱动表建立索引会使用到索引

所以在以小表驱动大表的情况下，再给大表建立索引会大大提高执行速度

举例子测试一下：

假设有2张表：A表，B表，分别建立索引

```
select * from A a left join B b on a.name=b.name;
```

发现只有B表name使用到索引

如果同时只给A表的name建立索引会是什么情况？

在这种情况下，A表索引失效

所以可以通过给被驱动表建立索引来优化SQL

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35df83f43cbd4a2f89e02968c5073393~tplv-k3u1fbpfcp-zoom-1.image)

# Join原理

mysql的join算法叫做Nested-Loop Join（嵌套循环连接）

而这个Nested-Loop Join有三种变种，下面分别介绍下

## Simple Nested-Loop

这个算法相当简单、直接。即驱动表中的每一条记录与被驱动表中的记录进行比较判断（就是个笛卡尔积）。对于两表联接来说，驱动表只会被访问一遍，但被驱动表却要被访问到好多遍

假设R为驱动表，S被驱动表，用伪代码表示一下这个过程就是这样：

```
for r in R                      -- 扫描R表（驱动表）
    for s in S                    -- 扫描S表（被驱动表）
        if (r and s satisfy the join condition)  -- 如果r和s满足join条件
            output result    -- 返回结果集
```

所以如果R有1万条数据，S有1万条数据，那么数据比较的次数`1万 * 1万 =1亿次`，这种查询效率会非常慢。

## Index Nested-Loop

这个是基于索引进行连接的算法

它要求被驱动表上有索引，可以通过索引来加速查询。

假设R为驱动表，S被驱动表，用伪代码表示一下这个过程就是这样：

```
For r in R                  -- 扫描R表
    for s in Sindex                    -- 查询S表的索引（固定3~4次IO，B+树高度）
        if （s == r)                   -- 如果r匹配了索引s
            output result   -- 返回结果集
```

## Block Nested-Loop

这个算法较Simple Nested-Loop Join的改进就在于可以减少被驱动表的扫描次数

因为它使用Join Buffer来减少内部循环读取表的次数

假设R为驱动表，S被驱动表，用伪代码表示一下这个过程就是这样：

```
for r in R                             -- 扫描表R
    store p from R in Join Buffer    -- 将部分或者全部R的记录保存到Join Buffer中，记为p
    for s in S                        -- 扫描表S
        if (p and s satisfy the join condition)        -- p与s满足join条件
           output result                    -- 返回为结果集
```

可以看到相比Simple Nested-Loop Join算法，Block Nested-LoopJoin算法仅多了一个所谓的Join Buffer

**为什么这样就能减少被驱动表的扫描次数呢？**

下图相比更好地解释了Block Nested-Loop Join算法的运行过程

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78a35e345c14463795293d5f634f6adb~tplv-k3u1fbpfcp-zoom-1.image)

可以看到Join Buffer用以缓存联接需要的列（所以再次提醒我们，最好不要把*作为查询列表，只需要把我们关心的列放到查询列表就好了，这样还可以在join buffer中放置更多的记录呢，是不是这个道理哈，哈哈）

然后以Join Buffer批量的形式和被驱动表中的数据进行联接比较。

**关于Join Buffer**

1. Join Buffer会缓存所有参与查询的列而不是只有Join的列。
2. `join_buffer_size`的默认值是256K

## 总结

在选择Join算法时，会有优先级：

```
Index Nested-LoopJoin > Block Nested-Loop Join > Simple Nested-Loop Join
```

当不使用`Index Nested-Loop Join`的时候，默认使用`Block Nested-Loop Join`。

使用Block Nested-Loop Join算法需要开启优化器管理配置的`optimizer_switch`的设置`block_nested_loop`为on，默认为开启。

# Join优化

**通过上面的简单介绍，可以总结出以下几种优化思路**

1.用小结果集驱动大结果集，减少外层循环的数据量

2.如果小结果集和大结果集连接的列都是索引列，mysql在join时也会选择用小结果集驱动大结果集，因为索引查询的成本是比较固定的，这时候外层的循环越少，join的速度便越快。

3.为匹配的条件增加索引：争取使用Index Nested-Loop Join，减少内层表的循环次数

4.增大`join buffer size`的大小：当使用Block Nested-Loop Join时，一次缓存的数据越多，那么外层表循环的次数就越少，减少不必要的字段查询：

5.当用到Block Nested-Loop Join时，字段越少，join buffer 所缓存的数据就越多，外层表的循环次数就越少；

**觉得有收获，帮忙点赞，转发，分享下吧，谢谢**

# 最后

微信搜索：月伴飞鱼，交个朋友

公众号后台回复666，获得免费电子书籍，会持续更新

参考：

官网：https://dev.mysql.com/doc/refman/8.0/en/

书籍：MySQL性能调优与架构设计