---
layout: post
title: 我所理解的MySQL系列·第b篇·为什么LIMIT百万偏移量这么慢
categories: [MySQL]
keywords: MySQL, limit
---



在 MySQL 中通常我们使用 limit 来完成页面上的分页功能，但是当数据量达到一个很大的值之后，越往后翻页，接口的响应速度就越慢。本文主要讨论 limit 分页大偏移量执行速度慢的原因及优化方案，为了模拟这种情况，下面首先介绍表结构和执行的 SQL。



![MySQL封面](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/a/MySQL封面.4tcykd7jjyi0.jpg)



## 1. 开篇词

在 MySQL 中通常我们使用 limit 来完成页面上的分页功能，但是当数据量达到一个很大的值之后，越往后翻页，接口的响应速度就越慢。

本文主要讨论 limit 分页大偏移量执行速度慢的原因及优化方案，为了模拟这种情况，下面首先介绍表结构和执行的 SQL。




## 2. 场景模拟


### 2.1 建表语句


首先建立一张用户表 user，它的结构比较简单，包含 id、sex 和 name 三个字段，为了让 SQL 的执行时间变化更加明显，这里拷贝了9个姓名列。


```sql
CREATE TABLE `user`  (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `sex` tinyint(4) NULL DEFAULT NULL COMMENT '性别 0-男 1-女',
  `name1` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '姓名',
  `name2` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '姓名',
  `name3` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '姓名',
  `name4` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '姓名',
  `name5` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '姓名',
  `name6` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '姓名',
  `name7` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '姓名',
  `name8` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '姓名',
  `name9` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '姓名',
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `sex`(`sex`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;
```



### 2.2 数据填充

然后我创建了一个存储过程来进行数据的填充，一共9000000条数据。这个函数执行会比较耗时，我运行了617.284秒。

```sql
CREATE DEFINER=`root`@`localhost` PROCEDURE `data`()
begin 
  declare i int; 
  set i=1; 
  while(i<=9000000)do 
    insert into user values(i,0,i,i,i,i,i,i,i,i,i);
    set i=i+1; 
  end while;
end
```

执行完函数后再执行一句修改性别字段的 SQL，是为了体现索引的效果。

```sql
-- 将id为偶数的user设置性别为1-女
update user set sex=1 where id%2=0;
```



### 2.3 SQL 与执行时间

| SQL | 执行时间 |
| --- | --- |
| select * from user where sex = 1 limit 100, 10; | OK, Time: 0.005000s |
| select * from user where sex = 1 limit 1000, 10; | OK, Time: 0.007000s |
| select * from user where sex = 1 limit 10000, 10; | OK, Time: 0.016000s |
| select * from user where sex = 1 limit 100000, 10; | OK, Time: 0.169000s |
| select * from user where sex = 1 limit 1000000, 10; | OK, Time: 5.892000s |
| select * from user where sex = 1 limit 10000000, 10; | OK, Time: 33.465000s |

可以看到，limit 的偏移量越大，执行时间越长，百万级别时执行时间已经有半分钟了，这还仅仅是一张简单的用户表，如果是真实的业务表，可以想象执行时间会有多久。



### 2.4 原因分析

首先来分析一下这句 SQL 执行的过程。

由于 sex 列是索引列，MySQL会走 sex 这棵索引树，命中 sex=1 的数据。


然后又由于普通索引中存储的是主键 id 的值，且查询语句要求查询所有列，所以这里会发生一个**回表**的情况。即在命中 sex 索引树中值为1的数据后，拿着它叶子节点上的值也就是主键 id 去主键索引树上查询这一行其他列（name、sex）的值，最后返回到结果集中，这样第一行数据就查询成功了。


最后这句 SQL 要求 `limit 100, 10`，也就是查询第101到110个数据，但是 MySQL 会**查询前110行，然后将前100行抛弃**，最后结果集中就只剩下了第101到110行，执行结束。


小结一下，在上述的执行过程中，造成 limit 大偏移量执行时间变久的原因有：


- 查询所有列导致回表
- `limit a, b` 会查询前a+b条数据，然后丢弃前a条数据

综合上述两个原因，MySQL 花费了大量时间在回表上，而其中a次回表的结果又不会出现在结果集中，这才导致查询时间变得越来越长。




## 3. 优化方案


### 3.1 覆盖索引


既然无效的回表是导致查询变慢的主要原因，那么优化方案就主要从减少回表次数方面入手，假设在 `limit a, b` 中我们首先得到了 a+1 到 a+b 条数据的id，然后再进行回表获取其他列数据，那么就减少了a次回表操作，速度肯定会快上不少。


这里就涉及到**覆盖索引**了，所谓的覆盖索引就是从普通索引树中就能查到的想要数据，而不需要通过回表从主键索引中查询其他列，能够显著提升性能。


基于这样的思路，优化方案就是先查询得到主键id，然后再根据主键id查询其他列数据，优化后的 SQL 以及执行时间如下表。

| 优化后的 SQL | 执行时间 |
| --- | --- |
| select * from user a join (select id from user where sex = 1 limit 100, 10) b on a.id=b.id; | OK, Time: 0.000000s |
| select * from user a join (select id from user where sex = 1 limit 1000, 10) b on a.id=b.id; | OK, Time: 0.00000s |
| select * from user a join (select id from user where sex = 1 limit 10000, 10) b on a.id=b.id; | OK, Time: 0.002000s |
| select * from user a join (select id from user where sex = 1 limit 100000, 10) b on a.id=b.id; | OK, Time: 0.015000s |
| select * from user a join (select id from user where sex = 1 limit 1000000, 10) b on a.id=b.id; | OK, Time: 0.151000s |
| select * from user a join (select id from user where sex = 1 limit 10000000, 10) b on a.id=b.id; | OK, Time: 1.161000s |

果然，从执行时间上看，执行效率得到了显著提升。



### 3.2 条件过滤


当然还有一种有缺陷的方法是基于排序做条件过滤。


比如像上面的示例 user 表，我要使用 limit 分页得到1000001到1000010条数据，可以这样写 SQL：


```sql
select * from user where sex = 1 and id > (select id from user where sex = 1 limit 1000000, 1) limit 10;
```

使用这样的方式优化是有条件的：主键id必须是有序的。

在有序的条件下，也可以使用比如创建时间等其他字段来代替主键id，但是前提是这个字段是建立了索引的。


总之，使用条件过滤的方式来优化 limit 是有诸多限制的，一般还是推荐使用覆盖索引的方式来优化。



## 4. 小结


本文主要分析了 limit 分页大偏移量执行速度慢的原因，同时也提出了相应的优化方案，推荐使用**覆盖索引**的方式来优化 limit 分页大偏移执行速度慢的问题。


希望能帮助到大家。


最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。

