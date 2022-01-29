---
layout: post
title: 我所理解的MySQL系列·第1篇·MySQL有哪些组成部分
categories: [MySQL]
keywords: MySQL
---



这是 MySQL 系列的第一篇，主要介绍 MySQL 的基础架构以及各个组成部分的功能，包括 Server 层的 bin log 和 InnoDB 特有的 redo log 这两种日志模块。



![MySQL封面](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/1/MySQL封面.4khjjmxzq1k0.jpg)




作为一个正经的 CRUD 工程师，与数据库的交互是日常工作中比重较大的内容，比如日常迭代的增删改查、处理历史数据、优化 SQL 性能等等。随着项目数据量的增长，从前为了赶项目进度而埋下的深坑正慢慢显露它们的威力，这也让我不得不全面且深入的学习 MySQL，而不仅仅是停留在基础的 CRUD 上。

这是 MySQL 系列的第一篇，主要介绍 MySQL 的基础架构以及各个组成部分的功能，包括 Server 层的 bin log 和 InnoDB 特有的 redo log 这两种日志模块。



## 1. MySQL 架构简介
根据 DB-Engines 发布的[最受欢迎的数据库管理系统排行榜](https://db-engines.com/en/ranking)，MySQL 稳坐第二把交椅。

![mysql-1-1](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/1/mysql-1-1.3mtxylbskom0.jpg)

作为最受欢迎的关系型数据库管理系统之一，MySQL 采用的是C/S架构，即 Client & Server 架构。比如开发者使用 Navicat 连接到 MySQL，那么前者就是客户端，后者就是服务端。

同时，MySQL 也是单进程多线程的数据库。这很好理解，正在运行的 MySQL 实例就是那个“单进程”，而在这个进程中会有很多个线程，比如主线程 Master Thread，IO Thread 等，这些线程被用于处理不同的任务。



## 2. MySQL 组成部分
前面说到 MySQL 采用的是C/S架构，用户通过客户端连接到 MySQL 服务器，然后提交 SQL 语句到服务器，然后服务器就会把执行结果返回给客服端。

在这一小节的内容中，我们主要关注 MySQL 服务端的逻辑组成，先来看一张图。

![mysql-1-2](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/1/mysql-1-2.15abw3mvvnsw.jpg)

从上图可以看到，与客户端的交互中，MySQL 的服务端分别经过了连接器、查询缓存、分析器、优化器、执行器和存储引擎这几部分。

下面就以一条简单的查询语句的执行流程来描述 MySQL 服务端的各组成部分及它们所起的作用。



### 2.1 连接器
在客户端提交查询语句之前，需要与服务端建立连接。所以最先来到的是连接器，连接器的作用就是**负责与客户端建立、管理连接，同时查询用户的权限**。

需要注意的是：
- 连接器只获取用户的权限，并不做校验，校验是在查询缓存或执行器才进行。
- 一旦建立连接同时获取用户的权限之后，只有建立新的连接才会刷新用户权限。
- 对于长时间没有发送请求的客户端，连接器会自动断开连接。这里的「长时间」是由 wait_timeout 参数来决定的，它的默认值为8小时。



### 2.2 查询缓存
在经过连接器的建立连接、获取用户权限之后，接下来用户可以提交查询语句了。

最先经过的是查询缓存部分，由它的名字也能够猜到，查询缓存的作用就是**查询 MySQL 是否执行过客户端提交的查询语句**，如果这条 SQL 之前执行过，并且用户对该表有执行该语句的权限，就会直接返回之前执行的结果。

所以在某些时候，多次执行一句 SQL 并不能得到它的平均执行时间，因为查询缓存的关系，后面的执行时间往往比第一次执行要短。

如果你不想使用缓存，可以在每次查询后都用 update 语句更新表，当然这是非常麻烦并且憨的方法。MySQL也提供了相应的配置项—— `query_cache_type`，你可以在 `my.cnf` 文件中将 `query_cache_type` 设置为0以关闭查询缓存。

需要注意的是：
- 查询缓存部分是以 `key-value` 形式进行存储的，key 为查询语句，value 是查询结果。
- 当对数据表进行更新时，关于这张表的所有查询缓存都会失效，所以一般来说查询缓存的命中率是很低的。
- 在 `MySQL 8.0` 的版本中，查询缓存的功能已经被删除。



### 2.3 分析器

我使用的 MySQL 版本是5.7.21，所以客户端提交的查询语句会走查询缓存，如果没有命中，那么将继续往下走，来到分析器。

分析器会对提交的语句进行词法分析（解析语句）和语法分析（判断语句是否符合 MySQL 的语法规则），所以分析器的作用就是**解析 SQL 语句并检查其合法性**。

需要注意的是：
- MySQL 在检查 SQL 语句合法性时，仅会在最先不符合 MySQL 语法规则的地方提示错误，并不会将 SQL 语句中所有语法错误的地方全部展示。

举个例子：

```sql
select * form user_info limit 1;
```

上面这句 SQL 有两个错误，第一是 from 拼写错误，第二是不存在 user_info 这张表，在执行之后，MySQL只会提醒一个错误，下面展示了三次执行 SQL 的结果信息。

```sql
第一次的执行信息：
1064 - You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'form user_info limit 1' at line 1, Time: 0.000000s

修改为from后第二次的执行信息：
1146 - Table 'windfall.user_info' doesn't exist, Time: 0.000000s

修改为 user 表后第三次的执行信息：
OK, Time: 0.000000s
```



### 2.4 优化器

在校验了 SQL 语句的合法性之后，MySQL 已经知道用户提交的语句是干什么的了，但是在真正执行之前，还需要经过非常“玄学”的优化器。

![mysql-1-3](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/1/mysql-1-3.6gvtymzznps0.jpg)

优化器的作用是**为 SQL 语句生成最优的执行计划**。

之所以说优化器很“玄学”，是因为它在优化 SQL 语句的过程中可能会生成出乎用户意料之外的执行计划（索引选择、多表关联连接顺序、隐式函数转换等）。当然优化器有时候也会“选错”索引，这与数据量、索引统计信息等因素有关。

需要注意的是：
- 如果你需要优化一条生产环境的 SQL，请尽量在本地还原与生产环境数据量相同的表，然后根据执行计划进行优化。
- 在写查询语句的时候，一定要考虑到索引的最左匹配原则（关于最左匹配原则的整理在索引篇再写）。

关于 MySQL 优化器的工作流程，可以看看这篇博客：[MySQL 优化器原来是这样工作的](https://blog.csdn.net/horses/article/details/105841886)

MySQL 的执行计划也是一项必须要掌握的技能，这篇博客写得非常详细，值得一读：[不会看 Explain执行计划，劝你简历别写熟悉 SQL优化](https://juejin.im/post/6844904163969630221)



### 2.5 执行器

在优化器生成了 MySQL 认为最优的执行计划之后，最后来到了执行器，执行器的作用当然就是**执行SQL语句**了。

但是在执行之前，先要做权限验证，验证用户对表是否有查询权限。然后再根据表定义的引擎类型，去使用相对应引擎提供的接口来对该表进行条件查询，最后将该表所有满足条件的数据行作为结果集返回客户端，这样整个 SQL 的执行就结束了。

需要注意的是：
- 在执行器执行 SQL 语句前会做校验：判断用户对表是否具有操作权限。



### 2.6 存储引擎

MySQL 支持的存储引擎有很多种，比如：InnoDB、MyISAM、Memory 等等。

#### 2.6.1 InnoDB
InnoDB 是当下最常用的的 MySQL 存储引擎，同时也是 MySQL 5.5 之后的默认存储引擎。

InnoDB 支持事务、MVCC（多版本并发控制）、外键、行级锁和自增列。但是 InnoDB 不支持全文索引，同时它占用的数据空间更大。

#### 2.6.2 MyISAM
MyISAM 是 MySQL 5.1 及之前的默认存储引擎，支持全文索引、压缩、空间函数、表级锁。

MyISAM 的数据以紧密格式存储所以占用空间更小，它拥有较高的插入和查询速度，但是 MyISAM 不支持事务，且崩溃后无法安全恢复。

#### 2.6.3 Memory
Memory 的所有数据都保存的内存中，由于不需要磁盘 I/O，所以它的速度比 MyISAM 和 InnoDB 快了一个数量级。但如果数据库关闭或重启，Memory 引擎的数据就会消失。

Memory 支持 Hash 索引，但由于它使用表级锁，因此并发写入的性能比较低。

值得一提的是，MySQL 中的临时表，一般是用 Memory 表保存的，如果中间表数据量过大或含有 BLOB 类型或 TEXT 类型的字段，就会使用 MyISAM 表。

> 关于存储引擎，由于本人接触的比较少，等看完《MySQL技术内幕：InnoDB存储引擎》之后再整理，这里只是简单地提一下。



## 3. 日志模块

前面所说的执行流程主要是描述查询语句，如果是更新语句还涉及到 MySQL 的日志模块。

从客户端到执行器的之间的逻辑查询语句和更新语句是相同的，只是在到执行器这一层的时候，更新语句会和 MySQL 的日志模块产生交互，这是查询语句和更新语句不一样的地方。



### 3.1 物理日志 redo log

#### 3.1.1 redo log 中记录的内容
对于 InnoDB 存储引擎来说，它有一个特有的日志模块——物理日志（重做日志）**redo log**，它是 InnoDB 存储引擎的日志，它所记录的是**数据页的物理修改**。

举个例子，现在有一张 user 表，有一条主键 id=1，age=18 的数据，然后用户提交了下面这条 SQL，执行器准备执行。

```sql
update user set age=age+1 where id=1;
```

对于这条 SQL，在 redo log 中记录的内容大致是：`将 user 表中主键 id=1 行的 age 字段值修改为19`。

#### 3.1.2 WAL
MySQL 的更新持久化逻辑运用到了 **WAL**(Write-Ahead Logging，写前日志记录) 的思想：先写日志，再写磁盘。

需要注意的是这里的写日志也是写到磁盘中，但由于日志是**顺序写入**的，所以速度很快。而如果没有 redo log，直接更新磁盘中的数据，那么首先需要找到那条记录，然后再把新的值更新进入，由于查询和读写I/O，就相对会慢一些。

最后，当 InnoDB 引擎空闲的时候，它会去执行 redo log 中的逻辑，将数据持久化到磁盘中。

#### 3.1.3 redo log 日志文件
redo log 日志文件大小是固定的，我把它理解为一个循环链表，链表的每个节点都可以存放日志，在这个链表中有两个指针：write（黑） 和 read（白）。

![mysql-1-4](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/1/mysql-1-4.6c7y8p4pm300.jpg)

最开始这两个指针都指向同一个节点，且节点日志元素都为空，表示此时 redo log 为空。当用户开始提交更新语句，write 节点开始往前移动，假设移动到3的位置。而此时的情况就是 redo log 中有1-3这三个日志元素需要被持久化到磁盘中，当 InnoDB 空闲时，read 指针往前移动，就代表着将 redo log 持久化到磁盘。

但这里有一种特殊情况，就是 InnoDB 一直没有空闲，write 指针一直在写入日志，直到它写到5的位置，再往前写又回到了最开始1的位置（也就是上图的位置，但不同的是链表节点中都存在日志数据）。

此时发现1的位置已经有日志数据了，同时 read 指针也在。那么这时候 write 指针就会暂停写入，InnoDB 引擎开始催动 read 指针移动，把 redo log 清空掉一部分之后再让 write 指针写入日志文件。

#### 3.1.4 redo log 的作用
我们已经知道，redo log 中记录的是**数据页的物理修改**，所以 redo log 能够保证在数据库发生异常重启时，记录尚未写入磁盘，但是在重启后可以通过 redo log 来“redo”，从而不会发生记录丢失的情况，保证了事务的持久性。

这一能力也被称作 **crash-safe**。



### 3.2 归档日志 bin log

前面说到 redo log 是 InnoDB 特有的日志，而 bin log 则是属于 MySQL Server 层的日志，在默认的 Statement Level 下它记录的是更新语句的原始逻辑，即 SQL 本身。

另外需要注意的是：
- bin log 的日志文件大小并不固定，它是“追加写入”的模式，写完一个文件后会切换到下一个文件写入。
- bin log 没有 crash-safe 的能力。
- bin log 是在事务最终提交前写入的，而 redo log 是在事务执行中不断写入的。

最后，bin log 常用于恢复数据，比如说主从复制，从节点根据父节点的 bin log 来进行数据同步，实现主从同步。



### 3.3 两阶段提交

为了让 redo log 和 bin log 的状态保持一致，MySQL 使用**两阶段提交**的方式来写入 redo log 日志。

在执行器调用 InnoDB 引擎的接口将写入更新数据时，InnoDB 引擎会将本次更新记录到 redo log 中，同时将 redo log 的状态标记为 prepare，表示可以提交事务。

随后执行器生成本次操作的 bin log 数据，并写入 bin log 的日志文件中。

最后执行器调用 InnoDB 的提交事务接口，存储引擎把刚写入的 redo log 记录状态修改为 commit，本次更新结束。

在这个过程中有三个步骤 `add redo log and mark as prepare` -> `add bin log` -> `commit`，即：
1. 写入 redo log 日志并标记为 prepare
2. 写入 bin log
3. 提交事务

如果在第二个步骤，也就是写入 bin log 之前系统崩溃或重启，启动后由于 bin log 中没有记录，会将 redo log 中的记录回滚至执行本次更新语句前。

如果在第三个步骤前，也就是提交之前系统崩溃或重启，即便没有 commit 但是满足 redo log 中记录为 prepare 状态并且 bin log 中也有完整记录，在重启后会自动 commit，并不会回滚。



## 4. 小结

本文主要介绍 MySQL 的基础架构以及各个组成部分的功能，最后介绍了 MySQL Server 层的 bin log 和 InnoDB 特有的 redo log 这两种日志模块。



## 5. 温故知新

以下的几个问题是对本文所描述内容的提问，巩固知识，正所谓“温故而知新，可以为师矣”。

1. 如果查询语句中字段不存在、字段有歧义、关键字拼写错误，是由哪个部分报错？
2. 如果用户对表没有查询权限，是哪个部分报错？
3. 为什么 MySQL 的查询缓存会无效？
4. 一条 select 查询语句是如何执行的？
5. MySQL 常用的存储引擎有哪些？
6. MySQL 的日志模块有哪些？分别起到什么作用？
7. redo log 写满了怎么办？
8. 如何理解 redo log 的两阶段提交？
9. redo log 和 bin log 的区别？



## 6. 参考资料

- [《MySQL实战45讲》01 | 基础架构：一条SQL查询语句是如何执行的？](https://time.geekbang.org/column/article/a57ae0dd0daac940338e9c1b084b0b7d/share?code=cbHwlgRYeERonSROIcKJXe0PPC4kk7tccvKC0sfh3Rc%3D&oss_token=2046f4a7a951dd81)（此链接有20个免费阅读该文章的名额）
- [《MySQL实战45讲》02 | 日志系统：一条SQL更新语句是如何执行的？](https://time.geekbang.org/column/article/ef8825593d260c8a45ede7f9c211f1a0/share?code=cbHwlgRYeERonSROIcKJXe0PPC4kk7tccvKC0sfh3Rc%3D)（此链接有20个免费阅读该文章的名额）
- [MySQL存储引擎的区别与比较](https://blog.csdn.net/keil_wang/article/details/88392433)

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。