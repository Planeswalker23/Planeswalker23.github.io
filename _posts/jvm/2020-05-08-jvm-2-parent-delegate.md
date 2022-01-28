---
layout: post
title: 我所理解的JVM系列·第2篇·类加载的基石——双亲委派机制
categories: [JVM]
keywords: JVM, 双亲委派模型
---



JVM 系列的第二篇，主题是双亲委派机制，内容包括 JVM 双亲委派机制的具体描述及源码分析、类加载器分类、双亲委派机制的好处以及打破双亲委派机制的四种场景等内容。



![cover](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643355614476-ffa9a998-fa3a-4c0a-a3b8-bac274a2a668.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_26%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fresize%2Cw_900%2Climit_0)



## 1. 开篇词

JVM 是每个后端同学都应该要懂但可能不会经常使用到的技术，同时在面试中的出现频率也是较高的。


就本人不多的面试经验而言，JVM 垃圾回收机制、JVM 内存模型以及 JVM 调优是面试官比较喜欢深究的问题，而其他比如类加载机制、双亲委派机制等都是背背八股文就能对答如流的。


本系列博客主要讨论 JVM 类加载机制、双亲委派机制、内存模型、垃圾回收机制以及调优这五个主题，预计篇幅为五篇博客，主要也是为了帮助自己整理关于 JVM 的重要内容。

JVM 系列的第二篇，主题是双亲委派机制，内容包括 JVM 双亲委派机制的具体描述及源码分析、类加载器分类、双亲委派机制的好处以及打破双亲委派机制的四种场景等内容。




## 2. 双亲委派机制


我们知道类加载机制是将一个类从字节码文件转化为虚拟机可以直接使用类的过程，但是具体由谁来执行加载过程，在执行过程中又是如何保障类加载的准确性和安全性呢？答案就是类加载器以及双亲委派机制。


双亲委派机制是：**当类加载器接收到加载类的请求时，它不会尝试自己去加载这个类，而是把这个请求委派给父类加载器去完成，只有当父类加载器反馈无法完成这个加载请求时，子加载器才会尝试自己去加载类。**

我们可以从 JDK 源码中将它的工作机制一窥究竟。




### 2.1 ClassLoader#loadClass(String, boolean)


这是在 jdk1.8 的 **java.lang.ClassLoader** 类中的源码，JVM 就是通过这个方法来实现类的加载的。


```
public class ClassLoader {
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException{
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            // 首先，检查该类是否已经被当前类加载器加载，若当前类加载未加载过该类，调用父类的加载类方法去加载该类（如果父类为null的话交给启动类加载器加载）
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    // 如果父类未完成加载，使用当前类加载器去加载该类
                    long t1 = System.nanoTime();
                    c = findClass(name);
                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                // 链接指定的类
                resolveClass(c);
            }
            return c;
        }
    }
}
```


看完了上面的代码以及注释，我们知道这显然就是双亲委派机制的代码实现，它的具体流程是：


1. 当类加载器接收到类加载的请求时，首先检查该类是否已经被当前类加载器加载；
1. 若该类未被加载过，当前类加载器会将加载请求委托给父类加载器去完成；
1. 若当前类加载器的父类加载器（或父类的父类……向上递归）为 null，会委托启动类加载器完成加载；
1. 若父类加载器无法完成类的加载，当前类加载器才会去尝试加载该类。



## 3. 类加载器分类

在 JVM 中预定义的类加载器有3种：启动类加载器(Bootstrap ClassLoader)、扩展类加载器(Extension ClassLoader)、应用类类加载器(App ClassLoader)，另外还有一种是用户自定义的类加载器，它们各自有各自的职责。




### 3.1 启动类加载器 Bootstrap ClassLoader


启动类加载器是所有类加载器的"老祖宗"，它是由 C++ 实现的，不继承于 **java.lang.ClassLoader** 类。启动类加载器的加载是由虚拟机中的一段 C++ 代码在虚拟机启动时完成的，所以它没有父类加载器，在加载完成后，它会负责去加载扩展类加载器和应用类加载器。

启动类加载器会加载 **Java 的核心类**——位于**<JAVA_HOME>\lib** 中的类，或者被 **-Xbootclasspath** 参数所指定的路径中，并且是虚拟机能够识别的类库（仅按照文件名识别,如 **rt.jar、tools.jar**，名字不符合的类库即使放在lib目录中也不会被加载）。




### 3.2 拓展类加载器 Extension ClassLoader


拓展类加载器继承于 **java.lang.ClassLoader** 类，它的父类加载器是启动类加载器，而启动类加载器在 Java 中被表示为 null。


> 引自 jdk1.8 ClassLoader#getParent() 方法的注释，这个方法是用于获取类加载器的父类加载器: Returns the parent class loader for delegation. Some implementations may use null to represent the bootstrap class loader. This method will return null in such implementations if this class loader's parent is the bootstrap class loader.



拓展类加载器负责加载 **<JAVA_HOME>\lib\ext** 目录中的，或者被 **java.ext.dirs** 系统变量所指定的路径中的所有类。

需要注意的是扩展类加载器仅支持加载被打包为 **.jar**格式的字节码文件。




### 3.3 应用类类加载器 App ClassLoader


应用类加载器继承于 **java.lang.ClassLoader** 类，它的父类加载器是扩展类加载器。

应用类加载器负责加载用户类路径 classpath 上所指定的类库。如果应用程序中没有自定义的类加载器，一般情况下应用类加载器就是程序中默认的类加载器。




### 3.4 自定义类加载器 Custom ClassLoader


