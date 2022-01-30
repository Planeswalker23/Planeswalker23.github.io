---
layout: post
title: 我所理解的MySQL系列·第6篇·关于MySQL锁的原理你清楚吗
categories: [MySQL]
keywords: MySQL
---



MySQL 系列的第六篇，主要内容是锁（Lock），包括锁的粒度分类、行锁、间隙锁以及加锁规则等。

![mysql-6-封面](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-封面.dwwhqcxkwoo.jpg)



## 1. 开篇词

MySQL 系列的第六篇，主要内容是锁（Lock），包括锁的粒度分类、行锁、间隙锁以及加锁规则等。

MySQL 引入锁的目的是为了解决并发写的问题，比如两个事务同时对同一条记录进行写操作，如果允许它们同时进行，那就会产生**脏写**的问题，这是任何一种隔离级别都不允许发生的异常情况，而锁的作用就是让两个并发写操作按照一定的顺序执行，避免脏写问题。

首先申明本文中所使用到的示例，同时本文所述示例都是在 MySQL InnoDB 存储引擎以及可重复读（Repeatable Read）隔离级别下。
```sql
CREATE TABLE `user`  (
  `id` int(12) NOT NULL AUTO_INCREMENT,
  `name` varchar(36) NULL DEFAULT NULL,
  `age` int(12) NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `age`(`age`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1;

insert into user values (5,'重塑',5),(10,'达达',10),(15,'刺猬',15);
```



## 2. 锁的粒度分类

从锁的粒度来看，MySQL 中的锁可以分为全局锁、表级锁和行锁三种。



### 2.1 全局锁

全局锁会将整个数据库都加上锁，此时数据库将处于只读状态，任何修改数据库的语句，包括 DDL（Data Definition Language）及增删改的 DML（Data Manipulation Language）语句都将被阻塞，直到数据库全局锁释放。

最常使用到全就锁的地方就是进行**全库备份**，我们可以通过以下的语句实现全局锁的加锁与释放锁操作：

```sql
-- 加全局锁
flush tables with read lock;

-- 释放全局锁
unlock table;
```

需要注意的是：若客户端链接断开，也会自动释放全局锁。



### 2.2 表级锁

表级锁会将整张表加上锁，MySQL 中的表级锁有：**表锁**、**元数据锁**（Meta Data Lock）、**意向锁**（Intention Lock）和**自增锁**（AUTO-INC Lock）。



#### 2.2.1 表锁

表锁的加锁和释放锁方式：
- 加锁：`lock table tableName read/write;`
- 释放锁：`unlock table;`

需要注意的是，表锁的加锁也限制了同一个客户端链接的操作权限，如加了表级读锁 **lock table user read**，那么在同一个客户端链接中在释放表级读锁以前，对该表也只能进行读操作，无法进行写操作，而其他客户端链接对该表只能进行读操作，无法进行写操作。

如加了表级写锁 **lock table user write**，在同一个客户端链接中可对表进行读写操作，而其他客户端链接既无法进行读操作也无法进行写操作。



#### 2.2.2 元数据锁

第二种表级锁是**元数据锁**（MDL, Meta Data Lock），元数据锁会在客户端访问表的时候自动加锁，在客户端提交事务时释放锁，它防止了以下场景出现的问题：

| sessionA            | sessionB                                       |
| ------------------- | ---------------------------------------------- |
| begin;              |                                                |
| select * from user; |                                                |
|                     | alter table user add column birthday datetime; |
| select * from user; |                                                |

如上表，sessionA 开启了一个事务，并进行一次查询，在这之后另外一个客户端 sessionB 给 user 表新增了一个 birthday 字段，然后 sessionA 再进行一次查询，如果没有元数据锁，就可能会出现在同一个事务中，前后两次查询到的记录，表字段列数不一致的情况，这显然是需要避免的。

DDL 操作对表加的是元数据写锁，对其他事务的元数据读写锁都不兼容；DML 操作对表加的是元数据读锁，可与其他事务的元数据读锁共享，但与其他事务的元数据写锁不兼容。



#### 2.2.3 意向锁

第三种表级锁是**意向锁**，它表示事务想要获取一张表中某几行的锁（共享锁或排它锁）。

