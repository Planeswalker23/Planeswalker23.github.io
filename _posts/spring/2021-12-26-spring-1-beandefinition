---
layout: post
title: 第1篇·Spring 核心元信息——BeanDefinition
categories: [Spring]
keywords: Spring, BeanDefinition
---



## 1. 开篇词

Spring 已经是我辈 Java 开发者日常工作中最不可不懂的核心框架了，没有之一。

同时 Spring 也是面试博弈中绕不过去的一座大山，凡必问 IOC、AOP、源码、设计模式等等。毫不夸张的说，只要把 Spring 的几个核心问题搞清楚搞明白了，就算其他问题回答的并不尽如人意，也会被面试官标记上“对 Spring 有很好的理解”这种标签而从众多候选人中脱颖而出，最终升官加爵，封侯拜相。

Spring 一直是我想整理的一个框架，但限于自身水平的原因迟迟没有动笔。年关将近，总有一种愁绪涌上心头，所以才能狠下决心来开始动笔。Spring 系列目前只有下面几个主题是确定的：

- Spring BeanDefinition
- Spring Bean 作用域
- Spring Bean 生命周期
- 依赖查找
- 依赖注入
- 循环依赖问题
- AOP 原理
- AOP 失效场景及解决方案
- ...

后续也会慢慢补充其他方面的东西，比如 Spring 事件、自动装配、条件化配置等。

这篇文章的主要议题是 BeanDefinition。



## 2. BeanDefinition 是什么

BeanDefinition 是 Spring Framework 中最为核心的一个概念，它描述的是 Bean 定义的元信息。如果将 Spring 类比为 Java 语言的话，那么 BeanDefinition 就相当于是 Class 类，而 Bean 就相当于是一个个对象。

在 Spring Framework 5.3.5 版本的注释是这么描述 BeanDefinition 的：

> A BeanDefinition describes a bean instance, which has property values, constructor argument values, and further information supplied by concrete implementations.
> This is just a minimal interface: The main intention is to allow a BeanFactoryPostProcessor to introspect and modify property values and other bean metadata.

BeanDefinition 描述了一个 Bean 实例，该实例具有属性值、构造器参数值以及由具体实现提供的进一步信息。这只是一个最小的接口：主要目的是允许 BeanFactoryPostProcessor 内省（运行时类型检查）和修改属性值和其他 Bean 元数据。

关于 BeanFactoryPostProcessor 的内容在 Spring 系列的后续内容中将有阐述，现在我们只需要知道基于这个接口我们可以对 BeanDefinition 进行修改就可以。



## 3. BeanDefinition 源码

既然 BeanDefinition 是 Spring 对 Bean 的抽象定义，那么它到底有哪些属性呢？下面我们就来揭晓。

首先需要开宗明义的是：BeanDefinition 是一个接口。它继承了 AttributeAccessor、BeanMetadataElement 这两个接口。

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement { }
```

### 3.1 AttributeAccessor

AttributeAccessor 接口定义了用于向任意对象设置和访问元数据的通用方法。

```java
public interface AttributeAccessor {
  // 将对象的 name 属性设置为 value 值
	void setAttribute(String name, @Nullable Object value);

	// 获取对象的 name 属性的值
  @Nullable
	Object getAttribute(String name);

	// 获取对象的 name 属性的值，若该值为 null，则通过传入的 Function 为它设置一个新值
	<T> T computeAttribute(String name, Function<String, T> computeFunction);

  // 删除对象的 name 属性的值并返回这个值，若该属性不存在则返回 null
	@Nullable
	Object removeAttribute(String name);
  
  // 判断 name 属性在对象中是否有定义
	boolean hasAttribute(String name);

