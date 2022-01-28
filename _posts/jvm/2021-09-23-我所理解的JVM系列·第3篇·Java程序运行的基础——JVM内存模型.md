---
layout: post
title: 我所理解的JVM系列·第3篇·Java程序运行的基础——JVM内存模型
categories: [JVM]
keywords: JVM, 内存模型
---



JVM 系列的第三篇，主题是内存模型，内容包括 JVM 内存区域划分、常见溢出场景以及常用内存参数设置等内容。



![JVM-封面](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/jvm/3/JVM-封面.4ym0ebfxwhk0.jpg)



## 1. 开篇词

JVM 是每个后端同学都应该要懂但可能不会经常使用到的技术，同时在面试中的出现频率也是较高的。


就本人不多的面试经验而言，JVM 垃圾回收机制、JVM 内存模型以及 JVM 调优是面试官比较喜欢深究的问题，而其他比如类加载机制、双亲委派机制等都是背背八股文就能对答如流的。


本系列博客主要讨论 JVM 类加载机制、双亲委派机制、内存模型、垃圾回收机制以及调优这五个主题，预计篇幅为五篇博客，主要也是为了帮助自己整理关于 JVM 的重要内容。


JVM 系列的第三篇，主题是内存模型，内容包括 JVM 内存区域划分、常见溢出场景以及常用内存参数设置等内容。


在我上一次求职的时候，我记得印象最深的关于 JVM 的面试题中有一个就是面试官让我说一说 JVM 内存区域划分，由于当时只准备了类加载机制以及双亲委派机制，所以只能灰溜溜地说自己对这块内容不太熟悉，口气黯然且不自信，于是下定决心改过自新重新做人，这才整理出这篇文章。


> 注：若没有特殊说明，本文所描述的 JDK 版本为 1.8 的 HotSpot 虚拟机。



## 2. JVM 内存模型与 Java 内存模型


在梳理 JVM 内存模型之前，需要提前说明的一个点是：JVM 内存模型和 Java 内存模型（JMM）是两个完全不一样的概念，这也是网上很多博客经常搞混的两个东西。


- **JVM 内存模型**是 Java 程序运行的基础，它描述了 Java 虚拟机运行时的内存区域划分，包括虚拟机分为哪几个区域、各个区域的作用域以及存放的内容等。
- **Java 内存模型**（Java Memory Model, JMM）是一种虚拟机规范，它用于**_屏蔽掉各种硬件和操作系统的内存访问差异，使得 Java 程序在各种平台下都能达到一致的并发效果_**。具体点来说，JMM 控制了 Java 线程之间的通信，它规定了一个线程如何和何时可以看到由其他线程修改过后的共享变量的值，以及如何同步地访问共享变量。



## 3. 内存区域划分


根据《Java 虚拟机规范》，Java 虚拟机所管理的内存被划分为下面几个区域：


![jvm-3-1-jvm内存模型](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/jvm/3/jvm-3-1-jvm内存模型.66m7552t7c40.jpg)

- 堆
- 方法区
- 虚拟机栈
- 本地方法栈
- 程序计数器

下面依次来梳理各个区域的用途。




### 3.1 堆


堆（Heap）是 Java 虚拟机所管理的内存中最大的一块区域，它是被**所有线程共享**的。


堆的作用就是**存放对象实例**，几乎所有的对象实例以及数组都会在堆中分配内存。

随着对象的创建，堆空间越来越大，当超过一定限度之后，虚拟机就会对其中不再使用的对象进行清理，这就是我们常说的**垃圾回收**（Garbage Collection, GC），一般我们所说的垃圾回收的区域就是堆空间。




#### 3.1.1 基于分代收集理论划分堆内存


现代垃圾收集器大部分都是基于分代收集理论设计的，在分代收集理论中，堆空间又可以分为**新生代**（Young Generation）、**老年代**（Old Generation），新生代又可以细分为 **Eden 区和 Survivor 区**，Survivor 区又分为 **From Survivor 区和 To Survivor 区**。


![jvm-3-2-jdk1](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/jvm/3/jvm-3-2-jdk1.2rm2ph7bidy0.jpg)



#### 3.1.2 模拟堆内存溢出