意向锁是为了避免在表中已经存在行锁的情况下，另一个事务去申请表锁而扫描表中的每一行是否存在行锁的系统消耗。

| sessionA                                  | sessionB               |
| ----------------------------------------- | ---------------------- |
| begin;                                    |                        |
| select * from user where id=5 for update; |                        |
|                                           | flush table user read; |

例如，sessionA 开启了一个事务，并对 id=5 这一行加上了行级排它锁，此时 sessionB 将对 user 表加上表级排它锁（只要 user 表中有一行被其他事务持有读锁或写锁即加锁失败）。

如果没有意向锁，sessionB 将扫描 user 表中的每一行，判断它们是否被其他事务加锁，然后才能得出 sessionB 的此次表级排它锁加锁是否成功。

而有了意向锁之后，在 sessionB 将对 user 表加锁时，会直接判断 user 表是否被其他事务加上了意向锁，若有则加锁失败，若无则可以加上表级排它锁。

**意向锁的加锁规则**：
- 事务在获取行级共享锁（S锁）前，必须获取表的意向共享锁（IS锁）或意向排它锁（IX锁）
- 事务在获取行级排它锁（X锁）前，必须获取表的意向排它锁（IX锁）



#### 2.2.4 自增锁

第四种表级锁是**自增锁**，这是一种特殊的表级锁，只存在于被设置为 AUTO_INCREMENT 的自增列，如 user 表中的 id 列。

自增锁会在 insert 语句执行完成后立即释放。同时，自增锁与其他事务的意向锁可共享，但与其他事务的自增锁、共享锁和排它锁都是不兼容的。



### 2.3 行锁

行锁是由存储引擎实现的，从行锁的兼容性来看，InnoDB 实现了两种标准行锁：**共享锁**（Shared Locks，简称S锁）和**排它锁**（Exclusive Locks，简称X锁）。

这两种行锁的兼容关系与上面元数据锁的兼容关系是一样的，可以用下面的表格表示。

| 事务A\事务B   | 共享锁（S锁） | 排它锁（X锁） |
| ------------- | ------------- | ------------- |
| 共享锁（S锁） | 兼容          | 冲突          |
| 排它锁（X锁） | 冲突          | 冲突          |

而从行锁的粒度继续细分，又可以分为**记录锁**（Record Lock）、**间隙锁**（Gap Lock）、**Next-key Lock**。



#### 2.3.1 记录锁（Record Lock）

我们一般所说的行锁都是指记录锁，它会把数据库中的指定记录行加上锁。

假设事务A中执行以下语句（未提交）：

```sql
begin;
update user set name='达闻西' where id=5;
```

InnoDB 至少会在 id=5 这一行上加一把行级排它锁（X锁），不允许其他事务操作 id=5 这一行。需要注意的是，这把锁是加在 id 列的主键索引上的，也就是说行级锁是加在索引上的。

假设现在有另一个事务B想要执行一条更新语句：

```sql
update user set name='大波浪' where id=5;
```

这时候，这条更新语句将被阻塞，直到事务A提交以后，事务B才能继续执行。

![mysql-6-lock1-记录锁示意图](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock1-记录锁示意图.400mz9lg12m0.png)



#### 2.3.2 间隙锁（Gap Lock）

间隙锁，顾名思义就是给记录之间的间隙加上锁。这里需要注意的是，间隙锁只存在于可重复读（Repeatable Read）隔离级别下。

不知道大家还记不记得幻读？幻读是指在同一事务中，连续执行两次同样的查询语句，第二次的查询语句可能会返回之前不存在的行。

间隙锁的提出正是为了防止幻读中描述的幻影记录的插入而提出的，举个例子。

| sessionA                                   | sessionB                                |
| ------------------------------------------ | --------------------------------------- |
| begin;                                     |                                         |
| select * from user where age=5;(N1)        |                                         |
|                                            | insert into user values(2, '大波浪', 5) |
| update user set name='达闻西' where age=5; |                                         |
| select * from user where age=5;(N2)        |                                         |

sessionA 中有两处查询N1和N2，它们的查询条件都是 age=5，唯一不同的是在N2处的查询前有一条更新语句。