  // 返回对象的所有属性 name 数组
	String[] attributeNames();
}
```

### 3.2 BeanMetadataElement

BeanMetadataElement 接口提供了一个 getResource() 方法，用来传输一个可配置的资源对象。

```java
public interface BeanMetadataElement {
  // 返回此元数据元素的配置源对象(可以为空)
	default Object getSource() {
		return null;
	}
}
```

### 3.2 BeanDefinition 核心属性

除了以上两个接口外，BeanDefinition 自身还定义了很多方法，首先是可修改的属性：

- setParentName / getParentName：Bean 的父类的名称
- setBeanClassName / getBeanClassName：Bean 的**全限定路径名**
- **setScope / getScope**：Bean 的作用域，默认为 Singleton
- **setLazyInit / isLazyInit**：Bean 是否开启懒加载。懒加载仅适用于单例 Bean，若为 false 则 Bean 在 Spring 容器启动时就会加载，若为 true 则 Bean 在使用时才会进行实例化
- setDependsOn / getDependsOn：Bean 依赖的 Bean 名称，Spring 容器在初始化当前 Bean 时将优先初始化它所依赖的 Bean
- setAutowireCandidate / isAutowireCandidate：Bean 是否是其他 Bean 进行依赖注入的候选对象
- **setPrimary / isPrimary**：Bean 是否是主要（优先级最高）的候选对象
- setFactoryBeanName / getFactoryBeanName：创建Bean 的工厂全限定路径名
- setFactoryBeanMethodName / getFactoryBeanMethodName：创建Bean 的工厂方法名
- getConstructorArgumentValues / hasConstructorArgumentValues：Bean 的构造函数参数值
- getPropertyValues / hasPropertyValues：Bean 实例化需要用到的属性值
- **setInitMethodName / getInitMethodName**：Bean 的初始化方法
- **setDestroyMethodName / getDestroyMethodName**：Bean 的销毁方法
- setRole / getRole：Bean 的角色属性，有三种角色
  - ROLE_APPLICATION：表明该 BeanDefinition 是应用程序的主要部分，通常对应于用户定义的 Bean
  - ROLE_SUPPORT：表明该 BeanDefinition 是一些较大配置的支持部分，通常是从配置文件中生成的 Bean
  - ROLE_INFRASTRUCTURE：表明 BeanDefinition 是一个完全的后台角色，通常是 Spring 内置的 Bean，与用户（开发者）无关
- setDescription / getDescription：Bean 对应的描述性信息

还有一些只读属性：

- getResolvableType：根据 Bean 类信息或其他特定元信息，返回此 BeanDefinition 的可解析类型
- **isSingleton**：Bean 作用域是否是单例的
- **isPrototype**：Bean 作用域是否是原型的
- **isAbstract**：Bean 是否是抽象类，即不能被实例化的
- getResourceDescription：Bean 定义的资源的描述
- getOriginatingBeanDefinition：返回原始的 BeanDefinition，如果没有则返回 null



## 4. BeanDefinition 示例及应用场景

基于上面的内容，我们已经知道 BeanDefinition 是什么，以及 BeanDefinition 源码中描述了什么信息。那么在 Spring 中是如何使用 BeanDefinition 的呢？

事实上，在开发者启动 Spring 容器后（通常是通过 org.springframework.context.support.AbstractApplicationContext#refresh 方法来启动 Spring 容器），Spring 容器会解析配置（可以是 XML 配置文件，也可以是注解驱动的配置类），将开发者自定义的需要 Spring 管理的类都基于 BeanDefinition 包装起来，然后注册到 Spring 容器中，最后在 Spring 容器初始化过程中就会进行 Bean 的初始化。

我们来举一个例子，首先定义一个普通的 Bean，它只有一个 name 属性。

```java
public class Bean {

    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Bean{" +
                "name='" + name + '\'' +
                '}';
    }
}
```

然后我们编写一些测试代码来通过 BeanDefinition 把这个 Bean 注册到 Spring 容器中。

```java
public void register() {
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
    // 定义 BeanDefinition
    BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(Bean.class);
    beanDefinitionBuilder.addPropertyValue("name", "Planeswalker23");
    BeanDefinition beanDefinition = beanDefinitionBuilder.getBeanDefinition();
    // 注册 BeanDefinition
    applicationContext.registerBeanDefinition("bean", beanDefinition);
    // 启动 Spring 容器
    applicationContext.refresh();

    Bean bean1 = applicationContext.getBean(Bean.class);
    Bean bean2 = applicationContext.getBean("bean", Bean.class);
    Assert.assertEquals(bean1, bean2);

    BeanDefinition beanDefinitionFromSpring = applicationContext.getBeanDefinition("bean");
    Assert.assertEquals(beanDefinition, beanDefinitionFromSpring);

    // 关闭 Spring 容器
    applicationContext.close();
}
```

我们通过 BeanDefinitionBuilder.genericBeanDefinition(Bean.class) 来构建一个 BeanDefinitionBuilder，这使用到的是建造者模式。随后往 PropertyValue 中设置属性键值对，最后通过 org.springframework.context.support.GenericApplicationContext#registerBeanDefinition 就成功向 Spring 容器注册了 Bean 类的 BeanDefinition 对象。

在容器启动后通过依赖查找也验证了 Bean 对象成功被 Spring 容器管理。

或许有的同学也会发现了，通过 BeanDefinition 来定义 Bean 跟通过 XML 定义 Bean 其实是差不多的，如果我们通过 XML 来定义 Bean 的话，应该这么写：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="bean" class="io.walkers.planes.pandora.spring.ioc.bean.definition.Bean">
        <property name="name" value="PlanesWalker23"/>
    </bean>

</beans>
```

