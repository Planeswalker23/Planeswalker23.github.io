---
layout: post
title: 静态代理、动态代理及 $Proxy0 类源码分析
categories: [Java]
description: 静态代理、动态代理及 $Proxy0 源码分析
keywords: 代理, 静态代理, 动态代理, Proxy
---

从前从前，有个面试官问我动态代理和静态代理的区别，我当时支支吾吾没说清楚，只提到了动态代理需要实现`InvocationHandler`接口，然后使用`Proxy`类反射创建实例云云。至于静态代理……这玩意不就是一种设计思想？

面试官笑了笑，从此天涯路人不相逢。

我痛定思痛，一定要把代理这一块搞懂，于是乎有了这篇文章。以后再也不怕面试官问我关于静态代理和动态代理的问题了！

## 1. 什么是代理
说到代理，就不得不提设计模式中的**代理模式**，代理模式就是对代理思想的一种设计模式实现。

百度百科对于代理模式的定义是这样的：

> 为其他对象提供一种代理以控制对这个对象的访问。在某些情况下，一个对象不适合或者不能直接引用另一个对象，而代理对象可以在客户端和目标对象之间起到中介的作用。

在这段定义中有这样两个字：中介。

我想下面这个例子可以比较好的解释代理模式。

相信在一个陌生的城市打拼的程序员们在初期都会遇到这样一个问题：租房。我们通常有三种方式，第一可以自己在闲鱼、豆瓣、自如等信息网站去找房源，第二直接去心仪的小区公告栏看看有没有招租信息（当然可能被中介的广告霸占），第三就是联系房产中介，中介会帮你挑选你想要租的房子，只不过需要付一笔服务费。