照理说在 RR 隔离级别下，同一个事务中两次查询相同的记录，结果应该是一样的。但是在经过更新语句的当前读查询后（更新语句的影响行数是2），N1和N2的查询结果并不相同，N2的查询将 sessionB 插入的数据也查出来了，这就是幻读。

而如果在 sessionA 中的两次次查询都用上间隙锁，比如都改为 **select * from user where age=5 for update**,那么 sessionA 中的当前读查询语句至少会将id在 (-∞, 5) 和 (5, 10) 之间的间隙加上间隙锁，不允许其他事务插入主键id属于这两个区间的记录，即会将 sessionB 的插入语句阻塞，直到 sessionA 提交之后，sessionB 才会继续执行。

也就是说，当N2处的查询执行时，sessionB 依旧是被阻塞的状态，所以N1和N2的查询结果是一样的，都是(5,重塑,5)，也就解决了幻读的问题。

![mysql-6-lock2-间隙锁示意图](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock2-间隙锁示意图.5s9mqudc1h8.png)



#### 2.3.3 Next-key Lock

Next-key Lock 其实就是**记录锁与记录锁前面间隙的间隙锁**组合的产物，它既阻止了其他事务在间隙的插入操作，也阻止了其他事务对记录的修改操作。

![mysql-6-lock3-Next-key-mysql-6-lock锁示意图](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock3-Next-key-mysql-6-lock锁示意图.2s0nbnu0cdk0.png)



## 3. MySQL是如何加锁的

不知道大家有没有注意到，我在行锁部分描述记录锁、间隙锁加锁的具体记录时，用的是「至少」二字，并没有详细说明具体加锁的是哪些记录，这是因为记录锁、间隙锁和 Next-key Lock 的加锁规则是十分复杂的，这也是下面部分将要主要讨论的内容。

关于加锁规则的叙述将分为三个方面：唯一索引列、普通索引列和普通列，每一方面又将细分为等值查询和范围查询两方面。

需要注意的是，这里加的锁都是指排它锁。在开始之前，先来回顾一下示例表以及表中可能存在的行级锁。

```sql
mysql> select * from user;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  5 | 重塑   |    5 |
| 10 | 达达   |   10 |
| 15 | 刺猬   |   15 |
+----+--------+------+
3 rows in set (0.00 sec)
```

表中可能包含的行级锁首先是每一行的记录锁——(5,重塑,5),(10,达达,5),(15,刺猬,15)。

假设 user 表的索引值有最大值 maxIndex 和最小值 minIndex，user 表还可能存在间隙锁(minIndex,5),(5,10),(10,15),(15,maxIndex)。

共三个记录锁和四个间隙锁，当然还包括若干个 Next-key Lock。



### 3.1 唯一索引列等值查询

#### 3.1.1 唯一索引列等值查询命中 

首先来说唯一索引列的等值查询，这里的等值查询可以分为两种情况：命中与未命中。

当唯一索引列的等值查询命中时：

| sessionA                                  | sessionB                                                     | 阻塞情况    |
| ----------------------------------------- | ------------------------------------------------------------ | ----------- |
| begin;                                    |                                                              |             |
| select * from user where id=5 for update; |                                                              |             |
|                                           | insert into user values(1,'斯斯与帆',1),(6,'夏日阳光',6),(11,'告五人',11),(16,'面孔',16); |             |
|                                           | update user set age=18 where id=5;                           | **Blocked** |
|                                           | update user set age=18 where id=10;                          |             |
|                                           | update user set age=18 where id=15;                          |             |

上表中 sessionB 的执行结果是除了 id=5 行的更新语句被阻塞，其他语句都正常执行。

sessionB 中的 insert 语句是为了检查间隙锁，update 语句是为了检查记录锁（行锁）。执行结果表明 user 表的所有间隙都没有被上锁，记录锁中只有 id=5 这一行被上锁了。

![mysql-6-lock4-select-*-from-user-where-id=5-for-update-加锁区域示意图](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock4-select-*-from-user-where-id=5-for-update-加锁区域示意图.765ce77dp9w0.png)

所以，当唯一索引列的等值查询命中时，**只会给命中的记录加锁**。



#### 3.1.2 唯一索引列等值查询未命中

当唯一索引列的等值查询未命中时：

