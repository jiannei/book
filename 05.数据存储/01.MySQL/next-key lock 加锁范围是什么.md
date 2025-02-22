> 原文地址 [segmentfault.com](https://segmentfault.com/a/1190000040129107)

> 前言某天，突然被问到 MySQL 的 next-key lock，我瞬间的反应就是：这都是啥啥啥？？？这一个截图我啥也看不出来呀？仔细一看，好像似曾相识，这不是《MySQL ...

#### 前言

某天，突然被问到 MySQL 的 next-key lock，我瞬间的反应就是：

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221510630.png)

这都是啥啥啥？？？

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221512941.png)

这一个截图我啥也看不出来呀？

仔细一看，好像似曾相识，这不是《MySQL 45 讲》里面的内容么？

### 什么是 next-key lock

> A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.

官网的解释大概意思就是：[next-key](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-next-key-locks) 锁是索引记录上的记录锁和索引记录之前的间隙上的间隙锁的组合。

先给自己来一串小问号？？？

1.  在主键、唯一索引、普通索引以及普通字段上加锁，是锁住了哪些索引？
2.  不同的查询条件，分别锁住了哪些范围的数据？
3.  for share 和 for update 等值查询和范围查询的锁范围？
4.  当查询的等值不存在时，锁范围是什么？
5.  当查询条件分别是主键、唯一索引、普通索引时有什么区别？

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221514605.png)

既然啥都不懂，那只好从头开始操作实践一把了！

先看看看 《MySQL 45 讲》中丁奇老师的结论：

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221514524.png)

看了这结论，应该可以解答一大部分问题，不过有一句非常非常重点的话需要关注：`MySQL 后面的版本可能会改变加锁策略，所以这个规则只限于截止到现在的最新版本，即 5.x 系列<=5.7.24，8.0 系列 <=8.0.13`

所以，以上的规则，对现在的版本并不一定适用，下面我以 `MySQL 8.0.25` 版本为例，进行多角度验证 next-key lock 加锁范围。

### 环境准备

MySQL 版本：8.0.25

隔离级别：可重复读（RR）

存储引擎：InnoDB

```
mysql> select @@global.transaction_isolation,@@transaction_isolation\G
mysql> show create table t\G
```

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221515744.png)

> 如何使用 Docker 安装 MySQL，可以参考另一篇文章《使用 Docker 安装并连接 MySQL》

### 主键索引

首先来验证主键索引的 next-key lock 的范围

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221515868.png)

此时数据库的数据如图所示，对主键索引来说此时数据间隙如下：

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221515183.png)

#### 主键等值查询 —— 数据存在

```
mysql> begin; select * from t where id = 10 for update;
```

这条 SQL，对 `id = 10` 进行加锁，可以先思考一下加了什么锁？锁住了什么数据？

可以通过 `data_locks` 查看锁信息，SQL 如下：

```
# mysql> select * from performance_schema.data_locks;
mysql> select * from performance_schema.data_locks\G
```

具体字段含义可以参考 [官方文档](https://dev.mysql.com/doc/mysql-perfschema-excerpt/8.0/en/performance-schema-data-locks-table.html)

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221516764.png)

结果主要包含引擎、库、表等信息，咱们需要重点关注以下几个字段：

- INDEX_NAME：锁定索引的名称
- LOCK_TYPE：锁的类型，对于 InnoDB，允许的值为 RECORD 行级锁 和 TABLE 表级锁。
- LOCK_MODE：锁的类型：S, X, IS, IX, and gap locks
- LOCK_DATA：锁关联的数据，对于 InnoDB，当 LOCK_TYPE 是 RECORD（行锁），则显示值。当锁在主键索引上时，则值是锁定记录的主键值。当锁是在辅助索引上时，则显示辅助索引的值，并附加上主键值。

结果很明显，这里是对表添加了一个 IX 锁 并对主键索引 id = 10 的记录，添加了一个 `X,REC_NOT_GAP` 锁，表示只锁定了记录。

同样 `for share` 是对表添加了一个 IS 锁并对主键索引 id = 10 的记录，添加了一个 S 锁。

可以得出结论：

对主键等值加锁，且值存在时，会对表添加意向锁，同时会对主键索引添加行锁。

#### 主键等值查询 —— 数据不存在

```
mysql> select * from t where id = 11 for update;
```

如果是数据不存在的时候，会加什么锁呢？锁的范围又是什么？

在验证之前，分析一下数据的间隙。

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221516852.png)

1.  `id = 11` 是肯定不存在的。但是加了 `for update`，这时需要加 next-key lock，`id = 11` 所属区间为 (10,15] 的~前开后闭~区间；
2.  因为是`等值查询`，不需要锁 `id = 15` 那条记录，next-key lock 会退化为间隙锁；
3.  最终区间为 (10,15) 的前开后开区间。

使用 data_locks 分析一下锁信息：

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221517286.png)

看下锁的信息 `X,GAP` 表示加了间隙锁，其中 LOCK_DATA = 15，表示锁的是 主键索引 id = 15 之前的间隙。

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221517499.png)

此时在另一个 Session 执行 SQL，答案显而易见，是 id = 12 不可以插入，而 id = 15 是可以更新的。

可以得出结论，在数据不存在时，主键等值查询，会锁住该主键查询条件所在的间隙。

#### 主键范围查询（重点）

