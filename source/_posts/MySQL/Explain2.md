**原文：** https://mp.weixin.qq.com/s/twTghH8wTA_0uZghOdawkw

先看看具体有哪些字段：

```
mysql> EXPLAIN SELECT 1;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGufvGQPORWl2OQicEg4wvpIvWvdhjRSCaDiaEZtB4zlIscsxkcdg0G9FAw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

其实除了以SELECT开头的查询语句，其余的DELETE、INSERT、REPLACE以及UPDATE语句前边都可以加上EXPLAIN这个词儿，用来查看这些语句的执行计划

建两张测试表：

```
CREATE TABLE t1 (
    id INT NOT NULL AUTO_INCREMENT,
    key1 VARCHAR(100),
    key2 VARCHAR(100),
    key3 VARCHAR(100),
    name VARCHAR(100),
    PRIMARY KEY (id),
    KEY idx_key1 (key1),
    KEY idx_key2_key3(key2, key3)
) Engine=InnoDB CHARSET=utf8;

CREATE TABLE t2 (
    id INT NOT NULL AUTO_INCREMENT,
    key1 VARCHAR(100),
    key2 VARCHAR(100),
    key3 VARCHAR(100),
    name VARCHAR(100),
    PRIMARY KEY (id),
    KEY idx_key1 (key1),
    KEY idx_key2_key3(key2, key3)
) Engine=InnoDB CHARSET=utf8;
```

# 两个变种

## explain extended

会在 explain 的基础上额外提供一些查询优化的信息。紧随其后通过 show warnings 命令可以 得到优化后的查询语句，从而看出优化器优化了什么

```
explain extended SELECT * FROM t1 where key1 = '11';
show warnings;
```

## explain partitions

相比 explain 多了个 partitions 字段，如果查询是基于分区表的话，会显示查询将访问的分区。

```
EXPLAIN PARTITIONS SELECT * FROM t1 INNER JOIN t2 ON t1.key3 = t2.key3;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGu6k29vCoKGHxuD2uB2JbtC9srTFcErebx4Ykic6nOPrm4fmQXQ4Ct4bw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

# table列

这一列表示 explain 的一行正在访问哪个表

```
mysql> EXPLAIN SELECT * FROM t1;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGux5SArU1P4o5H6foBnmoJctpLuzT3PhtMNoQqKjfO5yDUzZicqSk7fNw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这个查询语句只涉及对t1表的单表查询，所以EXPLAIN输出中只有一条记录，其中的table列的值是t1，表明这条记录是用来说明对t1表的单表访问。

```
mysql> EXPLAIN SELECT * FROM t1 INNER JOIN t2;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGu8ToDRut3RZaG2ib5Bqx56QfD4AFGgU0m3pUYAkOkEMkoe69Lh4baVpw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到这个连接查询的执行计划中有两条记录，这两条记录的table列分别是t1和t2，这两条记录用来分别说明对t1表和t2表的访问

注意：

> 当 from 子句中有子查询时，table列是 `<derivenN>` 格式，表示当前查询依赖 id=N 的查询，于是先执行 id=N 的查询。当有 union 时，UNION RESULT 的 table 列的值为<union1,2>，1和2表示参与 union 的 select 行id。

# id列

id列的编号是 select 的序列号，有几个 select 就有几个id，并且id的顺序是按 select 出现的顺序增长的。

id列越大执行优先级越高，id相同则从上往下执行，id为NULL最后执行

比如下边这个查询中只有一个SELECT关键字，所以EXPLAIN的结果中也就只有一条id列为1的记录：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 = 'e038f672a8';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuIY5INxbmjjdL9CvasEaKVIytn872CW2aEdCGIoXCXBgdCFBn9icibz3g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

对于连接查询来说，一个SELECT关键字后边的FROM子句中可以跟随多个表，所以在连接查询的执行计划中，每个表都会对应一条记录，但是这些记录的id值都是相同的，比如：

