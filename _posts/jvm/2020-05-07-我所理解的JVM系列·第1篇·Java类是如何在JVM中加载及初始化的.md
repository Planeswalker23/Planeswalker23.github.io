---
layout: post
title: 我所理解的JVM系列·第1篇·Java类是如何在JVM中加载及初始化的？
categories: [JVM]
keywords: JVM, 类加载机制, 初始化时机
---



JVM 系列的第一篇，主题是类加载机制，内容包括 JVM 类加载机制的具体描述、类生命周期各阶段的具体工作、类初始化阶段的主动引用及被动引用区分等内容。



![JVM-封面](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/jvm/1/JVM-封面.3gqgtcoukqk0.jpg))



## 1. 开篇词

JVM 是每个后端同学都应该要懂但可能不会经常使用到的技术，同时在面试中的出现频率也是较高的。


就本人不多的面试经验而言，JVM 垃圾回收机制、JVM 内存模型以及 JVM 调优是面试官比较喜欢深究的问题，而其他比如类加载机制、双亲委派机制等都是背背八股文就能对答如流的。


本系列博客主要讨论 JVM 类加载机制、双亲委派机制、内存模型、垃圾回收机制以及调优这五个主题，预计篇幅为五篇博客，主要也是为了帮助自己整理关于 JVM 的重要内容。

JVM 系列的第一篇，主题是类加载机制，内容包括 JVM 类加载机制的具体描述、类生命周期各阶段的具体工作、类初始化阶段的主动引用及被动引用区分等内容。




## 2. 类加载机制简述


Java 虚拟机把描述类的数据从 Class 文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的 Java 类型，这个过程被称作**虚拟机的类加载机制**。


> 上面这段话是在周志明大佬的《深入理解Java虚拟机》中的，作为类加载机制的概念在此摘录。



## 3. 类加载过程


Java 虚拟机的类加载过程分为：加载、验证、准备、解析、初始化五个步骤，其中验证、准备、解析被统称为链接阶段。


![jvm-1-1](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/jvm/1/jvm-1-1.h3t5ubxov54.jpg)



### 3.1 加载

在加载阶段，Java 虚拟机会**通过类的全限定名查找并加载类的二进制字节流到方法区中**。我们可以通过在 Java 程序启动时添加 **-XX:+TraceClassLoading** 参数，打开输出类的加载顺序功能。




### 3.2 验证


在验证阶段，虚拟机需要确保被加载类**符合虚拟机规范**，同时低版本的 JVM是无法加载高版本JVM入的类库的。


在此阶段中又有四个阶段的验证：


- 文件格式验证
- 元数据验证
- 字节码验
- 和符号引用验证

> 具体可参考《深入理解Java虚拟机（第三版）》7.3.2节。



### 3.3 准备


在准备阶段，虚拟机会**为类的静态变量分配内存，并将其初始化为静态变量所指定类型的默认值**。


如果静态变量是 **ConstantValue** 属性的基本类型或字符串（即静态变量是被 final 修饰的常量），那在准备阶段它的值就会被初始化为**初始值**。


需要注意的是，在此阶段中，类的实例对象尚未分配内存，所以该阶段是在方法区上进行的。


例如，有一个 HelloWorld 类中有两个类的静态变量定义如下，那么在准备阶段这两个变量的值的情况分别是 a=0，b=2。


```java
public class HelloWorld {
    private static int a = 1;
    private static final int b = 2;
}
```



### 3.4 解析

在解析阶段，虚拟机会**将常量池内的符号引用替换为直接引用**。如果符号引用指向一个未被加载的类，那么解析阶段将触发未被加载类的加载。




### 3.5 初始化

在初始化阶段，虚拟机会**为类的静态变量赋予正确的初始值**，这些赋值操作以及静态代码块中的代码会被编译器统一置于一个 <Clinit> 方法中，这个方法仅会被执行一次。所以我们可以根据类的 <Clinit> 方法只会执行一次的特性，通过观察类的静态代码块是否执行来判断一个类是否进行了初始化。

下面做一个验证初始化阶段为类的静态变量赋予正确的初始值，以及类的初始化方法只执行一次的实验：


```java
public class HelloWorld {
    private static int a = 2;

    static {
        System.out.println("开始执行类的初始化方法，此时a=" + a);
        a = 1;
        System.out.println("结束执行类的初始化方法，此时a=" + a);
    }

    public HelloWorld() {
        System.out.println("执行类的构造方法");
    }

    public static void main(String[] args) {
        HelloWorld helloWorld = new HelloWorld();
        helloWorld = new HelloWorld();
    }
}
```


执行程序，控制台输出内容如下：


```java
开始执行类的初始化方法，此时a=2
结束执行类的初始化方法，此时a=1
执行类的构造方法
执行类的构造方法
```

从程序的执行结果也可以看出：在执行类的静态代码块之前，静态变量已经被赋予了初始值，同时，类的静态代码块只会执行一次。




## 4. 主动引用


在 Java 虚拟机规范中规定了多种触发初始化的情况，被称为对类的**主动引用**，具体触发初始化的情况如下：


1. 虚拟机启动时被标为启动类的类（包含 main 方法的类）
1. 在类的字节码中遇到 **new、getstatic、putstatic、invokestatic** 这四条指令时，会触发类的初始化。
   - **new** : 创建类的实例
   - **getstatic** : 访问一个类的静态字段
   - **putstatic** : 设置一个类的静态字段
   - **invokestatic** : 调用一个类的静态方法。
   - 这里的静态字段不包括在编译期就能确定的静态常量，因为静态常量会存放到调用方的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。