| sessionA                                  | sessionB                                  | 阻塞情况    |
| ----------------------------------------- | ----------------------------------------- | ----------- |
| begin;                                    |                                           |             |
| select * from user where id=7 for update; |                                           |             |
|                                           | insert into user values (2,'反光镜',2);   | **Blocked** |
|                                           | update user set age=18 where id=5;        |             |
|                                           | insert into user values (6,'夏日阳光',6); |             |
|                                           | update user set age=18 where id=10;       |             |
|                                           | insert into user values (11,'告五人',11); |             |
|                                           | update user set age=18 where id=15;       |             |
|                                           | insert into user values (16,'面孔',16);   |             |

上表的执行结果是 sessionB 中 id=6 的记录插入被阻塞，其他语句正常执行。

根据执行结果可以知道 sessionA 给 user 表加的锁是间隙锁(5, 10)。

![mysql-6-lock5-select-*-from-user-where-id=3-for-update-加锁区域示意图](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock5-select-*-from-user-where-id=3-for-update-加锁区域示意图.1bl6w1j5h5ls.png)

所以，当唯一索引列的等值查询未命中时，**会给id值所在的间隙加上间隙锁**。



### 3.2 唯一索引列范围查询

范围查询比等值查询要更复杂一些，它需要考虑到边界值存在于表中，以及是否命中边界值。

#### 3.2.1 边界值存在，但未命中

首先来看边界值存在于表中，但未命中时，sessionB 的阻塞情况：

| sessionA                                   | sessionB                                  | 阻塞情况    |
| ------------------------------------------ | ----------------------------------------- | ----------- |
| begin;                                     |                                           |             |
| select * from user where id<10 for update; |                                           |             |
|                                            | insert into user values (1,'斯斯与帆',1); | **Blocked** |
|                                            | update user set age=18 where id=5;        | **Blocked** |
|                                            | insert into user values (6,'夏日阳光',6); | **Blocked** |
|                                            | update user set age=18 where id=10;       | **Blocked** |
|                                            | insert into user values (11,'告五人',11); |             |
|                                            | update user set age=18 where id=15;       |             |
|                                            | insert into user values (16,'面孔',16) ;  |             |

此时 sessionA 给 user 表加上的锁是记录锁 id=5,id=10 以及间隙锁(minIndex,5),(5,10)。

我们知道记录锁+间隙锁就是 **Next-key Lock**，所以上述的加锁情况可以看作是两条 Next-key Lock：(minIndex, 5],(5,10]，而加锁情况连起来就是 (minIndex,10]。

![mysql-6-lock6-select-*-from-user-where-id<10-for-update-加锁区域示意图](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock6-select-*-from-user-where-id<10-for-update-加锁区域示意图.61n6vce47000.png)



#### 3.2.1 边界值存在，且命中

当边界值存在于表中，同时命中时 sessionB 的阻塞情况：

| sessionA                                    | sessionB                                                 |
| ------------------------------------------- | -------------------------------------------------------- |
| begin;                                      |                                                          |
| select * from user where id<=10 for update; |                                                          |
|                                             | insert into user values (1,'斯斯与帆',1);（**Blocked**） |
|                                             | update user set age=18 where id=5;（**Blocked**）        |
|                                             | insert into user values (6,'夏日阳光',6);（**Blocked**） |
|                                             | update user set age=18 where id=10;（**Blocked**）       |
|                                             | insert into user values (11,'告五人',11);（**Blocked**） |
|                                             | update user set age=18 where id=15;（**Blocked**）       |
|                                             | insert into user values (16,'面孔',16) ;                 |

此时 sessionA 给 user 表加上的锁是三个 Next-key Lock，加起来就是 (minIndex,15]。

![mysql-6-lock7-select-*-from-user-where-id<=10-for-update-加锁区域示意图](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock7-select-*-from-user-where-id<=10-for-update-加锁区域示意图.1kuscinxsrkw.png)

#### 3.2.3 边界值不存在

当边界值不存在于表中时，不可能命中，故只有未命中一种情况：