```
mysql> EXPLAIN SELECT * FROM t1 INNER JOIN t2;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuSll6ftefLJicOm9icfO6WAuhvdnruC6DBMN5DGgr2NZbGX9O3v1rZ7Mw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到，上述连接查询中参与连接的t1和t2表分别对应一条记录，但是这两条记录对应的id值都是1。

注意：

> 在连接查询的执行计划中，每个表都会对应一条记录，这些记录的id列的值是相同的，出现在前边的表表示驱动表，出现在后边的表表示被驱动表。所以从上边的EXPLAIN输出中我们可以看出，查询优化器准备让t2表作为驱动表，让t1表作为被驱动表来执行查询

对于包含子查询的查询语句来说，就可能涉及多个SELECT关键字，所以在包含子查询的查询语句的执行计划中，每个SELECT关键字都会对应一个唯一的id值，比如这样：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 IN (SELECT key1 FROM t2) OR key3 = 'a1b6cee57a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGut9Mk3d3zgBktlKa3ibSBwVlz7XnJubVIjf67icGsrI21wlXFNURxINzw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从输出结果中我们可以看到，t1表在外层查询中，外层查询有一个独立的SELECT关键字，所以第一条记录的id值就是1，t2表在子查询中，子查询有一个独立的SELECT关键字，所以第二条记录的id值就是2。

但是这里大家需要特别注意，查询优化器可能对涉及子查询的查询语句进行重写，从而转换为连接查询。所以如果我们想知道查询优化器对某个包含子查询的语句是否进行了重写，直接查看执行计划就好了，比如说：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 IN (SELECT key3 FROM t2 WHERE t1.key1 = 'a1b6cee57a');
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuBS0stb5IrclZFibUak3kmNtu6aHiafo6VJASJlwtFrGdSHfJwKrpMEibw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到，虽然我们的查询语句是一个子查询，但是执行计划中t1和t2表对应的记录的id值全部是1，这就表明了查询优化器将子查询转换为了连接查询。

对于包含UNION子句的查询语句来说，每个SELECT关键字对应一个id值也是没错的，不过还是有点儿特别的东西，比方说下边这个查询：

```
mysql> EXPLAIN SELECT * FROM t1 UNION SELECT * FROM t2;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGu3Ke1HFpJjAiaMHAfdFVic0pib7Ugexjo2a5RnqPGgHaIgy2zEgtwIqEFQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

UNION子句是为了把id为1的查询和id为2的查询的结果集合并起来并去重，所以在内部创建了一个名为`<union1, 2>`的临时表（就是执行计划第三条记录的table列的名称），id为NULL表明这个临时表是为了合并两个查询的结果集而创建的。

跟UNION对比起来，UNION ALL就不需要为最终的结果集进行去重，它只是单纯的把多个查询的结果集中的记录合并成一个并返回给用户，所以也就不需要使用临时表。所以在包含UNION ALL子句的查询的执行计划中，就没有那个id为NULL的记录，如下所示：

```
mysql> EXPLAIN SELECT * FROM t1 UNION ALL SELECT * FROM t2;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuI1GMsudcXbQVHQPOXcDGMRRpKeASKa7Bw3ZbVicgcKFtvKcZQ2PIUcw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

# select_type列

MySQL每一个SELECT关键字代表的小查询都定义了一个称之为`select_type`的属性，意思是我们只要知道了某个小查询的`select_type`属性，就知道了这个小查询在整个大查询中扮演了一个什么角色

下面是官方文档介绍：

https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_select_type

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuhI93VKzX4Kxj2H1CgZ9bfiaVh24MhkGBeQE9S8heQU05lnes0WQ7nzA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**SIMPLE**

查询语句中不包含UNION或者子查询的查询都算作是SIMPLE类型，比方说下边这个单表查询的`select_type`的值就是SIMPLE：

```
mysql> EXPLAIN SELECT * FROM t1;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuyOcibZTA1559eeXicmqmCoYWHUppiaZRicb8gVFO9pXUhuIVbyAKb2MlTQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**PRIMARY**

对于包含UNION、UNION ALL或者子查询的大查询来说，它是由几个小查询组成的，其中最左边的那个查询的`select_type`值就是PRIMARY，比方说：