我们已经知道，堆内存中存放的是对象实例，所以在程序运行时无法被回收的对象实例数量过多时，就可能会产生堆内存溢出的现象。为了方便观察程序运行的效果，在启动时设置 JVM 启动参数：**-Xmx16m**（设置堆内存最大值为16m），测试代码如下：


```java
public class OomHeap {

    private byte[] byteObj = new byte[1024*1024];

    public static void main(String[] args) {
        List<OomHeap> list = new ArrayList<>();
        while (true) {
            list.add(new OomHeap());
        }
    }
}
```


在程序运行数秒后，就发生了异常：


```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at io.walkers.planes.pandora.jvm.memory.OomHeap.<init>(OomHeap.java:14)
	at io.walkers.planes.pandora.jvm.memory.OomHeap.main(OomHeap.java:19)
```

这就是经典的堆内存溢出异常。




### 3.2 方法区


方法区（Method Area）也是被**所有线程共享**的区域，它用于**存储已被虚拟机加载的类信息、常量、静态变量等数据**。


- 类信息包括类的版本、字段、方法、接口和父类等信息
- 常量：指**运行时常量池**中的数据，包含了 Class 文件中的常量池表（Constant Pool Table）
   - 常量池表用于存放编译期生成的各种**字面量与符号引用**，这些内容在类加载后被存放于方法区的运行时常量池
      - 字面量：文本字符串、被声明为 final 的常量值、基本数据类型的值、其他
      - 符号引用：类和结构的完全限定名、字段名称和描述符、方法名称和描述符
   - 虚拟机还会把**符号引用翻译出来的直接引用**也存放在运行时常量池中
   - 除此之外，由于运行时常量池具备动态性（这也是它相对于 Class 文件中的常量池表具备的另外一个重要特征），程序运行期间也可以将新的常量放入运行时常量池中，比如 String 类的 intern() 方法
- 静态变量：指**字符串常量池**中的数据。
   - 为了提升性能以及减少内存开销，避免字符串的重复创建，JVM 维护了一块特殊的内存空间，就是字符串常量池。
   - 字符串常量池是由 String 类维护的。



#### 3.2.1 方法区的变迁


在 JDK1.7 以前，类信息、运行时常量池和字符串常量池是存放在方法区中的。由于当时的 HotSpot 虚拟机是基于分代收集理论进行开发的，他们通过**永久代**（Permanent Generation）来实现方法区，故而很多人将方法区与永久代视为同一个东西。事实上，开发人员只是通过永久代来实现了方法区而已。


下图就是 JDK 1.6 方法区的结构：


![jvm-3-3-方法区1](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/jvm/3/jvm-3-3-方法区1.35z3y66nhjw0.jpg)


而在 JDK1.7 的 HotSpot 虚拟机中，开发人员已经将运行时常量池和字符串常量池从方法区移到了堆中，其余部分（类信息）则存储在非堆内存中（方法区又被称为非堆 Non-Heap），如下图所示：


![jvm-3-4-方法区1](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/jvm/3/jvm-3-4-方法区1.1mtoj8zi09i8.jpg)


到了 JDK1.8 已经完全废弃了永久代的概念，并使用**存储在本地内存**中的**元空间**（Meta-space）来代替，将原来方法区中剩余的部分（类信息）存储在元空间中。而运行时常量池和字符串常量池与 JDK1.7 一样还是存在堆中，JDK1.8 方法区的结果如下图所示：


![jvm-3-5-方法区1](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/jvm/3/jvm-3-5-方法区1.1yif1zzq3n6o.jpg)


需要注意的是：方法区是 JVM 规范中的一个逻辑区域，在 JDK1.7 以前，HotSpot 虚拟机使用永久代来实现方法区，而在 JDK1.8 中，方法区包括存放在堆中的字符串常量池和运行时常量池，以及存放在本地内存中的元空间。



#### 3.2.2 使用元空间代替永久代的原因


至于为什么 JDK1.8 的开发团队要使用元空间来替代永久代，官方给出的解释是：


