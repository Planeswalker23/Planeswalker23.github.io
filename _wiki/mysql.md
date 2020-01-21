---
layout: wiki
title: MySQL 相关的常用命令
categories: [MySQL]
description: MySQL 相关的常用命令
keywords: MySQL
---
## 查看表结构
查看表结构命令：`desc 表名`
```
mysql> use recommend;
Database changed

mysql> desc user;
+--------------+--------------+------+-----+---------+-------+
| Field        | Type         | Null | Key | Default | Extra |
+--------------+--------------+------+-----+---------+-------+
| id           | bigint(20)   | NO   | PRI | NULL    |       |
| username     | varchar(20)  | NO   |     | NULL    |       |
| password     | varchar(20)  | NO   |     | NULL    |       |
| hometown     | varchar(20)  | YES  |     | NULL    |       |
| gender       | varchar(20)  | YES  |     | NULL    |       |
| birthday     | varchar(20)  | YES  |     | NULL    |       |
| email        | varchar(20)  | YES  |     | NULL    |       |
| phone_number | varchar(20)  | YES  |     | NULL    |       |
| profession   | varchar(20)  | YES  |     | NULL    |       |
| hobby        | varchar(100) | YES  |     | NULL    |       |
+--------------+--------------+------+-----+---------+-------+
10 rows in set (0.00 sec)
```

## 查看建表语句
查看建表语句命令：`show create table 表名`
```
mysql> show create table user;
+-------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
+-------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| user  | CREATE TABLE `user` (
  `id` bigint(20) NOT NULL,
  `username` varchar(20) NOT NULL,
  `password` varchar(20) NOT NULL,
  `hometown` varchar(20) DEFAULT NULL,
  `gender` varchar(20) DEFAULT NULL,
  `birthday` varchar(20) DEFAULT NULL,
  `email` varchar(20) DEFAULT NULL,
  `phone_number` varchar(20) DEFAULT NULL,
  `profession` varchar(20) DEFAULT NULL,
  `hobby` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 |
+-------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```
注：数据库名->`recommend`；表名->`user`

## 查看默认端口号
查看 MySQL 默认端口号：`show global variables like 'port';`

```
mysql> show global variables like 'port';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3306  |
+---------------+-------+
1 row in set (0.00 sec)

## Mac 导出 SQL 文件
首先`cd`到你所想要将`sql`文件导出的位置，例如：
```
cd /Users/nanbei/Desktop
```
注：nanbei为我的ID。

然后输入：
```
mysqldump -u root -p 数据库名 [表名] > 生成文件名;
```
加表名表示导出表，不加表名表示导出数据库。

再输入密码即可完成导出，完整命令如下：

```
 cd /Users/nanbei/Desktop
 mysql -u root -p test user > user.sql;
```

## Mac 导入 SQL 文件
```
mysql -u root -p
use test;
source /Users/nanbei/Desktop/user.sql;
```
`source`后面的内容可以直接将`.sql`文件直接拖拽至终端，自动补全其文件目录。
在导入的过程中可能出现如下错误：
```
ASCII '\0' appeared in the statement, but this is not allowed 
unless option --binary-mode is enabled and mysql is run in non-
interactive mode. Set --binary-mode to 1 if ASCII '\0' is 
expected. Query: '?-'.
```
其原因是想要导入的`.sql·`文件是在`powershell`里写命令导出的。

解决方法是在`cmd`中重新导出`.sql`文件再导入。