```
mysql> EXPLAIN SELECT * FROM t1 UNION SELECT * FROM t2;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGubSEKDiaUglibFic3lSDLNxtRtsRgGjm66zTYRZe0uR9mG9dyjCibed2hJg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从结果中可以看到，最左边的小查询`SELECT * FROM t1`对应的是执行计划中的第一条记录，它的`select_type`值就是PRIMARY。

**UNION**

对于包含UNION或者UNION ALL的大查询来说，它是由几个小查询组成的，其中除了最左边的那个小查询以外，其余的小查询的`select_type`值就是UNION，可以对比上一个例子的效果

**UNION RESULT**

MySQL选择使用临时表来完成UNION查询的去重工作，针对该临时表的查询的`select_type`就是UNION RESULT，同样对比上面的例子

**SUBQUERY**

如果包含子查询的查询语句不能够转为对应的semi-join的形式，并且该子查询是不相关子查询，并且查询优化器决定采用将该子查询物化的方案来执行该子查询时，该子查询的第一个SELECT关键字代表的那个查询的`select_type`就是SUBQUERY，比如下边这个查询：

概念解释：

> semi-join子查询，是指当一张表在另一张表找到匹配的记录之后，半连接（semi-jion）返回第一张表中的记录。与条件连接相反，即使在右节点中找到几条匹配的记录，左节点 的表也只会返回一条记录。另外，右节点的表一条记录也不会返回。半连接通常使用IN 或 EXISTS 作为连接条件

> 物化：这个将子查询结果集中的记录保存到临时表的过程称之为物化（Materialize）。那个存储子查询结果集的临时表称之为物化表。正因为物化表中的记录都建立了索引（基于内存的物化表有哈希索引，基于磁盘的有B+树索引），通过索引执行IN语句判断某个操作数在不在子查询结果集中变得非常快，从而提升了子查询语句的性能。

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 IN (SELECT key1 FROM t2) OR key3 = 'a1b6cee57a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuPxm8PlLG69wqzro7vPzPOa2om7snYnGN3XzzYCcpTw2LIfIkvLkicPg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到，外层查询的`select_type`就是PRIMARY，子查询的`select_type`就是SUBQUERY。

**DEPENDENT SUBQUERY**

如果包含子查询的查询语句不能够转为对应的semi-join的形式，并且该子查询是相关子查询，则该子查询的第一个SELECT关键字代表的那个查询的`select_type`就是DEPENDENT SUBQUERY，比如下边这个查询：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 IN (SELECT key1 FROM t2 WHERE t1.key2 = t2.key2) OR key3 = 'a1b6cee57a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGukHrSlrDITZy0RNxBoBnpCgfYiaVQvu1MXzY7q7ZORrNLMv8z1zzVuIA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**DEPENDENT UNION**

在包含UNION或者UNION ALL的大查询中，如果各个小查询都依赖于外层查询的话，那除了最左边的那个小查询之外，其余的小查询的`select_type`的值就是DEPENDENT UNION。比方说下边这个查询：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 IN (SELECT key1 FROM t2 WHERE key1 = 'a1b6cee57a' UNION SELECT key1 FROM t1 WHERE key1 = 'a1b6cee57a');
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuhpSRmCNcoxOyvd36IA77IiaHDiaFL5vxw1fkuFb33qJ4ee7xrYkjL4Yg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这个查询比较复杂啊，大查询里包含了一个子查询，子查询里又是由UNION连起来的两个小查询。从执行计划中可以看出来，`SELECT key1 FROM t2 WHERE key1 = 'a1b6cee57a'`这个小查询由于是子查询中第一个查询，所以它的`select_type`是DEPENDENT SUBQUERY，而`SELECT key1 FROM t1 WHERE key1 = 'a1b6cee57a'`这个查询的`select_type`就是DEPENDENT UNION。

**DERIVED**

对于采用物化的方式执行的包含派生表的查询，该派生表对应的子查询的`select_type`就是DERIVED，比方说下边这个查询：

```
mysql> EXPLAIN SELECT * FROM (SELECT key1, count(*) as t FROM t1 GROUP BY key1) AS derived_t1 where t > 1;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGu7icK9CBPDx7ehel6zEmSICK3YlQRAHxggdkllKu5KaovV78nj1kP7icw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从执行计划中可以看出，id为2的记录就代表子查询的执行方式，它的`select_type`是DERIVED，说明该子查询是以物化的方式执行的。id为1的记录代表外层查询，大家注意看它的table列显示的是`<derived2>`，表示该查询是针对将派生表物化之后的表进行查询的。

