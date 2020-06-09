---
layout: post
title: H2 数据库使用问题小记——初始化数据库失败
categories: [数据库]
description: H2 数据库使用问题小记——初始化数据库失败
keywords: H2, DataBase
---

## 背景
由于项目需要减少依赖，将数据库由 MySQL 更换为更轻量级的 H2 数据库，在此记录使用 schema.sql 的初始化脚本中出现的问题。

## 源代码及脚本文件
schema.sql 脚本文件，此脚本是由 navicat 导出的 sql 文件
```sql
DROP TABLE IF EXISTS USER;
CREATE TABLE USER (
`user_id` varchar(36) NOT NULL COMMENT '用户主键id',
`user_name` varchar(55) DEFAULT NULL COMMENT '账号',
`password` varchar(55) DEFAULT NULL COMMENT '密码',
`email` varchar(55) DEFAULT NULL COMMENT '邮箱',
`create_time` datetime DEFAULT NULL COMMENT '创建日期',
`update_time` datetime DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
`version` int(255) DEFAULT '0' COMMENT '乐观锁更新的版本号',
PRIMARY KEY (`user_id`) USING BTREE,
UNIQUE KEY `unique_key_email` (`email`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户表';
```

## 异常堆栈
```java
2020-01-27 14:03:34.169 ERROR 9464 --- [           main] o.s.b.web.embedded.tomcat.TomcatStarter  : Error starting Tomcat context. Exception: org.springframework.beans.factory.BeanCreationException. Message: Error creating bean with name 'h2Console' defined in class path resource [org/springframework/boot/autoconfigure/h2/H2ConsoleAutoConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.boot.web.servlet.ServletRegistrationBean]: Factory method 'h2Console' threw exception; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource' defined in class path resource [org/springframework/boot/autoconfigure/jdbc/DataSourceConfiguration$Hikari.class]: Initialization of bean failed; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'org.springframework.boot.autoconfigure.jdbc.DataSourceInitializerInvoker': Invocation of init method failed; nested exception is org.springframework.jdbc.datasource.init.ScriptStatementFailedException: Failed to execute SQL script statement #2 of URL [file:/Users/nanbei/workspace/Windfall/target/classes/schema.sql]: CREATE TABLE USER ( `user_id` varchar(36) NOT NULL COMMENT '用户主键id', `user_name` varchar(55) DEFAULT NULL COMMENT '账号', `password` varchar(55) DEFAULT NULL COMMENT '密码', `email` varchar(55) DEFAULT NULL COMMENT '邮箱', `create_time` datetime DEFAULT NULL COMMENT '创建日期', `update_time` datetime DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间', `version` int(255) DEFAULT '0' COMMENT '乐观锁更新的版本号', PRIMARY KEY (`user_id`) USING BTREE, UNIQUE KEY `unique_key_email` (`email`) USING BTREE ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户表'; nested exception is org.h2.jdbc.JdbcSQLSyntaxErrorException: Syntax error in SQL statement "CREATE TABLE USER ( `USER_ID` VARCHAR(36) NOT NULL COMMENT '用户主键id', `USER_NAME` VARCHAR(55) DEFAULT NULL COMMENT '账号', `PASSWORD` VARCHAR(55) DEFAULT NULL COMMENT '密码', `EMAIL` VARCHAR(55) DEFAULT NULL COMMENT '邮箱', `CREATE_TIME` DATETIME DEFAULT NULL COMMENT '创建日期', `UPDATE_TIME` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间', `VERSION` INT(255) DEFAULT '0' COMMENT '乐观锁更新的版本号', PRIMARY KEY (`USER_ID`) USING[*] BTREE, UNIQUE KEY `UNIQUE_KEY_EMAIL` (`EMAIL`) USING BTREE ) ENGINE=INNODB DEFAULT CHARSET=UTF8 COMMENT='用户表'"; expected "INDEX, ,, )"; SQL statement:
CREATE TABLE USER ( `user_id` varchar(36) NOT NULL COMMENT '用户主键id', `user_name` varchar(55) DEFAULT NULL COMMENT '账号', `password` varchar(55) DEFAULT NULL COMMENT '密码', `email` varchar(55) DEFAULT NULL COMMENT '邮箱', `create_time` datetime DEFAULT NULL COMMENT '创建日期', `update_time` datetime DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间', `version` int(255) DEFAULT '0' COMMENT '乐观锁更新的版本号', PRIMARY KEY (`user_id`) USING BTREE, UNIQUE KEY `unique_key_email` (`email`) USING BTREE ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户表' [42001-200]
2020-01-27 14:03:34.223 ERROR 9464 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Failed to destroy the filter named [Tomcat WebSocket (JSR356) Filter] of type [org.apache.tomcat.websocket.server.WsFilter]
(其余省略)
......
```
由异常堆栈可以看出（`Failed to execute SQL script statement #2 ...`），在执行 schema.sql 脚本时失败，导致程序启动失败。

## 内事不决问百度
秉承内事不决问百度的优良作风，我以“springboot h2 创建表时报错”的关键字进行检索，找到了一些案例。

- [写了一个简单的微服务框架，运行时出现如下错误，及解决办法](https://blog.csdn.net/qq_38254897/article/details/89317833) —— 这篇博文的解决思路是：sql 文件大小写不统一，将脚本改为小写就解决了问题。亲测无效。

- [[Java] h2数据库初始化表失败问题解决记录](https://blog.csdn.net/petrel2015/article/details/81784288) —— 这篇博文倒是很有用，感谢原博，原来是因为在我的 sql 脚本中有一些 MySQL 数据库中特有的配置例如：InnoDB。

至此，我明白了本问题的解决思路：删除 schema.sql 脚本中存在的 MySQL 数据库独有的关键字。

## 解决方案
1. 首先我删除了脚本中的 `ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户表'` 语句，重启程序后，还是会报脚本执行失败的错误。由此可见在脚本中还是存在 H2 无法识别的内容，但是此语句还是需要删除，因为 InnoDB 确实不是 H2 数据库可以识别的内容。

2. 然后我使用排除法，将目标锁定在了脚本中的最后两行——索引相关的语句。最后得出结论—— `PRIMARY KEY (`user_id`) USING BTREE,` 本行语句需要删除 `USING BTREE` 然后执行才能够执行成功，可能是因为在 H2 数据库中主键索引无法使用 B+ 树吧。

最后能够成功执行的 sql 脚本：
```sql
DROP TABLE IF EXISTS USER;
CREATE TABLE USER (
  `user_id` varchar(36) NOT NULL COMMENT '用户主键id',
  `user_name` varchar(55) DEFAULT NULL COMMENT '账号',
  `password` varchar(55) DEFAULT NULL COMMENT '密码',
  `email` varchar(55) DEFAULT NULL COMMENT '邮箱',
  `create_time` datetime DEFAULT NULL COMMENT '创建日期',
  `update_time` datetime DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `version` int(255) DEFAULT '0' COMMENT '乐观锁更新的版本号',
  PRIMARY KEY (`user_id`),
  UNIQUE KEY `unique_key_email` (`email`) USING BTREE
);
```
> 版权声明：本文为[Planeswalker23](https://github.com/Planeswalker23)所创，转载请带上原文链接，感谢。<br>
> 本文已上传个人公众号，欢迎扫码关注。

![wechat](https://planeswalker23.github.io/images/wechat.png)