自定义类加载器继承于 **java.lang.ClassLoader** 类，它的父类加载器是应用类加载器。这是用户自定义的类加载器,可加载指定路径的字节码文件。

自定义类加载器需要继承 **java.lang.ClassLoader** 类并重写 **findClass** 方法（下文有说明为什么不重写 **loadClass** 方法），用于实现自定义的类加载逻辑。




## 4. 双亲委派机制的好处


1. 基于双亲委派机制规定的这种带有优先级的层次性关系，虚拟机运行程序时就能够**避免类的重复加载**。
   - 当父类类加载器已经加载过类时，如果再有该类的加载请求传递到子类类加载器，子类类加载器执行 **loadClass** 方法，然后委托给父类类加载器尝试加载该类，但是父类类加载器执行 **Class<?> c = findLoadedClass(name);** 检查该类是否已经被加载过这一阶段就会检查到该类已经被加载过，直接返回该类，而不会再次加载此类。
2. 双亲委派机制能够**避免核心类篡改**。一般我们描述的核心类是 **rt.jar、tools.jar** 这些由启动类加载器加载的类，这些类库在日常开发中被广泛运用，如果被篡改，后果将不堪设想。
   - 假设我们自定义了一个 **java.lang.Integer** 类，当加载类的请求传递到启动类加载器时，启动类加载器执行 **findLoadedClass(String)** 方法发现 **java.lang.Integer** 已经被加载过，然后直接返回该类，加载该类的请求结束。虽然避免核心类被篡改这一点的原因与避免类的重复加载一致，但这还是能够作为双亲委派机制的好处之一的。



## 5. 打破双亲委派机制

当双亲委派机制不满足需求时，自然促使了用户通过自定义的类加载机制进行类的加载，这里举例常见的打破双亲委派机制的四种场景。




### 5.1 重写 ClassLoader#loadClass 方法

由于历史原因，即**ClassLoader** 类在 JDK1.0 时就已经存在，而双亲委派机制是在 JDK1.2 之后才引入的。在未引入双亲委派机制时，用户自定义的类加载器需要继承 **java.lang.ClassLoader** 类并重写 **loadClass()** 方法，因为虚拟机在加载类时会调用 **ClassLoader#loadClassInternal(String)**，而这个方法（源码如下）会调用自定义类加载重写的 **loadClass()** 方法。


而在引入双亲委派机制后，**ClassLoader#loadClass** 方法实际就是双亲委派机制的实现，如果重写了此方法，相当于打破了双亲委派机制。为了让用户自定义的类加载器也遵从双亲委派机制，JDK 新增了 **findClass** 方法，用于实现自定义的类加载逻辑。


```java
class ClassLoader {
    // This method is invoked by the virtual machine to load a class.
    private Class<?> loadClassInternal(String name) throws ClassNotFoundException{
        // For backward compatibility, explicitly lock on 'this' when
        // the current class loader is not parallel capable.
        if (parallelLockMap == null) {
            synchronized (this) {
                 return loadClass(name);
            }
        } else {
            return loadClass(name);
        }
    }
    // 其余方法省略......
}
```



### 4.2 线程上下文类加载器


**由于双亲委派机制规定的层次性关系，导致子类类加载器加载的类能访问父类类加载器加载的类，而父类类加载器加载的类无法访问子类类加载器加载的类。**


为了让上层类加载器加载的类能够访问下层类加载器加载的类，或者说让父类类加载器委托子类类加载器完成加载请求，JDK 引入了线程上下文类加载器，藉由它来打破双亲委派机制的屏障。

如我们常用的 **JDBC** 就是通过线程上下文类加载器来打破双亲委派机制的。




### 4.3 Tomcat 类加载机制


为了实现一个 Web 容器能够部署多个依赖不同版本公共代码的应用程序、相同版本类库可共享、Web 容器依赖类库与应用程序依赖类库隔离以及热部署 JSP 文件的功能，Tomcat 有自己特有的类加载机制。


除了启动类加载器、拓展类加载器、应用类加载器之外，Tomcat 还设计了 **Common ClassLoader**, **Catalina ClassLoader**, **Shared ClassLoader**, **Webapp ClassLoader** 以及 **JasperLoader**，各个类加载器的作用如下：


- CommonLoader：Tomcat 最基本的类加载器，加载路径中的 class 可以被 Tomcat 容器本身以及各个 Webapp 访问
- catalinaLoader：Tomcat 容器私有的类加载器，加载路径中的 class 对于 Webapp 不可见
- sharedLoader：各个 Webapp 共享的类加载器，加载路径中的 class 对于所有 Webapp 可见，但是对于 Tomcat 容器不可见
- WebappClassLoader：各个 Webapp 私有的类加载器，加载路径中的 class 只对当前 Webapp 可见
- JasperLoader：JasperLoader 的加载范围仅仅是这个 JSP 文件所编译出来的那一个 .Class 文件，它出现的目的就是为了被丢弃。当Web容器检测到 JSP 文件被修改时，会替换掉目前的 JasperLoader 的实例，并通过再建立一个新的 JSP 类加载器来实现 JSP 文件的 HotSwap 功能。



### 4.4 网状类加载器


当用户需要程序的动态性，比如代码热替换、模块热部署等时，双亲委派机制就不再适用，类加载器会发展为更为复杂的网状结构。



## 5. 小结


1. 双亲委派机制源码分析
1. 类加载器的分类及作用
1. 双亲委派机制的好处
1. 打破双亲委派机制的四种场景



## 6. 参考


- 深入理解Java虚拟机（第3版）

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。