**MATERIALIZED**

当查询优化器在执行包含子查询的语句时，选择将子查询物化之后与外层查询进行连接查询时，该子查询对应的`select_type`属性就是MATERIALIZED，比如下边这个查询：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 IN (SELECT key1 FROM t2);
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGu1t0aMpekMPaIxxQfYVAGBhEvxaJ6Pd4qS01RibTgePgTw7aAgmTPYeQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

执行计划的第三条记录的id值为2，说明该条记录对应的是一个单表查询，从它的`select_type`值为MATERIALIZED可以看出，查询优化器是要把子查询先转换成物化表。然后看执行计划的前两条记录的id值都为1，说明这两条记录对应的表进行连接查询，需要注意的是第二条记录的table列的值是`<subquery2>`，说明该表其实就是id为2对应的子查询执行之后产生的物化表，然后将s1和该物化表进行连接查询。

# type列

这一列表示关联类型或访问类型，即MySQL决定如何查找表中的行，查找数据行记录的大概范围。依次从最优到最差分别为：`system > const > eq_ref > ref > range > index > ALL`一般来说，得保证查询达到range级别，最好达到ref

**NULL**

mysql能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引。例如：在索引列中选取最小值，可以单独查找索引来完成，不需要在执行时访问表

```
mysql> explain select min(id) from t1;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGu4ia4ECXcJVefOtGytiaDlDAgic9vT1XQHNVKJcyfPgXOCzVgG14MpWreg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**eq_ref**

primary key 或 unique key 索引的所有部分被连接使用 ，最多只会返回一条符合条件的记录。这可能是在 const 之外最好的联接类型了，简单的 select 查询不会出现这种 type。

在连接查询时，如果被驱动表是通过主键或者唯一二级索引列等值匹配的方式进行访问的（如果该主键或者唯一二级索引是联合索引的话，所有的索引列都必须进行等值比较），则对该被驱动表的访问方法就是`eq_ref`，比方说：

```
mysql> EXPLAIN SELECT * FROM t1 INNER JOIN t2 ON t1.id = t2.id;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGud6JhJrBSPUTicWauZOlCESRDbN3DMuFQoybW51NoR4aXZRd1iaJjYzLA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从执行计划的结果中可以看出，MySQL打算将t2作为驱动表，t1作为被驱动表，重点关注t1的访问方法是`eq_ref`，表明在访问t1表的时候可以通过主键的等值匹配来进行访问。

**ref**

当通过普通的二级索引列与常量进行等值匹配时来查询某个表，那么对该表的访问方法就可能是ref

相比 `eq_ref`，不使用唯一索引，而是使用普通索引或者唯一性索引的部分前缀，索引要和某个值相比较，可能会找到多个符合条件的行。

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 = 'a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGu5x2s9sfXK2WdBLLqrWcNN1Oh8mo97P5Ccl7sgw3J6D1UicjyoQzOC5g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到type列的值是ref，表明MySQL即将使用ref访问方法来执行对t1表的查询

**system，const**

mysql能对查询的某部分进行优化并将其转化成一个常量（可以看show warnings 的结果）。用于 primary key 或 unique key 的所有列与常数比较时，所以表最多有一个匹配行，读取1次，速度比较快。system是const的特例，表里只有一条元组匹配时为system

