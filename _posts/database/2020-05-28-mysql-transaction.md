---
layout: post
title: MyBatis 查询结果与 MySQL 执行结果不一致？
categories: [数据库]
description: MyBatis 查询结果与 MySQL 执行结果不一致？
keywords: MySQL, MyBatis, 事务
---

## 1. 碎碎念
最近在业务中遇到一个问题，业务是这样的：在插入新用户时需要校验用户的某些信息是否唯一，而在程序中校验结果永远是不唯一的。然后我把 MyBatis 打印的执行 SQL 语句拿了出来在数据库中执行，发现没有数据。

然后我就奇怪了，数据库是同一个啊、SQL 是同一个啊、查询结果都没有变啊，为什么执行的结果在程序里面是 1，而在数据库中是0。

难道是因为 MyBatis 和数据库执行的结果不一样？

![2020052801](https://planeswalker23.github.io/images/posts/2020052801.png)

后来我才明白不一致的原因。

我编写了一个与实际业务类似的代码，用来模拟上述的问题。

## 2. 复现问题
### 2.1. 表结构
MySQL 数据库中创建了一张用户表，只有4个字段。
```sql
CREATE TABLE `user`  (
  `user_id` varchar(36) NOT NULL COMMENT '用户主键id',
  `user_name` varchar(55) NULL DEFAULT NULL COMMENT '账号',
  `password` varchar(55) NULL DEFAULT NULL COMMENT '密码',
  `email` varchar(55) NULL DEFAULT NULL COMMENT '邮箱',
  PRIMARY KEY (`user_id`) USING BTREE
);
```

### 2.2. 项目依赖
示例项目是一个 SpringBoot 工程，pom 文件中除了 web 依赖还有 mysql 的驱动、MyBatis 和 lombok。

```java
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>2.2.6.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.47</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.8</version>
    </dependency>
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>1.3.0</version>
    </dependency>
</dependencies>
```

### 2.2. 业务
业务流程是这样：新建一个用户，在新建用户之前首先校验邮箱是否已在数据库中存在，然后执行一些其他的业务，然后执行 insert 方法插入数据库，然后执行一些其他的业务，最后再校验 user_name 是否已存在。

```java
@Slf4j
@RestController
public class TestController {

    @Resource
    private UserMapper userMapper;

    /**
     * springboot Dao层查询结果与数据库实际执行结果不一致？
     */
    @GetMapping("test")
    @Transactional(rollbackFor = RuntimeException.class)
    public void transactionalDemo() {
        // 要插入的数据
        User user = new User();
        user.setUserId("userId");
        user.setUserName("planeswalker");
        user.setPassword("password");
        user.setEmail("123@gmail.com");
        // 校验邮箱
        if (userMapper.countByEmail(user.getEmail())>0) {
            throw new RuntimeException("插入失败，user_id 重复");
        }
        // 执行插入用户操作
        userMapper.insert(user);
        // 校验 user_name
        if (userMapper.countByName(user.getUserName())>0) {
            throw new RuntimeException("插入失败，user_name 重复");
        }
        log.info("do something others...");
    }
}
```

userMapper 接口类的代码如下:

```java
@Repository
public interface UserMapper {

    /**
     * 查询 email 是否重复
     * @param email
     * @return
     */
    @Select("select count(*) from user where email=#{email}")
    int countByEmail(String email);

    /**
     * 查询 name 是否重复
     * @param userName
     * @return
     */
    @Select("select count(*) from user where user_name=#{userName}")
    int countByName(String userName);
}
```

我承认这个方法确实可能不是特别好，比如校验重复的方法为什么有两次，比如 user_id 校验重复方法的合理性。但为了与我在项目中遇到的问题做模拟，这是很类似的，在项目中就是在插入后又校验了一次（因为公用的校验方法会查询两张表而其中一张表的数据是在校验之前插入的）。

可能很多同学已经知道这个业务的问题所在了，先不多说，执行就行了。

### 2.3. 测试
当我在浏览器上访问这个接口`http://127.0.0.1:8080/test`后，控制台输出了如下的内容：

```java
2020-05-27 14:07:09.183 DEBUG 18375 --- [nio-8080-exec-6] c.b.d.m.i.a.m.UserMapper.countByEmail    : ==>  Preparing: select count(*) from user where email=? 
2020-05-27 14:07:09.208 DEBUG 18375 --- [nio-8080-exec-6] c.b.d.m.i.a.m.UserMapper.countByEmail    : ==> Parameters: 123@gmail.com(String)
2020-05-27 14:07:09.218 DEBUG 18375 --- [nio-8080-exec-6] c.b.d.m.i.a.m.UserMapper.countByEmail    : <==      Total: 1
2020-05-27 14:07:09.233 DEBUG 18375 --- [nio-8080-exec-6] c.b.d.m.i.a.mapper.UserMapper.insert     : ==>  Preparing: INSERT INTO user ( user_id,user_name,password,email ) VALUES( ?,?,?,? ) 
2020-05-27 14:07:09.234 DEBUG 18375 --- [nio-8080-exec-6] c.b.d.m.i.a.mapper.UserMapper.insert     : ==> Parameters: userId(String), planeswalker(String), password(String), 123@gmail.com(String)
2020-05-27 14:07:09.237 DEBUG 18375 --- [nio-8080-exec-6] c.b.d.m.i.a.mapper.UserMapper.insert     : <==    Updates: 1
2020-05-27 14:07:09.237 DEBUG 18375 --- [nio-8080-exec-6] c.b.d.m.i.a.m.UserMapper.countByName     : ==>  Preparing: select count(*) from user where user_name=? 
2020-05-27 14:07:09.237 DEBUG 18375 --- [nio-8080-exec-6] c.b.d.m.i.a.m.UserMapper.countByName     : ==> Parameters: planeswalker(String)
2020-05-27 14:07:09.238 DEBUG 18375 --- [nio-8080-exec-6] c.b.d.m.i.a.m.UserMapper.countByName     : <==      Total: 1
2020-05-27 14:07:09.250 ERROR 18375 --- [nio-8080-exec-6] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.RuntimeException: 插入失败，user_name 重复] with root cause

java.lang.RuntimeException: 插入失败，user_name 重复
	at com.biosan.databasehandler.TestController.transactionalDemo(TestController.java:43) ~[classes/:na]
	at com.biosan.databasehandler.TestController$$FastClassBySpringCGLIB$$dadcb476.invoke(<generated>) ~[classes/:na]
    ......
```

在第二个校验方法的时候抛出了错误，说明数据库中存在相同 user_name 的数据，然后我又把 SQL 拿出来单独去数据库中执行，发现没有数据！

不信邪的我又在第二个校验方法上打了端点，当程序执行到此处时，它的执行结果是：

![2020052802.png](https://planeswalker23.github.io/images/posts/2020052802.png)

也就是说确实这时候存在这样的数据！

而此时我又在数据库当中查询，竟然也查不到这条数据！

这就让我开始考虑到，可能不是代码或者框架的原因，而是其他的问题了，比如数据库事务。

### 2.4. 原因
我们知道在 SpringBoot 的接口上标注了 `@Transactional`注解，就相当于开启了一个事务。

MySQL 默认的事务隔离级别是读已提交，即一个事务提交之后，它做的变更才会被其他事务看到。而在同一个事务中，如果先插入后查询，如果查询条件符合，是可以查询到插入的数据的。

当我的程序在执行完 insert 方法后，又去根据 user_name 查询，就可以查询到插入的数据，而此时我直接在数据库中查询该 user_name，相当于又开启了一个事务进行查询，由于读已提交的隔离级别，一个事务提交之后，它做的变更才会被其他事务看到，且业务方法未提交，所以在数据库中查询不到数据。

这也就是我在程序中和数据库中用同样的 SQL 进行查询，但查询结果却不相同的原因。

### 2.5. 修复
这个问题从业务上来说原本就是不合理的，我在查询重复数据时本就应该排除与将要插入数据相同 id 的数据，即 SQL 应该是：

```sql
select count(*) from user where user_name='planeswalker' and user_id!='userId'
```

同时，验证重复的业务逻辑应该在插入语句之前...

当然这就是后话了。

## 3. 小结
本文记录了一个关于 SpringBoot+MyBatis 框架查询数据库的小问题，这其实是数据库事务与隔离级别的问题，同时这也是一个业务上的问题，应该在插入之前进行验重。

关于数据库隔离级别，这里只是小小提了一下，以后有空再总结吧。