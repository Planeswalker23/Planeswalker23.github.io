---
layout: post
title: MySQL 中 update 语句踩坑日记
categories: [数据库]
description: MySQL 中 update 语句踩坑日记
keywords: MySQL, update
---

# 背景
最近在一次线上作业过程中执行了一句 DML 语句，本以为万无一失，结果应用反馈说没有更新，数据还是没有变，最后经过排查才发现是我语句写错了，导致 update 语句执行的结果与预期不符。

# 情景再现
为了方便演示，建立一张用户表，同时插入五条数据。
````mysql
create table user(
id int(12) COMMENT '用户主键id',
name varchar(36) COMMENT '用户名',
age int(12) comment '年龄');

insert into user values (1,'one',11),(2,'two',12),(3,'three',13),(4,'four',15),(5,'five',15);
````

执行完成后，现在 user 表中的数据如下: 
````mysql
id  name    age
--------------
1	one  	11
2	two 	12
3	three	13
4	four	15
5	five	15
````

现在需要把所有的年龄改成 10、用户名改成 user（假设此操作有意义）,我提交的 DML 语句如下:
````mysql
update user set age=10 and name='user';
````

当我刷新用户表，看到执行 update 语句后的表全部数据如下:
````mysql
id  name    age
--------------
1	one	    0
2	two	    0
3	three	0
4	four	0
5	five	0
````

神奇的事情发生了，age 全部被更新成0，而 name 字段竟然没有任何修改！

# 错误原因及修正
错误原因其实很简单，update 语句写错了。MySQL 中 update 语句的语法应该是`update user set age=10, name='user';`，如果更新多个字段，相邻字段间应该以逗号分隔而不是 and。

如果 update 语句使用 and 作为多个字段之间的分隔符，就像最开始那样——`update user set age=10 and name='user';`，这个更新语句将会变成`update user set age=(10 and name='user');`。

而`(10 and name='user')`作为一个返回值为 boolean 类型的判断语句，返回会被映射成 1 或 0，有 99% 的可能会让第一个更新变量更新为错误的数据。

# 教训
1. 在提交 DML 语句前现在测试环境试一下
2. 基础的 SQL 语法不要记错