```
mysql> EXPLAIN SELECT * FROM t1 WHERE id = 5;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGudEPAadyNNGibPCLQts42oSJQ1UKTXlGRH6hMGYicWHyLyXZofkq0wEsw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**ref_or_null**

当对普通二级索引进行等值匹配查询，该索引列的值也可以是NULL值时，那么对该表的访问方法就可能是`ref_or_null`，比如说：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 = 'a' OR key1 IS NULL;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuq8ibfhSyYL4u9bia3BhnDCeZiaDK9WibmYqh8b1biaHwnWbibOXrApos11RQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**index_merge**

一般情况下对于某个表的查询只能使用到一个索引，但在某些场景下可以使用多种索引合并的方式来执行查询，我们看一下执行计划中是怎么体现MySQL使用索引合并的方式来对某个表执行查询的：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 = 'a' OR key2 = 'a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuzU6ecQ66lPQR4wxNn2nuxtSpVfoYy4JrIYI2ooCJgwUX0ZLeSiaQTNA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从执行计划的type列的值是`index_merge`就可以看出，MySQL打算使用索引合并的方式来执行对t1表的查询。

**unique_subquery**

类似于两表连接中被驱动表的`eq_ref`访问方法，`unique_subquery`是针对在一些包含IN子查询的查询语句中，如果查询优化器决定将IN子查询转换为EXISTS子查询，而且子查询可以使用到主键进行等值匹配的话，那么该子查询执行计划的type列的值就是`unique_subquery`，比如下边的这个查询语句：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key2 IN (SELECT id FROM t2 where t1.key1 = t2.key1) OR key3 = 'a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGu5QlfgqrKmia2uz29aH8qjEuj4mwMCRNuO8bOCm6b1f6AMcK2S1ElTdw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到执行计划的第二条记录的type值就是`unique_subquery`，说明在执行子查询时会使用到id列的索引。

**range**

范围扫描通常出现在 in(), between ,> ,<, >= 等操作中。使用一个索引来检索给定范围的行。

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 IN ('a', 'b', 'c');
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGupD4WgYFPCXAvCOCgxzjqMnRzO6j42TPqU0X2yxGvJ7PZ9oImyN248A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**index**

当我们可以使用索引覆盖，但需要扫描全部的索引记录时，该表的访问方法就是index

扫描全表索引，这通常比ALL快一些。（index是从索引中读取的，而all是从硬盘中读取）

**ALL**

最熟悉的全表扫描

```
mysql> explain select * from t2;
```

一般来说，这些访问方法按照我们介绍它们的顺序性能依次变差。其中除了All这个访问方法外，其余的访问方法都能用到索引，除了`index_merge`访问方法外，其余的访问方法都最多只能用到一个索引。

# possible_keys和key列

`possible_keys`列显示查询可能使用哪些索引来查找。

explain 时可能出现 `possible_keys` 有列，而 key 显示 NULL 的情况，这种情况是因为表中数据不多，mysql认为索引对此查询帮助不大，选择了全表查询。

如果`possible_keys`列是NULL，则没有相关的索引。在这种情况下，可以通过检查 where 子句看是否可以创造一个适当的索引来提高查询性能，然后用 explain 查看效果。

key列显示mysql实际采用哪个索引来优化对该表的访问。如果没有使用索引，则该列是 NULL。如果想强制mysql使用或忽视`possible_keys`列中的索引，在查询中使用 force index、ignore index。

比方说下边这个查询：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 > 'z' AND key2 = 'a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGunVicRrtbha98wPhlmu4LCol760t9rhFkicjBW5VPu3UiaoYXsnABt6YFA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

上述执行计划的`possible_keys`列的值是`idx_key1,idx_key2_key3`，表示该查询可能使用到`idx_key1,idx_key2_key3`两个索引，然后key列的值是`idx_key3`，表示经过查询优化器计算使用不同索引的成本后，最后决定使用`idx_key3`来执行查询比较划算。

需要注意的一点是，`possible_keys`列中的值并不是越多越好，可能使用的索引越多，查询优化器计算查询成本时就得花费更长时间，所以如果可以的话，尽量删除那些用不到的索引。

# key_len列

这一列显示了mysql在索引里使用的字节数，通过这个值可以算出具体使用了索引中的哪些列

对于使用固定长度类型的索引列来说，它实际占用的存储空间的最大长度就是该固定值，对于指定字符集的变长类型的索引列来说，比如某个索引列的类型是VARCHAR(100)，使用的字符集是utf8，那么该列实际占用的最大存储空间就是100 × 3 = 300个字节。

如果该索引列可以存储NULL值，则`key_len`比不可以存储NULL值时多1个字节。

对于变长字段来说，都会有2个字节的空间来存储该变长列的实际长度。

当字符串过长时，mysql会做一个类似左前缀索引的处理，将前半部分的字符提取出来做索引。

> key_len计算规则如下：字符串 char(n)：n字节长度 varchar(n)：2字节存储字符串长度，如果是utf-8，则长度 3n + 2 数值类型 tinyint：1字节 smallint：2字节 int：4字节 bigint：8字节　　 时间类型　 date：3字节 timestamp：4字节 datetime：8字节

比如下边这个查询：

```
mysql> EXPLAIN SELECT * FROM s1 WHERE id = 5;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGu1QYXAkpfgyOjze1EwYsP9xfM3DQcl76kf2kqdSHcyiblKdx2AIAfU2g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