| sessionA                                   | sessionB                                  | 阻塞情况    |
| ------------------------------------------ | ----------------------------------------- | ----------- |
| begin;                                     |                                           |             |
| select * from user where id<=9 for update; |                                           |             |
|                                            | insert into user values (1,'斯斯与帆',1); | **Blocked** |
|                                            | update user set age=18 where id=5;        | **Blocked** |
|                                            | insert into user values (6,'夏日阳光',6); | **Blocked** |
|                                            | update user set age=18 where id=10;       | **Blocked** |
|                                            | insert into user values (11,'告五人',11); |             |
|                                            | update user set age=18 where id=15;       |             |
|                                            | insert into user values (16,'面孔',16) ;  |             |

此时 sessionA 给 user 表加上锁的范围是 (minIndex,10]，与第一种情况一样。

![mysql-6-lock8-select-*-from-user-where-id<=9-for-update-加锁区域示意图](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock8-select-*-from-user-where-id<=9-for-update-加锁区域示意图.7besel6nmfk0.png)

综上所述，在对唯一索引进行范围查询时：
1. **会给范围中的记录加上记录锁，间隙加上间隙锁**
2. **对于范围查询(大于/大于等于/小于/小于等于)是比较特殊的，它会将记录锁加到第一个边界之外的记录上，若其中有额外的间隙也会加上间隙锁（即会将 Next-key Lock 加到第一个边界之外的记录上）**

需要注意的是，第一条中所说的间隙指的是，边界值所在的间隙，如间隙为(5,10)，查询条件为 id>7 时，这个间隙锁就是(5,10)，而不是(7,10)。

第二条举例1：查询条件为 id<10，第一个边界之外的记录是 id=10，所以 Next-key Lock 锁会加到 id=10 的记录上，被锁住的范围是(minIndex,10]。

第二条举例2：查询条件为 id<=10，第一个边界之外的记录是 id=15，所以 Next-key Lock 锁会加到 id=15 的记录上，被锁住的范围是(minIndex,15]。

第二条举例3：查询条件为 id>10，第一个边界之外的记录是 id=10，Next-key Lock 锁会加到 id=10 的记录上，由于 Next-key Lock 锁指的是记录以左的部分，所以被锁住的范围是(5,maxIndex]。



### 3.3 普通索引列等值查询

普通索引与唯一索引的区别就在于唯一索引可以根据索引列确定唯一性，所以等值查询的加锁规则也有不同之处。

给 user 表再加一条记录:

```sql
INSERT INTO user VALUES (11, '达达2.0', 10);
```

这时 user 表的索引 age 结构如下图所示：

![mysql-6-lock9-索引-age-结构](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock9-索引-age-结构.6c02ny7xrpo0.png)

在索引 age 中可能存在的行锁是4个记录锁以及5个间隙锁。

先来看索引 age 上的加锁情况：

| sessionA                                    | sessionB                                            | 阻塞情况    |
| ------------------------------------------- | --------------------------------------------------- | ----------- |
| begin;                                      |                                                     |             |
| select * from user where age=10 for update; |                                                     |             |
|                                             | insert into user values (2,'达达',2);               |             |
|                                             | update user set name='痛仰' where age=5;            |             |
|                                             | insert into user values (6,'达达',6);               | **Blocked** |
|                                             | update user set name='痛仰' where age=10 and id=10; | **Blocked** |
|                                             | update user set name='痛仰' where age=10 and id=16; | **Blocked** |
|                                             | insert into user values (17,'达达',10);             | **Blocked** |
|                                             | insert into user values (11,'达达',11);             | **Blocked** |

由上表的语句及执行结果来看，索引 age 上的加锁情况是：

![mysql-6-lock10-select-*-from-user-where-age=10-for-update-索引age上的加锁情况](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock10-select-*-from-user-where-age=10-for-update-索引age上的加锁情况.2q33vt6km9e.png)

即索引 age 上的加锁区域为(5, 15)。

由于普通索引无法确定记录的唯一性，所以普通索引列等值查询中，为索引 age 加锁时，**会找到第一个age小于10的值（即5）和第一个age大于10的值（即15），在这个范围内的间隙加上间隙锁，记录加上记录锁**。

这是索引 age 上的加锁情况，由于查询语句是查询记录的所有列，根据查询规则，会通过索引 age 上对应的 id 值到主键索引树上进行回表操作，得到所有列，所以主键索引上也会加锁。在这里，满足 age=10 的记录的主键id分别是10和16，所以在主键索引上这两行也会被加上排它锁。