```
mysql> begin; select * from t where id >= 10 and id < 11 for update;
```

根据 《MySQL 45 讲》分析得出下面结果：

1.  `id >= 10` 定位到 10 所在的区间 (10,+∞)；
2.  因为是 >= 存在等值判断，所以需要包含 10 这个值，变为 [10,+∞) 前闭后闭区间；
3.  `id < 11` 限定后续范围，则根据 11 判断下一个区间为 15 的~前开后闭~区间；
4.  结合起来则是 [10,15]。（不完全正确）

先看下 data_locks

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221517018.png)

可以看到除了表锁之外，还有 id = 10 的行锁（`X,REC_NOT_GAP`）以及主键索引 id = 15 之前的间隙锁（`X,GAP`）。

所以实际上 id = 15 是可以进行更新的。也就是说`前开后闭区间`出现了问题，个人认为应该是 `id < 11` 这个条件判断，导致不需要进行了锁 15 这个行锁。

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221518655.png)

结果验证也是正确的，id = 12 插入阻塞，id = 15 更新成功。

当范围的右侧是包含等值查询呢？

```
mysql> begin; select * from t where id > 10 and id <= 15 for update;
```

来分析一下这个 SQL：

1.  `id > 10` 定位到 10 所在的区间 (10,+∞)；
2.  `id <= 15` 定位是 (-∞, 15]；
3.  结合起来则是 (10,15]。

同样先看一下 data_locks

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221518435.png)

可以看出只添加了一个主键索引 id = 15 的 X 锁。

验证下 id = 15 是否可以更新？再验证 id = 16 是否可以插入？

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221518574.png)

事实证明是没有问题的！

当然，这里有小伙伴会说，在 《MySQL 45 讲》 里面说这里有一个 bug，会锁住下一个 next-key。

!![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221519365.png)

事实证明，这个 bug 已经被修复了。修复版本为 `MySQL 8.0.18`。但是并没有完全修复！！！

> 参考链接地址：
>
> [https://dev.mysql.com/doc/rel...](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-18.html#mysqld-8-0-18-bug)
>
> 搜索关键字：Bug #29508068)

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221519747.png)

咱们可以分别用 8.0.17 进行复现一下：

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221520280.png)

在 8.0.17 中 `id <= 15` 会将 id = 20 这条数据也锁着，而在 8.0.25 版本中则不会。所以这个 bug 是被修复了的。

再来看下是`前开后闭`还是`前开后开`的问题，严谨一下，使用 8.0.17 和 8.0.18 做比较。

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221520629.png)

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221520713.png)

现在我估计大概率是在 8.0.18 版本修复 `Bug #29508068` 的时候，把这个`前开后闭`给优化成了`前开后开`了。

对比 data_locks 数据：

![](https://raw.githubusercontent.com/jiannei/images/main/images/202502221521056.png)

注意红色下划线部分，在 8.0.17 版本中 `id < 17` 时 LOCK_MODE 是 `X`，而在 8.0.25 版本中则是 `X,GAP`。

### 总结

本文主要通过实际操作，对主键加锁时的 next-key lock 范围进行了验证，并查阅资料，对比版本得出不同的结论。

#### 结论一：

1.  加锁时，会先给表添加意向锁，IX 或 IS；
2.  加锁是如果是多个范围，是分开加了多个锁，每个范围都有锁；（这个可以实践下 id < 20 的情况）
3.  主键等值查询，数据存在时，会对该主键索引的值加行锁 `X,REC_NOT_GAP`；
4.  主键等值查询，数据不存在时，会对查询条件主键值所在的间隙添加间隙锁 `X,GAP`；
5.  主键等值查询，范围查询时情况则比较复杂：

    1.  8.0.17 版本是前开后闭，而 8.0.18 版本及以后，进行了优化，主键时判断不等，不会锁住后闭的区间。
    2.  临界 `<=` 查询时，8.0.17 会锁住下一个 next-key 的前开后闭区间，而 8.0.18 及以后版本，修复了这个 bug。

> 优化后，导致后开，这个不知道是因为优化后，主键的区间会直接后开，还是因为是个 bug。具体小伙伴可以尝试一下。

#### 结论二

通过使用 `select * from performance_schema.data_locks;` 和操作实践，可以看出 LOCK_MODE 和 LOCK_DATE 的关系：

| LOCK_MODE     | LOCK_DATA | 锁范围                           |
| ------------- | --------- | -------------------------------- |
| X,REC_NOT_GAP | 15        | 15 那条数据的行锁                |
| X,GAP         | 15        | 15 那条数据之前的间隙，不包含 15 |
| X 15          | 15        | 那条数据的间隙，包含 15          |

1.  `LOCK_MODE = X` 是前开后闭区间；
2.  `X,GAP` 是前开后开区间（间隙锁）；
3.  `X,REC_NOT_GAP` 行锁。

基本已经摸清主键的 next-key lock 范围，注意版本使用的是 8.0.25。

#### 疑问

1.  那唯一索引的 next-key lock 范围是什么?
2.  当索引覆盖时锁的范围和加锁的索引分别是什么？
3.  我为什么说这个 bug 没有完全修复，也是在非主键唯一索引中复现了这个 bug​。

文章篇幅有限，小伙伴可以先自己思考一下，尽量自己操作试一试，实践出真知。至于具体答案，那就需要下一篇文章进行验证并总结结论了。