- **为了兼容 JRockit 与 HotSpot，统一 JVM 规范实现**。JRockit 虚拟机是没有永久代的概念的，而为了将 JRockit 中的优秀功能诸如 Java Mission Control 管理工具移植到 HotSpot 中，同时考虑到 HotSpot 虚拟机未来的发展，才将永久代移除。
- **避免永久代导致的 OOM**，永久代更容易因为内存不够用而报出 **java.lang.OutOfMemoryError: PermGen** 异常。在 JDK1.8 以前永久代的大小可以通过 **-XX：MaxPermSize** 参数来设置，即使不设置也有默认大小，所以更容易发生内存溢出问题，而使用元空间代替永久代之后，只要没有触碰到进程可用内存的上限，例如32位系统中的4GB限制，就不会发生内存溢出问题。



#### 3.2.3 模拟方法区溢出

首先需要再次申明的是：本文是基于 JDK1.8 来进行编码的。

由于在 JDK1.8 中已经彻底废除了永久代的概念，所以原本可能会出现的永久代溢出报错 **java.lang.OutOfMemoryError: PermGen space** 是不会出现了，同时关于永久代的 JVM 参数如设置永久代空间最小值 **-XX: PermSize** 和设置永久代空间最大值 **-XX: MaxPermSize** 也会无效。


在 JDK1.7 方法区中的运行时常量池和字符串常量池都移到了堆中，所以如果是要模拟方法区中运行时常量池和字符串常量池的溢出，最终得到的结果都会是堆空间溢出，即 **java.lang.OutOfMemoryError: Java heap space**。


所以说在 JDK1.8 中如果想要模拟方法区的溢出报错，就只剩下了类的元数据过多而导致的元空间内存溢出。而想要模拟这个现象，我们可以借助 CGLib 操作字节码来生成代理类来实现。


首先需要引入 CGLib 依赖:


```java
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.3.0</version>
</dependency>
```


为了方便观察程序运行的效果，在启动时设置 JVM 启动参数：**-XX: MaxMetaspaceSize=24m**（设置元空间内存最大值为24m，若元空间最大值设置的过小会报 **MaxMetaspaceSize is too small.** 异常），测试代码如下：


```java
public class OomMetaspace {

    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OomMetaspace.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                @Override
                public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                    return methodProxy.invokeSuper(objects, args);
                }
            });
            enhancer.create();
        }
    }
}
```


在程序运行数秒后，就发生了异常：


```java
Exception in thread "main" net.sf.cglib.core.CodeGenerationException: java.lang.reflect.InvocationTargetException-->null
	at net.sf.cglib.core.AbstractClassGenerator.generate(AbstractClassGenerator.java:348)
	at net.sf.cglib.proxy.Enhancer.generate(Enhancer.java:492)
	at net.sf.cglib.core.AbstractClassGenerator$ClassLoaderData.get(AbstractClassGenerator.java:117)
	at net.sf.cglib.core.AbstractClassGenerator.create(AbstractClassGenerator.java:294)
	at net.sf.cglib.proxy.Enhancer.createHelper(Enhancer.java:480)
	at net.sf.cglib.proxy.Enhancer.create(Enhancer.java:305)
	at io.walkers.planes.pandora.jvm.memory.OomMetaspace.main(OomMetaspace.java:26)
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.GeneratedMethodAccessor1.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at net.sf.cglib.core.ReflectUtils.defineClass(ReflectUtils.java:459)
	at net.sf.cglib.core.AbstractClassGenerator.generate(AbstractClassGenerator.java:339)
	... 6 more
Caused by: java.lang.OutOfMemoryError: Metaspace
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:756)
	... 11 more
```

如果看到 **java.lang.OutOfMemoryError: Metaspace** 的 报错，就说明是元空间内存溢出了。




### 3.3 虚拟机栈

虚拟机栈（VM Stack）是线程私有的，它的生命周期与线程相同。虚拟机栈描述的是 Java 方法执行的线程内存模型：每个方法被执行的时候，Java虚拟机都会同步创建一个栈帧（Stack Frame）用于存储**局部变量表、操作数栈、动态连接、方法出口**等信息。




#### 3.3.1 模拟虚拟机栈内存溢出


关于虚拟机栈和本地方法栈，在《Java 虚拟机规范》中描述了两种异常：