由于id列的类型是INT，并且不可以存储NULL值，所以在使用该列的索引时`key_len`大小就是4。

对于可变长度的索引列来说，比如下边这个查询：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 = 'a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuLLbggpDsWMu2n6pNKAtUdEZg54IYTjX8cgthEq7mXzuz28md9htKuw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

由于key1列的类型是`VARCHAR(100)`，所以该列实际最多占用的存储空间就是300字节，又因为该列允许存储NULL值，所以`key_len`需要加1，又因为该列是可变长度列，所以`key_len`需要加2，所以最后`ken_len`的值就是303。

# rows列

这一列是mysql估计要读取并检测的行数，注意这个不是结果集里的行数。

如果查询优化器决定使用全表扫描的方式对某个表执行查询时，执行计划的rows列就代表预计需要扫描的行数，如果使用索引来执行查询时，执行计划的rows列就代表预计扫描的索引记录行数。比如下边这个查询：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 > 'a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGucLSkrS43Kq3iamldnkmbwMeQC7xTjVicNZLTUicOtHwHj5897NuKmzgiaQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们看到执行计划的rows列的值是113，这意味着查询优化器在经过分析使用`idx_key1`进行查询的成本之后，觉得满足`key1 > 'a'`这个条件的记录只有113条。

# ref列

这一列显示了在key列记录的索引中，表查找值所用到的列或常量，常见的有：const（常量），字段名（例：`t1.id`）

ref列展示的就是与索引列作等值匹配的值什么，比如只是一个常数或者是某个列。大家看下边这个查询：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key1 = 'a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuXUGqA8NsicpuyCooyweTpicldKwW2VwmMcqj8MAVeRbWJoN2XLc4rMgw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到ref列的值是const，表明在使用`idx_key1`索引执行查询时，与key1列作等值匹配的对象是一个常数，当然有时候更复杂一点：

```
mysql> EXPLAIN SELECT * FROM t1 INNER JOIN t2 ON t1.id = t2.id;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuMo4bWgUXXh5icsib7DzQcBicicvYWcO7j8wNZgNpFeqtFH5gWVA6ibroWgg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到对被驱动表t1的访问方法是`eq_ref`，而对应的ref列的值是`canal_manager.t2.id`，这说明在对被驱动表进行访问时会用到PRIMARY索引，也就是聚簇索引与一个列进行等值匹配的条件，于t2表的id作等值匹配的对象就是`canal_manager.t2.id`列（注意这里把数据库名也写出来了）。

有的时候与索引列进行等值匹配的对象是一个函数，比方说下边这个查询

```
mysql> EXPLAIN SELECT * FROM t1 INNER JOIN t2 ON t2.key1 = UPPER(t1.key1);
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuQKyZ78XJy4IV6up1PH6hsAnpQ26IicQQiasnMR3Lqy8dDFuIsewGKdJw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们看执行计划的第二条记录，可以看到对t2表采用ref访问方法执行查询，然后在查询计划的ref列里输出的是func，说明与t2表的key1列进行等值匹配的对象是一个函数。

# Extra列

顾名思义，Extra列是用来说明一些额外信息的，我们可以通过这些额外信息来更准确的理解MySQL到底将如何执行给定的查询语句。

**Using index**

查询的列被索引覆盖，并且where筛选条件是索引的前导列，是性能高的表现。一般是使用了覆盖索引(索引包含了所有查询的字段)。对于innodb来说，如果是辅助索引性能会有不少提高

```
mysql> EXPLAIN SELECT key1 FROM t1 WHERE key1 = 'a';
```

**Using where**

当我们使用全表扫描来执行对某个表的查询，并且该语句的WHERE子句中有针对该表的搜索条件时，在Extra列中会提示上述额外信息。比如下边这个查询

