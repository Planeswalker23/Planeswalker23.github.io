---
layout: post
title: 我所理解的其他问题·第2篇·基于Mybatis拦截器实现关键信息加密
categories: [MyBatis]
keywords: MyBatis, 拦截器
---



## 1. 开篇词

先来看一条sql
```mysql
select * from user where mobile='10086';
```
相信这很容易理解，这条sql的意思是查询数据库中手机号为10086的所有用户的信息。

这里涉及到一个关键信息加密的问题，假设发生了一个最坏的情况，项目的数据库被盗，所有用户的数据被打包，如何保证在这样的情况下用户关键信息不被泄露？

很简单，我们只要在数据被写入数据库的时候将用户的隐私信息进行加密，这样就算数据库整个都泄露了，只要加密方式没有泄露，那么用户的隐私就是安全的。

对用户隐私信息的加密，应该是属于每步写入数据库操作都需要做的，同样对于查询结果的解密及修饰，也是必须的。那么在这里就遇到一个问题：在日常开发时，我们如何避开这种公共的操作，提高开发效率？



## 2. 解决方案

想要解决这个问题，就必须知道在项目中与数据库的交互过程。由于项目是使用mybatis来维护持久层的，我们就先来看一个基于mybatis的查询方法是如何执行的，在整个查询方法执行的过程中，我们或许能得到一些启示。

下面就是具体的代码示例。

```java
User user = new User();
user.setMobile("10086");

List<User> users = userMapper.select(user);
```

当代码执行到`userMapper.select(user)`方法时，接下来它会怎么运行？通过debug跟踪，可以看到代码进行到了`MapperProxy`类中，它根据传入的`Method`对象返回一个`MapperMethod`对象（缓存中若存在，直接返回，若不存在new一个，然后将创建的对象放入缓存），然后调用此对象的`excute`方法执行具体的命令。

在`MapperMethod`类中维护着两个属性，`SqlCommand`和`MethodSignature`，前者存储着本次执行方法的sql类型和id（StatementId，每个sql在mybatis中的唯一id，mapper全路径类名+方法名），后者存储着本次执行方法的参数和返回类型。

然后根据`SqlCommand`属性中的sql类型，去调用`sqlSession`对象的不同方法，在这里执行的是`selectList`方法，它会根据传入的statementId参数从配置对象中获取`MappedStatement`对象（对应着在mapper.xml文件中写的sql节点），然后将任务委托给Executor对象去执行。

在Executor对象执行具体的sql逻辑代码中，首先是根据传入的参数动态生成需要执行的sql语句，它被维护在一个BoundSql对象中。

经过一系列的动态生成sql语句操作（分析xml节点，解析mybatis的xml文件select方法中的if、trim等判断逻辑，不是本文的重点），最终找到了将参数设置进sql的地方：

```java
@Override
public void parameterize(Statement statement) throws SQLException {
    parameterHandler.setParameters((PreparedStatement) statement);
}
```

在`PreparedStatementHandler`类的`parameterize`方法中，执行将参数设置入sql的逻辑，此方法执行了持有的`ParameterHandler`对象的`setParameters`方法，而这个方法的参数类型是`PreparedStatement`类。

分析到这里，本文最初的那个问题就可以得到解决了：如何在日常开发时，避开这种公共的操作，提高开发效率？



## 3. 源码分析

如果`setParameters`方法前，执行数据库实体类属性的加密逻辑，那么在业务中就不需要额外去加密了。那么这个功能如何实现呢？

在跟踪mybatis源码的时候，发现这样一段代码：

```java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
	// 循环所有的interceptor拦截器
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }
```

这段代码通过传入的参数生成一个`StatementHandler`对象，这个对象是负责设置查询参数、处理JDBC返回的resultSet加工为集合返回的。然后这个`StatementHandler`对象经过了所有的`Interceptor`对象加工之后再返回。我们知道`Interceptor`是拦截器，而在mybatis中，有自己实现的拦截器: `org.apache.ibatis.plugin.Interceptor`。

这下思路就很明确了，我们只需要创建一个实现`Interceptor`接口的拦截器，这个拦截器需要拦截的内容是`setParameters`设置参数方法，在拦截器中进行加密操作，就省去了日常开发中那些繁琐的公共操作（如每个插入对象的隐私信息都要加密、公共字段的设置等）

mybatis拦截器接口有三个方法，setProperties方法是给拦截器设置前置参数的，plugin方法是判断该拦截器是否需要生成代理，防止自定义拦截器未指明类型或拦截点（mybatis拦截器仅支持四个类的拦截：Executor、ParameterHandler、StatementHandler、ResultSetHandler），intercept是拦截器的拦截逻辑。
```java
public interface Interceptor {
  Object intercept(Invocation invocation) throws Throwable;
  Object plugin(Object target);
  void setProperties(Properties properties);
}
```

为什么mybatis仅支持上述四个类的拦截？下面的代码是mybatis一次查询需要经过的拦截器遍历：
```java
public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
```
找出所有使用到这个方法的地方，发现只有在`Configuration`类的四个地方用到：即生成上述四个类的方法中。



## 4. 实现自定义拦截器

下面我们来实现一个自定义的加密拦截器：
```java
@Component
@Intercepts({
        @Signature(
                type = ParameterHandler.class,
                method = "setParameters",
                args = PreparedStatement.class)
        })
public class EncryptInterceptor implements Interceptor {
	@Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 获得拦截的类
        ParameterHandler parameterHandler = (ParameterHandler) invocation.getTarget();
        // 反射获取 ParameterHandler 对象的 parameterObject 属性，即传入的参数，同时设置访问权限
        Field parameterObjectField = parameterHandler.getClass().getDeclaredField("parameterObject");
        parameterObjectField.setAccessible(true);
        // 此对象为为sql设置字段值的对象，即parameterType中声明的对象
        Object paramObject = parameterObjectField.get(parameterHandler);
        // paramObject分为list和单个实体对象执行
        if (paramObject instanceof List) {
            // 将paramObject对象使用list指代
            List list = (List) paramObject;
            if (!list.isEmpty()) {
                // list.forEach((res)->加密逻辑);
            }
        } else {
            //加密逻辑
        }
		// 返回结果
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
}
```

创建`EncryptInterceptor`类，实现`Interceptor`接口，然后添加拦截注解。mybatis拦截器需要配置`@Intercepts`和`@Signature`两个注解来指定拦截点，还记得本文前面说到的`setParameters`方法吗，就是我们需要拦截的方法，而在`@Signature`注解中指定拦截点是，type就是`setParameters`方法所属的类，method就是`setParameters`方法，args就是此方法传入的参数。

在上述代码的`intercept`方法内，我们简单实现了加密逻辑。这样一个基于mybatis的加密拦截器就实现了。

加密完成了，解密还难吗？

mybatis拦截器除了拦截设置参数和拦截返回查询结果实现加密、解密或者公共字段的自动填充之外，还有许多用处，比如说可以用它实现分页、慢sql熔断等等。拦截器是一个强大的工具，但是要注意如果在拦截器中添加太多逻辑，可能会影响业务效率，拦截器的效率问题也是一个要注意的点。

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。