1. 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出 StackOverflowError 异常。
1. 如果虚拟机的栈内存允许动态扩展，当扩展栈容量无法申请到足够的内存时，将抛出 OutOfMemoryError 异常。

需要明确的是，在《Java虚拟机规范》中允许 Java 虚拟机的实现自行选择是否支持栈内存的动态扩展，而 HotSpot 虚拟机选择不支持栈内存的动态扩展。所以在 HotSpot 虚拟机中是不会在线程运行时因为扩展而导致虚拟机规范所描述的第二种栈内存溢出异常，除非在创建线程申请内存时就因无法获得足够内存而出现 OutOfMemoryError 异常。


为了方便观察程序运行的效果，在启动时设置 JVM 启动参数：**-Xss160k**（设置每个线程的栈容量为160k，若每个线程的栈容量过小会报 **The stack size specified is too small, Specify at least 160k** 异常）。模拟栈的 StackOverflowError 异常代码如下：


```java
public class StackOverflowError {

    private int stackLength = 1;

    public static void main(String[] args) {
        StackOverflowError stackOverflowError = new StackOverflowError();
        try {
            stackOverflowError.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length is " + stackOverflowError.stackLength);
            e.printStackTrace();
        }
    }

    private void stackLeak() {
        stackLength++;
        stackLeak();
    }
}
```


这段程序的运行结果如下：


```java
stack length is 774
java.lang.StackOverflowError
	at io.walkers.planes.pandora.jvm.memory.StackOverflowError.stackLeak(StackOverflowError.java:24)
	at io.walkers.planes.pandora.jvm.memory.StackOverflowError.stackLeak(StackOverflowError.java:25)
	......
```

结果表明，当线程请求的栈深度大于虚拟机所允许的最大深度，即当新的栈帧内存无法分配的时候，HotSpot 虚拟机会抛出 StackOverflowError 异常。




### 3.4 本地方法栈


本地方法栈（Native Method Stack）也是线程私有的，被用于**管理本地方法的调用**。

在 HotSpot 虚拟机中并不区分虚拟机栈与本地方法栈，因此模拟本地方法栈的内存溢出与虚拟机栈一致，在此不赘述。同时设置本地方法栈的虚拟机参数 **-Xoss** 在 HotSpot 虚拟机中是无效的，想要调节栈的容量，只能使用 **-Xss** 参数，如 **-Xss128k** 的意思是调节每个线程的栈大小为128k。



### 3.5 程序计数器


程序计数器（Program Counter Register）也是线程私有的，它是一块较小的内存空间，用于**记录各个线程执行的字节码的地址**。

程序计数器是唯一一个在《Java虚拟机规范》中没有规定任何 OutOfMemoryError 情况的区域。




## 4. 常用 JVM 参数


### 4.1 堆内存参数


- **-Xms** : 堆内存的初始值
   - 如 **-Xms256m** 代表设置堆内存的初始值为256M
- **-Xmx** : 堆内存的最大值
   - 如 **-Xmx256m** 代表设置堆内存的最大值为256M
- **-Xmn** : 新生代的最大值
   - 如 **-Xmn128m** 代表设置新生代的最大值为128M
- **-XX:SurvivorRatio** : 用来设置新生代中 eden 空间和 survivor 空间的比例
   - 如 **-XX:SurvivorRatio=8** 代表 eden 空间和两个 survivor 空间的比例是8:2，一个 survivor 空间占整个新生代的大小是 1/10
- **-XX:NewRatio** : 配置老年代与新生代与比例
   - 如 **-XX:NewRatio=2** 代表老年代与新生代的比例是2:1
- **-XX: MaxMetaspaceSize** : 设置元空间的最大值
   - 如 **-XX: MaxMetaspaceSize=24m** 代表将元空间内存的最大值设置为24m
- **-Xss** : 设置每个线程的栈容量
   - 如 **-Xss160k** 代表设置每个线程的栈容量为160k



## 5. 小结


- JVM 内存区域划分
- 为什么要使用元空间来替代永久代
- 常用 JVM 参数及设置方式



## 6. 参考资料


- 《深入理解 Java 虚拟机》（第三版）
- 《Java性能调优实战》（极客时间刘超）

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。