![2020052803](https://planeswalker23.github.io/images/posts/2020052803.png)

假设我选择**委托**中介来租房，在这个过程中就可以把房子抽象为一个类，这是我最终想要得到的东西。然后把帮我租房的中介抽象为一个类，通过**委托**中介，我可以得到自己想要的房子。同时，这两个类实现了相同的接口，可以这么去理解这里相同接口的作用：房子通过**接口**注册在数据库中，中介通过接口找到了注册在数据库中的房子。

而我**委托**中介帮我找房子的这个过程，就是代理。

## 2. 定义与类图
在根据上面说的租房的例子来编写实际的代码作为静态代理的示例之前，首先得了解一下代理模式中的几个角色，代理模式中有三个主要角色，即抽象主题角色、真实角色（被代理角色）以及代理类角色。
### 2.1. 主题角色 (Subject)
主题角色可以是接口，也可以是抽象类。它定义了真实角色和代理类角色共有的方法，主题类让这两者具有一致性。

同时，也正是基于主题角色，才能实现代理的功能。
### 2.2. 真实角色 (RealSubject)
真实角色就是被代理类，在上面的例子中就是房子，同时也是具体业务逻辑的执行者。
### 2.3. 代理类角色 (Proxy)
代理类角色的内部含有对真实角色 RealSubject 的引用，它负责对被代理角色的调用，并在被代理角色处理前后做预处理和后处理。

在上面租房的例子中就是房产中介。
### 2.4. Client
有人可能会问了，那“我”呢？简单点说，其实“我”就是测试方法中的 main 方法，负责调用代理角色的方法。

### 2.5 类图
![2020052804](https://planeswalker23.github.io/images/posts/2020052804.png)
> 图片来源于[Proxy模式——静态代理](https://www.jianshu.com/p/5004b0b48511)

## 3. 静态代理
所谓的静态代理，就是在程序启动之前代理类的 .class 文件就已经存在。而代理类可能是程序员直接创建的 .java 文件，或者是借助某些工具生成的 .java 文件，但无一例外都必须再由编译器编译成 .class 文件之后再启动程序。

### 3.1. 静态代理实现
基于上面租房的例子使用代码实现。

首先创建主题角色，它是一个接口，这个接口拥有一个方法，而这个方法是需要被其他两个角色重写的。

```java
public interface Subject {

    /**
     * 各个角色的公用方法
     */
    void job();
}
```

然后是真实角色，也就是被代理角色、真正的业务逻辑执行者。

```java
public class House implements Subject {

    @Override
    public void job() {
        System.out.println("我是客户想要的房子，通过 job 方法注册在数据库中");
    }
}
```

然后代理类，它能够增强被代理角色的方法。代理类就是帮助我”找到好房子“的房产中介。

当一个房产中介拥有一个客户（RealSubject）时，才会发挥他的作用，在我的”意图“被实现前后，它分别可以对我的”意图“进行增强。

```java
public class Proxy implements Subject {

    private RealSubject realSubject;

    public Proxy(RealSubject realSubject) {
        this.realSubject = realSubject;
    }

    @Override
    public void job() {
        System.out.println("我是中介，我会在数据库中检索，帮助客户找到心仪的房子");
        house.job();
        System.out.println("我是中介，找到了数据库中符合客户需求的方法");
    }
}
```

最后，“我”出场了，“我”委托中介寻找房子。

```java
public class Client {
    public static void main(String[] args) {
        // 我可能心仪的房子
        House house = new House();
        // 代理类——房产中介
        Proxy proxy = new Proxy(house);
        // "我"委托中介去寻找房子
        proxy.job();
    }   
}
```

最终“我”执行的是代理类（中介）的 job 方法，由于代理类持有一个真实角色（房子），程序又会执行真实角色的 job 方法，这样就实现了“我”委托中介找到房子的静态代理过程。

### 3.2. 静态代理的优缺点
#### 3.2.1. 优点
1. 业务类只需要关注业务逻辑本身，保证了业务类的重用性。
2. 客户端只需要知道代理，无需关注具体实现。
> 中介只需要关注自己能找房子的效率和质量就可以了，无论谁想来委托中介，都能找到房子。而“我”不需要知道中介是如何找房子的，只要他帮我找到房子，就可以了。

#### 3.2.2. 缺点
1. 由于代理类和被代理类都实现了主题接口，它们都有相同的方法，导致大量代码重复。同时如果主题接口新增了一个方法，那么代理类与被代理类也都需要实现这个方法，增加了维护代码的复杂度。
2. 如果代理类要为其他真实角色提供委托服务的话，就需要实现其他的接口，当规模变大时也会增加代码复杂度。
> 如果中介不仅提供租房服务，还提供打游戏、卖房子、卖电影票、卖彩票、陪聊天、陪玩游戏等等一系列服务，那么他将变得无比庞杂，没有人敢动他（这里的他指代码）。

## 4. 动态代理
上面讨论的是静态代理，接下来再聊聊动态代理。那么什么是动态代理呢？

所谓的动态代理，就是在程序运行时创建代理类的代理方式。而这也是静态代理和动态代理的区别。

### 4.1. 动态代理实现
既然是在程序运行时生成的代理类，那么必然需要借助其他的工具来生成，而在 Java 中就是通过 java.lang.reflect.Proxy 类来生成代理类的。同时，还需要实现`InvocationHandler`接口来实现方法调用，下面就用代码来实现动态代理。

同样是上面租房的例子，接口`Subject`不变，被代理类`House`也不变，需要新建一个动态代理类。
```java
public class DynamicProxy implements InvocationHandler {

    private Object target;

    public DynamicProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("被代理的类:" + proxy.getClass());
        System.out.println("被代理的类的执行方法:" + method.getName());
        Object object = method.invoke(target, args);
        System.out.println("被代理的类的方法执行完成");
        return object;
    }
}
```

动态代理类实现了`InvocationHandler`接口，同时与静态代理一样，在它内部也持有一个对象，这个对象正是被代理对象，然后在代理类的`invoke`方法中调用具体的方法。

最后再客户端，也就`Client`角色中编写测试代码。

```java
import java.lang.reflect.Proxy;

public class Client {

    public static void main(String[] args) {
    // 我可能心仪的房子
    Subject subject = new House();
    // 代理类——房产中介
    DynamicProxy dynamicProxy = new DynamicProxy(subject);
    // 获取代理类
    Subject proxyInstance = (Subject) Proxy.newProxyInstance(subject.getClass().getClassLoader(),
            subject.getClass().getInterfaces(),
            dynamicProxy);
    // "我"委托中介去寻找房子
    proxyInstance.job();
    }
}
```

与静态代理不同的是，我们需要通过`Proxy.newProxyInstance`方法来实例化动态生成的代理类，而这个方法中的参数分别代表的意义是：
- `ClassLoader loader`: 被代理类的类加载器
- `Class<?>[] interfaces`: 被代理类实现的接口
- `InvocationHandler h`: 实现指定接口`InvocationHandler`的实现类

需要这三个参数的原因是：需要通过与被代理类相同的类加载器去加载动态生成的代理类，同时代理类需要实现与被代理类相同的接口，最后需要通过实现指定接口`InvocationHandler`的实现类来完成代理调用方法的功能。

最终的输出结果是：
```
被代理的类:class com.sun.proxy.$Proxy0
被代理的类的执行方法:job
我是客户想要的房子，通过 job 方法注册在数据库中
被代理的类的方法执行完成
```

我们可以看到生成的代理类是`com.sun.proxy.$Proxy0`，通过动态代理它完成了与静态代理一样的委托任务。

### 4.2. 动态代理的优缺点
与静态代理相比，动态代理还具有不需要自己写代理类的优点，因为代理类时运行时程序自动生成的。

同时，动态代理的必须先实现`InvocationHandler`接口，然后使用`Proxy`类中的`newProxyInstance`方法动态的创建代理类，这就导致了动态代理只能代理接口。

## 5. 动态代理类源码分析
上文说到运行时生成的动态代理类会继承于`java.lang.reflect.Proxy`类，这是为什么呢？

### 5.1. 获取动态代理类源码
我们可以通过设置系统参数来保存动态生成的代理类。

```java
System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
```

你问我是怎么知道这个参数的？我也不知道，是 jdk 源码里面写死的。

有兴趣的同学可以跟踪一下`Proxy.newProxyInstance`这个方法，在经过数次跳转后，你就能找到这个系统参数了，下面给出调用链。
```java
Proxy.newProxyInstance
->getProxyClass0
->proxyClassCache.get(loader, interfaces)
->subKeyFactory.apply(key, parameter)
->ProxyClassFactory.apply
->ProxyGenerator.generateProxyClass
->saveGeneratedFiles
->private static final boolean saveGeneratedFiles = (Boolean)AccessController.doPrivileged(new GetBooleanAction("sun.misc.ProxyGenerator.saveGeneratedFiles"))
```

你问我怎么知道调用链是这样的？看注释啊...

在开启`saveGeneratedFiles`参数后，我们会发现在项目中多出了`com.sun.proxy.$Proxy0`类，打开它就是生成的动态代理类源码。
```java
public final class $Proxy0 extends Proxy implements Subject {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;
    // equals 和 hashCode 方法省略...

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void job() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("org.planeswalker.proxy.statical.Subject").getMethod("job");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

### 5.2. 为什么要重写`equals`、`toString`、`hashCode`方法
可以看到，在动态代理类中有四个私有静态成员变量，结合 static 代码块，我们知道这四个`Method`分别代表了`equals`、`toString`、`job`、`hashCode`方法。

`job`方法很好理解，因为这是我需要动态代理类去调用被代理类的方法。而另外三个方法，为什么需要重写？

从源码中可以看到，这三个方法实际是调用了`InvocationHandler`接口实现类的相应方法。而我们知道动态代理类其实相当于一个中间件，通过动态代理类我们实际想要调用的是被代理类的方法，这么一想就很好理解了——重写这三个方法的原因是为了让动态代理类与被代理类划上”≈“号。

如果没有重写这三个方法，那么它们的`hashcode`与`toString`将会返回不同值，这样实现的动态代理类也就不完善了。

为什么说是”≈“号而不是”=“号呢？因为动态代理类实际是一个`com.sun.proxy.$Proxy0`类，虽然它具有与被代理类相同的状态（包括大部分方法与属性），但实际上这两个类通过`equals`方法来比较返回的会是`false`，因为它们的内存地址是不一样的。

> 被代理类未重写`equals`方法，所以调用的是`Object#equals`，而这里比较的是内存地址。

### 5.3 为什么动态代理类要继承`Proxy`类
这个问题其实应该去问`jdk`的实现者，这是他们规定的，哪来的为什么？

我也去网上搜索了很多相关的问题，大部分还是指向了一个答案——继承`Proxy`类可以减少代码的冗余度。

在上面给出的动态生成的代理类源码中我们可以知道，动态代理类其实只是做了一个转发，调用的还是被代理类的方法。如果我们将被代理类的属性和方法都写在动态代理类中，而通过代理调用真实角色的方法或访问属性时依旧是通过转发，那么这些被继承的方法和属性实际上是根本没有用到的，这对于内存空间来说是一种浪费。

所以动态代理类要继承`Proxy`类。


## 6. 小结
本文讲述了代理模式、代理模式中的角色、静态代理代码实现以及优缺点、静态代理与动态代理的区别、动态代理代码实现及优缺点。

同时还提出了两个问题以及我对这两个的理解。

希望可以帮助到大家。

以上。

> 版权声明：本文为[Planeswalker23](https://github.com/Planeswalker23)所创，转载请带上原文链接，感谢。<br>
> 本文已上传个人公众号，欢迎扫码关注。

![wechat](https://planeswalker23.github.io/images/wechat.png)