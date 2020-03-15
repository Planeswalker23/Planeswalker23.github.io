---
layout: post
title: MySQL 中 count() 函数的正确使用方法
categories: [数据库]
description: MySQL 中 count() 函数的正确使用方法
keywords: MySQL, count
---

# 背景
在 MySQL 中，当我们需要获取表中的行数时，一般会选择使用`select count(*) from table;`语句，其实 count() 函数中除了 * 还可以放其他参数，比如1、主键id、字段，那么它们有什么区别呢？效率如何呢？我们应该使用哪种方式来获取表的行数呢？当搞清楚 count() 函数的运行原理后，相信上面两个问题就会有了答案。

# 表结构
> user 表有两个字段，主键id和name，name可为空。在该表中共有1000000条数据，前100000条数据行的name字段为空，其余数据行name=id。
```
-- 建表语句
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(255) DEFAULT NULL COMMENT '姓名',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8;

-- 使用存储过程造测试数据
delimiter;;
create procedure idata()
begin 
  declare i int; 
  set i=1; 
  while(i<=1000000)do 
    insert into user values(i, i);
    set i=i+1; 
  end while;
end;;
delimiter;
call idata();

-- 将前100000条数据的name字段置为null
update user set name=null where id<100000;
```

# 测试 SQL 语句
为了区分 count() 函数不同参数的区别，主要从执行时间和扫描行数这两方面来描述 sql 的执行效率，同时从返回结果来描述 count() 函数的特性。

- count(*)—— `select count(*) from user;`
- count(1)—— `select count(1) from user;`
- count(id)(即主键)—— `select count(id) from user;`
- count(name)(即字段)—— `select count(name) from user;`

# 测试结果
1. `select count(*) from user;`
  - 返回结果：1000000
  - 执行时间：0.157秒
  - 执行描述：遍历全表，不取值（优化后，必定不是null，不取值），累加计数，最终返回结果。
2. `select count(1) from user;`
  - 返回结果：1000000
  - 执行时间：0.171秒
  - 执行描述：遍历全表，一行行取数据，将每一行赋值为1，判断到该字段不可为空，累加计数，最终返回结果。
3. `select count(id) from user;`
  - 返回结果：1000000
  - 执行时间：0.183秒
  - 执行描述：遍历全表，一行行取数据（会选择最小的索引树来遍历，所以比相同情况下的count字段效率更高），取每行的主键id，判断到该字段不可为空，累加计数，最终返回结果。
4. `select count(name) from user;`
  - 返回结果：900001
  - 执行时间：0.203秒
  - 执行描述：
    - 若字段定义不为空：遍历全表，一行行取数据，取每行的该字段，判断到该字段不可为空，累加计数，最终返回结果。
    - 若字段定义可为空：遍历全表，一行行取数据，取每行的该字段，判断到该字段可能是null，然后再判断该字段的值是否为null，不为null才累加计数，最终返回结果。
    - 若该字段没有索引，将遍历主键索引树。

## 结果集
首先从结果集的角度来看，前三条 SQL 语句的目的是一样的——返回的是所有行数，而 count 函数的参数是普通字段且字段默认为 null 的时候，它返回的是该字段不为 null 的行数。

## 执行时间
从执行时间上来看的话，效率大致是 count(字段) < count(主键 id) < count(1) < count(*)，当然由于本次测试表结构简单且数据集只有100w条，所以这个执行时间并不能成为执行效率的有力证据，但从中还是能看出一些端倪的，即大差不差。

# 总结
count() 是一个聚合函数，对于返回的结果集，一行行地判断，如果 count 函数的参数不是 NULL，累计值就加 1，否则不加。最后返回累计值。

- count(*) 速度最快的原因是它不会在计数的时候去取每行数据值
- count(1）比 count(*) 稍慢的原因是它会取每个数据行并赋值为1
- count(主键id) 比 count(1) 稍慢的原因是它会从每个数据行中取出主键 id
- count(字段) 最慢的原因是它可能需要判断每个数据行中的改字段是否为 null

所以，最好还是用 count(*) .