3. 当初始化类的时候，如果父类还没有进行过初始化，则需要先触发其父类的初始化。
3. 使用反射 API 对类进行反射调用时，会初始化这个类。
3. 当初次调用 **MethodHandle** 实例时，初始化该 **MethodHandle** 指向的方法所在的类。
3. 当一个接口中定义了 **default** 默认方法时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。



## 5. 被动引用

对于上述的触发初始化的主动引用情况有一些例外的情况，被称为**被动引用**。




### 5.1 静态字段

对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。




### 5.2 数组


通过数组定义来引用类，不会触发此类的初始化。因为数组由 Java 虚拟机直接生成的，通过下面的例子来说明这一情况。


在 HelloWorld 类中定义一个主方法，并在主方法中定义两个数组变量如下：


```java
public class HelloWorld {
    public static void main(String[] args){
        int[] a = new int[10];
        MyObject[] b = new MyObject[10];
    }
}

class MyObject {
    static {
        System.out.println("MyObject 类初始化...");
    }
}
```


然后对字节码文件进行反编译，在命令行中输入**javap -c HelloWorld**，得到反编译的字节码指令如下：


```java
➜  classes git:(master) ✗ javap -c HelloWorld 
Compiled from "HelloWorld.java"
public class HelloWorld {
public HelloWorld();
    Code:
        0: aload_0
        1: invokespecial #1                  // Method java/lang/Object."<init>":()V
        4: return

public static void main(java.lang.String[]);
    Code:
        0: bipush        10
        2: newarray       int
        4: astore_1
        5: bipush        10
        7: anewarray     #2                  // class MyObject
    10: astore_2
    11: return
}
```


可以看到，对于原始类型的数组变量，在字节码中通过指令 **newarray** 完成创建；对于引用类型的数组变量，在字节码中通过指令 **anewarray** 完成创建。


而对于引用类型的 MyObject 类，**anewarray** 这条指令与上文中叙述的几种主动引用情况不符合，不满足初始化的条件，这与上述测试代码中没有执行 MyObject 类的静态代码块的情况相符，即没有触发 MyObject 类的初始化。



### 5.3 常量字段编译期不确定

当一个常量字段的值在编译期不确定时(如 **UUID.random().toString()**)，那么它不会被放到调用类的常量池中。因此即便这个静态字段是一个常量（被 final 关键字修饰），但由于它在编译期是不确定的，所以在程序运行时还是会主动使用这个常量所在的类，从而触发常量所在类的初始化。




### 5.4 父类接口


一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中运行期才确定的常量）才会初始化。


由于接口是无法定义静态代码块的，所以无法像类那样通过类静态代码块的执行与否判断是否发生初始化。但是在接口的初始化过程中，编译器同样会为接口生成 <clinit> 构造器，用于初始化接口中所定义的成员变量。


在初始化阶段，虚拟机会为类的静态变量赋予正确的初始值，当然接口类也会遵从这条规定。所以我们可以通过初始化阶段对静态字段的赋值来观察接口类是否进行了初始化，下面是验证的过程。


在父类接口中定义一个 `int parentA = 1/0;`，然后通过子类访问父类的 parentRand 常量，根据上述第三条的结论，编译期无法确定的常量 parentRand 不会被放入调用类 InterfaceDemo 的常量池，那么必然会触发子接口或父接口其中至少一个接口类的初始化（假设我们不知道结论），如果触发父接口的初始化，那么会将1/0的值赋值给 parentA，当虚拟机计算1/0时，会抛出 `java.lang.ArithmeticException: / by zero` 异常；如果只触发子接口的初始化，则不会抛出异常。


```java
public class InterfaceDemo {

    public static void main(String[] args) {
        System.out.println(ChildInterface.parentRand);
    }
}

interface ParentInterface {
    int parentA = 1/0;
    String parentRand = UUID.randomUUID().toString();
}

interface ChildInterface extends ParentInterface {
    String childRand = UUID.randomUUID().toString();
}
```


运行 InterfaceDemo 类，输出如下，其中 InterfaceDemo.java:15 第15行正是 parentA 定义的位置，从而可以得出结论：在子接口使用到父接口时会触发父接口的初始化。


```java
Exception in thread "main" java.lang.ExceptionInInitializerError
    at InterfaceDemo.main(InterfaceDemo.java:10)
Caused by: java.lang.ArithmeticException: / by zero
    at ParentInterface.<clinit>(InterfaceDemo.java:15)
    ... 1 more
```

然后将 `main` 函数中的输出改为 ChildInterface.childRand，运行的结果时输出了一个 UUID，没有抛出除零异常，从而得出结论：若子接口不使用父接口，不会触发父接口的初始化。
综上所述，验证了第4条结论——一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中运行期才确定的常量）才会初始化。



## 6. 小结


1. Java虚拟机把描述类的数据从 Class 文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的 Java 类型，这个过程被称作虚拟机的类加载机制。
1. 类加载过程分为加载、验证、准备、解析、初始化，其中验证、准备、解析三个阶段被统称为连接。以及各个阶段虚拟机做的工作。
1. 六种主动引用的情况。
1. 四种主动引用中被动引用的情况及示例。



## 7. 参考


- 深入理解Java虚拟机（第3版）
- 深入理解 JVM（张龙）
- [第03讲：大厂面试题：从覆盖 JDK 的类开始掌握类的加载机制](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=31#/detail/pc?id=1027)

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。