跟上一步示例的代码是同样的效果，只是 Spring 容器这时候就应该是 ClassPathXmlApplicationContext 类。

那么这时候可能有的同学会问了，我们为什么要手动通过创建 BeanDefinition  实例对象来注册 Bean 呢？这样不是硬编码了吗，到时候有需求变更改一下也会很麻烦。确实，通过创建 BeanDefinition 来手动注册 Bean 的行为我们很少干，但是这个示例代码只是为了向大家介绍 BeanDefinition 的作用，并不是实际的应用场景。

事实上，上文中也提到了，我们在基于 XML 编写配置文件后，Spring 容器会有一步解析配置文件的动作。解析完了之后就会包装为 BeanDefinition 对象，然后将 BeanDefinition 对象注册到 Spring 容器中，这才是 BeanDefinition 的实际应用场景。源代码可以参见：org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.support.DefaultListableBeanFactory)



## 5. 基于 BeanFactoryPostProcessor 修改 BeanDefinition 元信息

在上文中提到过 BeanDefinition 的代码注释中有这样一段话：“（BeanDefinition）允许 BeanFactoryPostProcessor 内省和修改属性值和其他 Bean 元数据”。接下来就来看看我们如何根据 BeanFactoryPostProcessor 在运行时修改 BeanDefinition。

继续沿用上面的例子，我们新增一个实现 BeanFactoryPostProcessor 接口的类。

```java
public class BeanModifyBeanFactoryPostProcessor implements BeanFactoryPostProcessor{
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        BeanDefinition beanDefinition = beanFactory.getBeanDefinition("bean");
        // 将 Bean 作用域改为原型
        beanDefinition.setScope(BeanDefinition.SCOPE_PROTOTYPE);
    }
}
```

通过 ConfigurableListableBeanFactory 获取 BeanDefinition 实例，然后修改它的作用域为原型，然后写一段测试代码。

```java
public void testBeanFactoryPostProcessor() {
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
    // 定义 BeanDefinition
    BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(Bean.class);
    beanDefinitionBuilder.addPropertyValue("name", "Planeswalker23");
    BeanDefinition beanDefinition = beanDefinitionBuilder.getBeanDefinition();
    // 注册 BeanDefinition
    applicationContext.registerBeanDefinition("bean", beanDefinition);
    // 注册 BeanFactoryPostProcessor
    applicationContext.addBeanFactoryPostProcessor(new BeanModifyBeanFactoryPostProcessor());

    // 启动 Spring 容器
    applicationContext.refresh();

    Bean bean1 = applicationContext.getBean(Bean.class);
    Bean bean2 = applicationContext.getBean("bean", Bean.class);
    Assert.assertNotSame(bean1, bean2);

    // 关闭 Spring 容器
    applicationContext.close();
}
```

需要注意的是这里需要通过 org.springframework.context.support.AbstractApplicationContext#addBeanFactoryPostProcessor 手动新增了 BeanFactoryPostProcessor 实例，否则会不生效。在 BeanModifyBeanFactoryPostProcessor 中将 Bean 的作用域修改为原型，最后的断言没有报错也证明了 BeanModifyBeanFactoryPostProcessor 成功在运行时修改了 BeanDefinition 的元信息。

事实上，BeanFactoryPostProcessor 是对 BeanFactory 的后置处理器，只是可以用于获取 BeanDefinition 实例从而在运行时改变 BeanDefinition 的元信息而已。

至此，关于 BeanDefinition 的介绍到这里就差不多结束了。



## 6. 小结

本文介绍了 BeanDefinition 的定义，BeanDefinition 提供的核心方法以及两个示例：基于 BeanDefinition 手动注册 Bean、基于 BeanFactoryPostProcessor 修改 BeanDefinition 元信息。



## 7. 参考资料

-  Spring Framework - 5.3.5 文档

最后，本文收录于个人语雀知识库：[我所理解的后端技术](https://www.yuque.com/planeswalker/bankend/index#NkKCh)，欢迎来访。