即普通索引列等值查询**如果需要回表，满足条件的记录对应的主键也会被加上记录锁**。但是这里如果把 sessionA 中的查询改为

```sql
select id from user where age=10 lock in share mode; 
```

则会因为覆盖索引优化而不进行回表操作，所以主键索引上也不会加锁。



### 3.4 普通索引列等值查询+LIMIT

这里需要额外提一提 LIMIT 这个语法，它的加锁范围（只讨论普通索引）要更小一些，请看示例：

| sessionA                                            | sessionB                                            | 阻塞情况    |
| --------------------------------------------------- | --------------------------------------------------- | ----------- |
| begin;                                              |                                                     |             |
| select * from user where age=10 limit 1 for update; |                                                     |             |
|                                                     | insert into user values (2,'达达',2);               |             |
|                                                     | update user set name='痛仰' where age=5;            |             |
|                                                     | insert into user values (6,'达达',6);               | **Blocked** |
|                                                     | update user set name='痛仰' where age=10 and id=10; | **Blocked** |
|                                                     | update user set name='痛仰' where age=10 and id=16; |             |
|                                                     | insert into user values (17,'达达',10);             |             |
|                                                     | insert into user values (11,'达达',11);             |             |

可以看到，与没有加 limit 相比，多了两条 insert 语句顺利执行了。

由上表的语句及执行结果来看，索引 age 上的加锁情况是：

![mysql-6-lock11-select-*-from-user-where-age=10-limit-1-for-update加锁区域示意图](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock11-select-*-from-user-where-age=10-limit-1-for-update加锁区域示意图.6pqd2vuo6g80.png)

由此可见：**LIMIT 语法只会将锁加到满足条件的记录**，能够减小加锁范围。



### 3.5 普通索引列范围查询

接下来看普通索引列上的范围查询，这里只讨论索引 age 的加锁范围，主键索引的加锁如果存在回表会锁住对应的id值：

| sessionA                                               | sessionB                                            | 阻塞情况    |
| ------------------------------------------------------ | --------------------------------------------------- | ----------- |
| begin;                                                 |                                                     |             |
| select * from user where age>8 and age<=12 for update; |                                                     |             |
|                                                        | insert into user values (2,'达达',2);               |             |
|                                                        | update user set name='痛仰' where age=5;            |             |
|                                                        | insert into user values (6,'达达',6);               | **Blocked** |
|                                                        | update user set name='痛仰' where age=10 and id=10; | **Blocked** |
|                                                        | update user set name='痛仰' where age=10 and id=16; | **Blocked** |
|                                                        | insert into user values (17,'达达',10);             | **Blocked** |
|                                                        | insert into user values (11,'达达',11);             | **Blocked** |

与普通索引列等值查询不同的是，范围查询比等值查询多了一个 age=15 的记录锁。

![mysql-6-lock12-select-*-from-user-where-age>8-and-age<=12-for-update-索引age上的加锁情况示意图](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-6-lock12-select-*-from-user-where-age>8-and-age<=12-for-update-索引age上的加锁情况示意图.6tomaqs8pmo.png)

这个边界值与唯一索引列范围查询的原理是一样的，可以参照上文所述来理解，这里不多加赘述了。

《MySQL实战45讲》的作者丁奇认为这是一个 BUG，但并未被官方接收，如果要深究这个边界值的原理，可能就需要看 MySQL 的源码了。



## 4. 温故知新

1. MySQL 中的锁按粒度来分可以分为几种？
2. MySQL 中行锁的加锁规则？
3. 请说出下面几条 SQL 的加锁区域：

```sql
select * from user where c=10 for update;
select * from user where c>=10 and c<11 for update;
select id from user where c>=10 and c<11 for update;
```



## 5. 参考资料

- [MySQL实战45讲](https://time.geekbang.org/column/intro/139)
- [MySQL技术内幕：InnoDB存储引擎（第2版）](https://weread.qq.com/web/reader/611329b059346e611427f1ckc81322c012c81e728d9d180)
- [抱歉，没早点把这么全面的InnoDB锁机制发给你](https://dbaplus.cn/news-11-2518-1.html)

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。