---
layout: post
title: JVM 类加载机制及初始化时机分析
categories: [JVM]
description: JVM 类加载机制及初始化时机分析
keywords: JVM, 类加载机制
---

> 学习 JVM 的第 n-2 天，了解了类加载机制，以及初始化主动引用及被动引用的各种情况，在此记录分享。

## 1. 类加载机制简述
Java虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这个过程被称作虚拟机的类加载机制。

> 上面这段话是在周志明大佬的《深入理解Java虚拟机》中的，作为类加载机制的概念在此摘录。

当我们在编译器中选择运行下面这个`Hello World`程序，从点击运行到程序停止运行会经过一系列复杂的过程，这些关于该类的过程就是类的生命周期。

```java
public class HelloWorld {
    public static void main(String[] args){
        System.out.println("Hello World"); 
    }
}
```

## 2. 类的生命周期
类的生命周期分为加载、验证、准备、解析、初始化、使用、卸载七个阶段。其中验证、准备、解析三个阶段被统称为连接。

![2020050701.png](https://planeswalker23.github.io/images/posts/2020050701.png)

下面，就针对各个阶段做一个简单的介绍。

### 2.1 加载
在加载阶段，虚拟机会通过类的全限定名查找并加载类的二进制字节流到方法区中。

我们可以通过在启动时添加 JVM 参数`-XX:+TraceClassLoading`，打开打印类的加载顺序功能。

### 2.2 验证
在验证阶段，虚拟机需要确保被加载类的正确性，符合虚拟机规范，以及不会危害虚拟机自身的安全。

在此阶段中又有四个阶段的验证：文件格式验证、元数据验证、字节码验证和符号引用验证。具体可参考《深入理解Java虚拟机（第三版）》7.3.2节。

### 2.3 准备
在准备阶段，虚拟机会为类的静态变量分配内存，并将其初始化为默认值。

如果类字段的字段属性表中存在`ConstantValue`属性的基本类型或字符串（即被final修饰），那在准备阶段变量值就会被初始化为初始值而非默认值。

假设上文的`HelloWorld`类中有两个类变量定义如下，那么在准备阶段这两个变量的值分别是 a=0，b=2。

```java
public class HelloWorld {
    private static int a = 1;
    private static final int b = 2;
}
```
      
### 2.4 解析
在解析阶段，虚拟机会将常量池内的符号引用替换为直接引用。如果符号引用指向一个未被加载的类，那么解析阶段将触发此类的加载。

### 2.5 初始化
在初始化阶段，虚拟机会为类的静态变量赋予正确的初始值，这些赋值操作以及静态代码块中的代码会被编译器统一置于一个`<Clinit>`方法中，这个方法仅会被执行一次。所以我们可以根据类的静态代码块是否执行来判断一个类是否进行了初始化。

## 3. 主动引用
在 Java 虚拟机规范中规定了多种触发初始化的情况，被称为对类的主动引用。
1. 虚拟机启动时被标为启动类的类（包含 main() 方法的类）
2. 在类的字节码中遇到`new、getstatic、putstatic、invokestatic`这四条指令时，会触发类的初始化。
    - 这四条字节码指令分别对应创建类的实例、访问一个类的静态字段、设置一个类的静态字段、调用一个类的静态方法。
    - 这里的静态字段不包括在编译期就能确定的静态常量，因为静态常量会存放到调用方的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。
3. 当初始化类的时候，如果父类还没有进行过初始化，则需要先触发其父类的初始化。
4. 使用反射 API 对类进行反射调用时，会初始化这个类。
5. 当初次调用 MethodHandle 实例时，初始化该 MethodHandle 指向的方法所在的类。
6. 当一个接口中定义了 default 默认方法时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。

## 4. 被动引用
对于上述的触发初始化的主动引用情况有一些例外的情况：

### 4.1 静态字段
对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。

### 4.2 数组
通过数组定义来引用类，不会触发此类的初始化。因为数组由 Java 虚拟机直接生成的，通过下面的例子来说明这一情况。

在`HelloWorld`类中定义一个主方法，并在主方法中定义两个数组变量如下：
```JAVA
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
    
然后对字节码文件进行反编译，在命令行中输入`javap -c HelloWorld`，得到反编译的字节码指令如下：

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

可以看到，对于原始类型的数组变量，在字节码中通过指令`newarray`完成创建；对于引用类型的数组变量，在字节码中通过指令`anewarray`完成创建。

而对于引用类型的`MyObject`类，`anewarray`这条指令与上文中叙述的几种主动引用情况不符合，不满足初始化的条件，这与上述测试代码中没有执行`MyObject`类的静态代码块的情况相符，即没有触发`MyObject`类的初始化。

### 4.3 常量字段编译期不确定
当一个常量字段的值在编译期不确定时(如`UUID.random().toString()`)，那么它不会被放到调用类的常量池中。因此即便这个静态字段是一个常量（被final关键字修饰），但由于它在编译期是不确定的，所以在程序运行时还是会主动使用这个常量所在的类，从而触发常量所在类的初始化。

### 4.4 父类接口
一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中运行期才确定的常量）才会初始化。

由于接口是无法定义静态代码块的，所以无法像类那样通过类静态代码块的执行与否判断是否发生初始化。但是在接口的初始化过程中，编译器同样会为接口生成`<clinit>()`构造器，用于初始化接口中所定义的成员变量。

在初始化阶段，虚拟机会为类的静态变量赋予正确的初始值，当然接口类也会遵从这条规定。所以我们可以通过初始化阶段对静态字段的赋值来观察接口类是否进行了初始化，下面是验证的过程。

在父类接口中定义一个`int parentA = 1/0;`，然后通过子类访问父类的`parentRand`常量，根据上述第三条的结论，编译期无法确定的常量`parentRand`不会被放入调用类`InterfaceDemo`的常量池，那么必然会触发子接口或父接口其中至少一个接口类的初始化（假设我们不知道结论），如果触发父接口的初始化，那么会将1/0的值赋值给`parentA`，当虚拟机计算1/0时，会抛出`java.lang.ArithmeticException: / by zero`异常；如果只触发子接口的初始化，则不会抛出异常。

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
    
运行`InterfaceDemo`类，输出如下，其中`InterfaceDemo.java:15`第15行正是`parentA`定义的位置，从而可以得出结论：在子接口使用到父接口时会触发父接口的初始化。

```java
Exception in thread "main" java.lang.ExceptionInInitializerError
    at InterfaceDemo.main(InterfaceDemo.java:10)
Caused by: java.lang.ArithmeticException: / by zero
    at ParentInterface.<clinit>(InterfaceDemo.java:15)
    ... 1 more
```
    
然后将`main`函数中的输出改为`ChildInterface.childRand`，运行的结果时输出了一个`UUID`，没有抛出除零异常，从而得出结论：若子接口不使用父接口，不会触发父接口的初始化。
综上所述，验证了第4条结论——一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中运行期才确定的常量）才会初始化。

## 总结
1. Java虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这个过程被称作虚拟机的类加载机制。
2. 类加载过程分为加载、验证、准备、解析、初始化，其中验证、准备、解析三个阶段被统称为连接。以及各个阶段虚拟机做的工作。
3. 六种主动引用的情况。
4. 四种主动引用中被动引用的情况及示例。

> 版权声明：本文为[Planeswalker23](https://github.com/Planeswalker23)所创，转载请带上原文链接，感谢。<br>
> 本文已上传个人公众号，欢迎扫码关注。

![wechat](https://planeswalker23.github.io/images/wechat.png)