```
mysql> EXPLAIN SELECT * FROM t1 WHERE name= 'a1b6cee57a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGus8LD6kgPucPxsN4JzwUb3ZXia3eguoFNQCFwPwPp3Igx9P5Y83cslnA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**Using where Using index**

查询的列被索引覆盖，并且where筛选条件是索引列之一但是不是索引的前导列，意味着无法直接通过索引查找来查询到符合条件的数据

```
mysql> EXPLAIN SELECT id FROM t1 WHERE key3= 'a1b6cee57a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGusiaMZoKsClq2l0q7ObaMdN4bvWJakicG7xFQrHX0Gcich81wX2haLrvKg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**NULL**

查询的列未被索引覆盖，并且where筛选条件是索引的前导列，意味着用到了索引，但是部分字段未被索引覆盖，必须通过“回表”来实现，不是纯粹地用到了索引，也不是完全没用到索引

```
mysql> EXPLAIN SELECT * FROM t1 WHERE key2= 'a1b6cee57a';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuEibEsnSttLrLOclIqYxNBqQD2FlDvxpl4XXibIvDcGNVm0HxU80nNJZA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**Using index condition**

与Using where类似，查询的列不完全被索引覆盖，where条件中是一个前导列的范围；

```
mysql>  EXPLAIN SELECT * FROM t1 WHERE key1 like '1';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGu6Uq9iburLoaTCObfop9TzUBUFHiaogmnBnap9aHbwia6QHb1dUKRIeiaog/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**Using temporary**

在许多查询的执行过程中，MySQL可能会借助临时表来完成一些功能，比如去重、排序之类的，比如我们在执行许多包含DISTINCT、GROUP BY、UNION等子句的查询过程中，如果不能有效利用索引来完成查询，MySQL很有可能寻求通过建立内部的临时表来执行查询。如果查询中使用到了内部的临时表，在执行计划的Extra列将会显示Using temporary提示，比方说这样：

name没有索引，此时创建了张临时表来distinct

```
mysql> explain select distinct name from t1;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuzstxAia6gc7bic5XfibP5TFFUicZ98Da09PfWtxFXrQbQBZzbtp0SGJ03A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

key1建立了idx_key1索引，此时查询时extra是using index,没有用临时表

```
mysql> explain select distinct key1 from t1;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuFJNRBibM7TXwXwj5UYBJhPWrYAPbMF5IVQmd4nBu6QzMYJCyby6Umvw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**Using filesort**

mysql 会对结果使用一个外部索引排序，而不是按索引次序从表里读取行。此时mysql会根据联接类型浏览所有符合条件的记录，并保存排序关键字和行指针，然后排序关键字并按顺序检索行信息。这种情况下一般也是要考虑使用索引来优化的。

name未创建索引，会浏览t1整个表，保存排序关键字name和对应的id，然后排序name并检索行记录

```
mysql> explain select * from t1 order by name;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGukjGWcRrYbD0kHEZrNhG2uueTNjNnptWPgz7MfcG0DhT2PmgFXZ12pg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

key1建立了idx_key1索引,此时查询时extra是using index

```
mysql> explain select * from t1 order by key1;
```

**Using join buffer (Block Nested Loop)**

在连接查询执行过程中，当被驱动表不能有效的利用索引加快访问速度，MySQL一般会为其分配一块名叫join buffer的内存块来加快查询速度，也就是我们所讲的基于块的嵌套循环算法，比如下边这个查询语句：

```
mysql> EXPLAIN SELECT * FROM t1 INNER JOIN t2 ON t1.key3 = t2.key3;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuxLG5lnU1IVJvQMkVtu37O6s2UmGyqE0nCRB8LKGptLfsV9hoE5yictQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**No tables used**

当查询语句的没有FROM子句时将会提示该额外信息，比如：

```
mysql> EXPLAIN SELECT 1;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGuUXjx9wQiblohBiaKjJNZlPicEgNOepILnj3ibY6rk0IiaibdIN8JlyNwLtUQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**Impossible WHERE**

查询语句的WHERE子句永远为FALSE时将会提示该额外信息，比方说：

```
mysql> EXPLAIN SELECT * FROM t1 WHERE 1 != 1;
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/hC3oNAJqSRyQMibf1k8Mq6GR7EboNNBGu4r67xqkQUCyicjow1blF2WEOHI4mx71cpWtcZAib8jHLJiaNF50vuaiasQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

参考：

https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain-extra-information