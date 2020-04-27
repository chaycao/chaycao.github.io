---
layout:     reprint
title:      【转】关于 MySQL Repeatable Read Isolation 常见的三个误区
date:       2020-04-24
author:     ChayCao
header-img: img/post-bg-2015.jpg 
catalog: true
tags:  MySQL
---


转自[《對於 MySQL Repeatable Read Isolation 常見的三個誤解》](https://medium.com/@chester.yw.chu/%E5%B0%8D%E6%96%BC-mysql-repeatable-read-isolation-%E5%B8%B8%E8%A6%8B%E7%9A%84%E4%B8%89%E5%80%8B%E8%AA%A4%E8%A7%A3-7a9afbac65af)。原文是繁体，而由于部分专有名词在繁体和简体中说法不同，为了方便简体阅读，对部分专有名词进行转换，正文如下：

如果对 Isolation Levels 记忆模糊可以参考这一篇：[复习资料库的 Isolation Level 与图解五个常见的 Race Conditions](https://medium.com/@chester.yw.chu/複習資料庫的-isolation-level-與常見的五個-race-conditions-圖解-16e8d472a25c?postPublishedType=repub)

一开始我们先看一个例子。有两个 Transaction 同时操作 gamer 这个表格，其中一个 select 所有的数据，另一个则在中间新增了一个新的玩家，Frank，然后 commit。在 MySQL InnoDB Engine 的环境下，使用 Repeatable Read Isolation (RR Isolation) 时，数据库的行为如下图：
![幻读不会发生](https://i2.tiimg.com/717558/5026ea63f61bbf8f.png)

从上图可以看到，在 Transaction B 新增了一条数据之后，Transaction A 还是只读取到 5 条数据，没有玩家 Frank 的数据，Phantom 现象并没有发生。所以大家可能会问，MySQL InnoDB Engine 的 RR Isolation 是不是 Phantom Safe 的呢？网络上的确也有不少文章是这么认为的。
但是让我们继续这个例子。 Transaction A 的任务是在每周的最后一天为当下分数最高的前三名玩家增加 credit，前三名玩家的 credit 都各增加 1 分。依照上图可以知道现在前三名的玩家分别是Alice、Carol 跟Bob，三个玩家的分数都达到了740 分以上，所以可以很简单的使用Atomic Update (credit = credit + 1)，为所有分数达到740 分以上的玩家credit 加1。
虽然目前数据库实际上有 6 条玩家的数据，但是从 Transaction A 的视角只有看到总共 5 条玩家的数据。在这样的情况下，Transaction A 所做的更新是不是理论上只会影响到这 5 条数据呢？实际的实验结果如下图：

![幻读导致写倾斜](https://i2.tiimg.com/717558/bfe110a7f8497612.png)

从上图中可以看到，在 Transaction A 执行更新命令后，如果马上再重新读取一次 gamer 表格，玩家 Frank 的数据竟然意外的出现在列表中，发生了 Phantom 现象。不仅如此，照原本的逻辑 Transaction A 应该只会为前 3 名的玩家增加 credit，但是因为 Frank 的分数也同样高于 740 分，同样也被增加了 credit。最后被增加 credit 的玩家总共有 4 个 ，比原本系统预计送出的 credit 还多。这种现象属于 Write Skew，在这个例子中是因为 Phamtom 而导致的 Write Skew。

## 误区1：MySQL Repeatable Read Isolation 可以避免 Phantom

这就是常见的第一个误解。在做过第一个实验后，我们常常会误以为 MySQL 的 RR Isolation 是 Phantom Safe，但其实不是。 MySQL InnoDB Engine 跟 PostgreSQL 一样，它们 RR Isolation 的实现都是采用 Snapshot Islolation。如果了解 Snapshot Isolation 的机制，就可以知道 Snapshot Isolation 在 read-only Transaction 中才可以避免 Phantom，但是像在像上面的例子使用的 read-write Transaction 中，就有可能出现 Phantom，进而导致 Write Skew。

> *Snapshot isolation avoids phantoms in read-only queries, but in read-write transactions, phantoms can lead to particularly tricky cases of write skew.*

Snapshot Isolation 会在每个 Transaction 第一次 SELECT 数据的时候，记录下一个概念上像是时间标记的数据，每个 Transaction 在执行完第一次 SELECT 之后，Transaction 就只允许读取:

1. 在这个时间标记之前就已经 commit 的数据
2. Transaction 自己本身对数据库所做的更动

这就像对数据库做了一个 Snapshot 一样，Transaction 只能看到这个 Snapshot 的内容，但是无法读取到其他 Transaction 所做的更新。但是在 InnoDB 的实现中，这个规则只限于 SELECT (DQL) 命令，其他像是 INSERT、UPDATE 和 DELETE 等 DML 命令，看到的就不是 Snapshot，而是命令执行当下所有已经被 commit 的数据。所以在上面的例子中，Transaction 在进行 UPDATE 命令时，看到的就是数据库当下真实的数据，所有已经被 Commit 的数据都包含在内。这也就是为什么 Transaction A 在执行 UPDATE 时可以看到玩家 Frank，并且帮他增加 credit。而且在执行完 UPDATE 后，重新 SELECT 一次时，玩家 Frank 也出现在列表中 (Transaction 可以看到自己所做的更新)。

同样是采用 Snapshot Isolation 实现 RR Isolation 的 PostgreSQL，它的 Snapshot 就不只在 SELECT 命令有效，其他像 INSERT、UPDATE 和 DELETE 等 DML 命令上也都有效。所以上面例子中的 Phantom 现象并不会在 PostgreSQL 发生。

## 如何避免 Phantom 跟 Write Skew？

1.在上面的例子里我们可以用很简单的命令来避免：

```sql
UPDATE gamer SET credit = credit + 1
WHERE name IN ("Alice", "Bob", "Carol");
```

因为是直接指定要增加 credit 的玩家，所以不会意外更新到刚被新增的玩家。
但是其实不是所有的 Write Skew 都可以用这种方法一劳永逸，每个 Write Skew 的情境都是不同的。当然直接改成 Serializable Isolation 就不会有 Write Skew 的发生，但是在不改变 Isolation Level 的情况下，为了避免 Write Skew，我们只能针对每种不同的 Write Skew 现象去设计不同的数据库结构和 Query 方法，或是用Materializing Conflicts 等技巧来防止 Write Skew 的发生。所以，在使用数据库的 Isolation 功能时，我们必须先了解各个 Isolation Level 所有可能发生的 Conflict 和 Race Conditions，才有办法在数据库设计的阶段就将这些因素考虑进去，避免后续的麻烦。

2.另外一种比较暴力的方法就是使用 MySQL 的 Share Lock 或是 Exclusive Lock 命令，Block 住其它想更改数据的 Transaction，例如使用 MySQL 的 LOCK IN SHARE MODE 命令：

```sql
SELECT * FROM gamer LOCK IN SHARE MODE;
UPDATE gamer SET credit = credit + 1
WHERE score >= 740;
COMMIT;
```

3.最直接的方法是将 MySQL 设定为 Serialzable Isolation，MySQL 就会自动为所有的 SELECT 都加上 LOCK IN SHARE MODE。

必须注意的是，不管是手动加 Lock 或是使用 Serialzable Isolation，都会影响到性能。尤其如果没有为字段做好 Index ，就有可能会造成 Full-Table-Lock，应该尽量避免使用。

## 误区2：Repeatable Read Isolation 不会有 Lost Update

在 WIKI Isolation Level 的页面上有下面这一张表：

![隔离级别vs幻读](https://i2.tiimg.com/717558/92a6b5ffc83a2ce3.png)

根据这张表，Repeatable Read Isolation 是可以避免 Lost Update 现象的。但是实际在 MySQL 上测试，如下图中的 Lost Update 例子却成功了：

![更新丢失](https://i2.tiimg.com/717558/afd9cdd112f3d680.png)

在这个例子中，两个 Transaction 同时进行卖出 Item A 的操作，一个卖出 4 个，一个卖出 1 个。理论上，库存记录应该从原本的 10 个减少为 5 个才对。但是最后库存的记录却是 quantity = 9，Transaction A 的更新被 Transaction B 的覆盖掉了，这就是 Lost Update 现象。 Lost Update 现象通常都发生在像这种对数据库做 read-modify-write 的操作。有的数据库会实现 Lost Update 的自动侦测机制来避免这种错误，像是 PostgreSQL 的 RR Isolation。但是 MySQL 则没有，所以 Lost Update 现象是有可能在 MySQL 的 RR Isolation 发生的。

## 如何避免 Lost Update？

1.使用 Atomic Operations

```sql
UPDATE inventory SET quantity = quantity - 4
WHERE item = A;
```

2.使用 SHARE LOCK / EXCLUSIVE LOCK (不建议)

```sql
SELECT * FROM inventory FOR UPDATE;
UPDATE inventory SET quantity = 6 WHERE item = A;
COMMIT;
```

## 正确的 Isolation Level 表格

原本 WIKI 给的表格应该改成如下：

![正确的隔离级别](https://i2.tiimg.com/717558/effce7723e80faed.png)

Repeatable Read Isolation 只保证不会出现 Non-repeatable Read 现象，并不保证不会出现 Lost Update，依照每个数据库对 RR Isolation 的实现方法不同，有的数据库能避免 Lost Update 现象，有的数据库则不能。还有一些例外像 PostgreSQL 的 RR Isolation 还可以避免 Phantom。

1992 年发表的 SQL Standard 对 RR Isolation 的定义其实非常模糊，只要能够避免 Dirty Read 和 Non-Repeatable Read 就可以称作是 RR Isolation，在实现上并没有特别的定义。而 Snapshot Isolation 刚好符合这项定义，所以 MySQL 跟 PostgreSQL 才会称呼他们的 Snapshot Isolation 为 RR Isolation。 PostgreSQL 还另外在 Snapshot Isolation 上实现Lost Update 自动侦测机制，但是 MySQL 则没有。
Lost Update 和 Write Skew 等现象是在 SQL Standard 之后才被发表的，目前都没有对这些现象订定出新的 Isolation Level 标准。所以对于一个数据库是否是 Lost Update Safe，我们无法直接从数据库设定的 Isolation Level 得知，必须另外去了解数据库背后对 RR Isolation 的实现，才能判断。所以，每个数据库的 Isolation Level 表格都会有一些差异，这边分别列出 MySQL 跟 PostgreSQL 的表格。

MySQL：

![MySQL隔离级别](https://i2.tiimg.com/717558/717ea75861d3a66f.png)

PostgreSQL：

![PostgreSQL隔离级别](https://i2.tiimg.com/717558/cbedf3d93c4cd7e3.png)

## 误区3：MySQL Range Locks on Indexed and Non-indexed Column

MySQL 的文档中介绍它的 Range Lock 采用的是 Next-Key Lock，Next-Key Lock 只对有 Index 的字段有作用，没有做 Index 的字段则没有作用，且可能造成 Full Table Lock。我们先看下面的 SELECT 命令：

```sql
SELECT * FROM student WHERE height >= 170 FOR UPDATE;
```

在 MySQL RR Isolation Level 中，SELECT 命令并不会对数据做任何的 Lock，除非额外加上 Shared Lock 或 Exclusive Lock 命令。像在上面的例子使用 FOR UPDATE 命令，就会对所有 SELECT 出来的数据加 Exclusive Lock。对数据加 Shared Lock 或 Exclusive Lock 之后，MySQL 还会另外加 Range Lock。以上面的例子来说，会对 height 这个字段上 Range Lock，Lock 的范围是 170 到无限大，不允许其他 Transaction 新增任何 height 的值介于这个范围内的数据，如下图：

![range lock](https://i2.tiimg.com/717558/b4331509b906a5e1.png)

在上图中，Transaction B 可以很顺利的新增一条 height = 160 的数据，但是想要新增另外一条 height = 180 的数据时，会被 Transaction A 的 Range Lock Block 住，要等到 Transaction A Commit 后才能执行。这个机制的好处是可以只 Lock 所有跟 Transaction A 有关的『数据 Range』，而不是 Lock 整张 Table ，减少对性能的影响。要特别注意的是在 MySQL RR Isolation，如果没有额外加上 Shared Lock 或是 Exclusive Lock 命令，Range Lock 就不会生效。

现在我们改看 weight 这个字段。与 height 字段的差别是，weight 字段并没有做 index，如果对 weight 字段做一样的操作时，结果如下图：

![没index的字段-观察rang lock](https://i2.tiimg.com/717558/bad39876da56a7de.png)

在上图中，Transaction B 想要新增一条 weight = 50 的数据。虽然 50 并不在 Range Lock 的范围 (58 到无限大)，却还是被 Block 住了。这是因为 MySQL 的Range Lock 其实是 Index-record Lock，当 weight 字段没有做 Index 时，就没有该字段的 Index Record 可以做 Lock，为了继续维持 Transaction 之间的Isolation，MySQL 就只好 Lock 整张 student 表格。所以其实不只是无法新增 weight = 50 的数据，在 Transaction A Commit 前，任何对 student 表的新增跟修改都是不允许的。如果没有特别注意，很容易在不知情的情况下造成 Full Table Lock，大大的影响性能。

## 小结

在上面的几个例子中我们可以看到 MySQL 的 Repeatable Read Isolation 对 Lost Update、Phantom 跟 Write Skew 现象的行为。而必须做这些实验的原因，就是

> Nobody really knows what repeatable read means.

这是 Designing Data-Intensive Applications 这本书里面对 Repeatable Read Isolation 的注解。其他三个 Isolation Level 我们都可以很清楚的知道它们分别避免哪些 Race Conditions，但是 Repeatable Read Isolation 的行为则依照每个数据库的实现而有所不同。需要靠使用者自己去阅读文档或是了解数据库背后的实现方法，才能够判别。让我们再复习一次这张表格：

![正确的隔离级别](https://i2.tiimg.com/717558/717ea75861d3a66f.png)

## 参考

- WIKI: [Isolation (database systems)](https://en.wikipedia.org/wiki/Isolation_(database_systems))

- [Designing Data-Intensive Applications](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321)
- [SQL Standard 1992, ISO/IEC 9075:1992](http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)
- [Understanding MySQL Isolation Levels: Repeatable-Read](https://blog.pythian.com/understanding-mysql-isolation-levels-repeatable-read/)
- [InnoDB Locking](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-gap-locks)
- [MySQL 5.7: Transaction Isolation Levels](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html)
- [PostgreSQL 11: Transaction Isolation](https://www.postgresql.org/docs/11/transaction-iso.html)