---
layout: post
title: 我所理解的JVM系列·第b篇·Java异常处理机制在JVM中是如何工作的
categories: [JVM]
keywords: JVM, 异常表, try-catch-finally
---



作为一个“有经验”的 Java 工程师，你一定知道什么是 try-catch-finally 代码块。但是你知道 JVM 是如何处理异常的吗？今天我们就来讲讲异常在 JVM 中的处理机制，以及字节码中异常表。



![cover](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643356091382-c14747da-02d8-4e7a-882a-72994b964517.png?x-oss-process=image%2Fresize%2Cw_900%2Climit_0)



## 1. 开篇词

作为一个“有经验”的 Java 工程师，你一定知道什么是 **try-catch-finally** 代码块。但是你知道 JVM 是如何处理异常的吗？今天我们就来讲讲异常在 JVM 中的处理机制，以及字节码中异常表。

在本文中使用到的环境及工具如下：


- jdk 1.8.0_151
- IntelliJ IDEA 及 jclasslib 插件



## 2. 字节码中的 try-catch

**Talk is cheap, show you my code!**



### 2.1 编译后的字节码


首先我编写了第一段测试代码，这里有一个 try-catch 代码块，每个代码块中都有一行输出，在 catch 代码块中捕获的是 Exception 异常。


```java
 public static void main(String[] args) {
   try {
     System.out.println("enter try block");
   } catch (Exception e) {
     System.out.println("enter catch block");
   }
 }
```


然后在命令行中先定位到这个类的字节码文件目录中，执行主方法后敲下 **javap -c 类名** 进行编译，或者直接在编译器中选择 **Build Project**，然后打开 jclasslib 工具就可以看到这个类的字节码。


我选择了第二个方法，主方法的字节码如下图：


![](https://cdn.nlark.com/yuque/0/2020/png/2331602/1600876693995-8bcbc2e1-fe1f-463f-9e62-ce9d818bfa64.png#align=left&display=inline&height=386&margin=%5Bobject%20Object%5D&originHeight=386&originWidth=800&size=0&status=done&style=none&width=800)

可以看到0~3行是 try 代码块中的输出语句，12~17行是 catch 代码块中的输出语句。

然后重点来了。


![](https://cdn.nlark.com/yuque/0/2020/png/2331602/1600876694022-52f382c6-ee1a-418f-97b8-f7356d625554.png#align=left&display=inline&height=282&margin=%5Bobject%20Object%5D&originHeight=282&originWidth=464&size=0&status=done&style=none&width=464)


第8行的字节码是 **8 goto 20**，这是什么意思呢？没错，盲猜就能猜到，这个字节码指令代表了指令将跳转到第20行开始执行。这一行是说，如果 try 代码块中没有出现异常，那么就跳转到第20行，也就是整个方法行完成后 return 了。


这是正常的代码执行流程，那么如果出现异常了，虚拟机是如何知道应该“监控” try 代码块？它又是怎么知道该捕获何种异常呢？

答案就是——异常表。




### 2.2 异常表


在一个类被编译成字节码之后，它的每个方法中都会有一张异常表。异常表中包含了“监控”的范围，“监控”何种异常以及抛出异常后去哪里处理。比如上述的示例代码，在 jclasslib 中它的异常表如下图。


![](https://cdn.nlark.com/yuque/0/2020/png/2331602/1600876694010-4b6f6414-88de-47c1-b035-036a56de1a62.png#align=left&display=inline&height=176&margin=%5Bobject%20Object%5D&originHeight=176&originWidth=800&size=0&status=done&style=none&width=800)


或者在 **javap -c** 命令下异常表是这样的：


```java
Exception table:
   from    to  target type
       0     8    11   Class java/lang/Exception
```


无论是哪种形式的异常表，我们可以知道的是，异常表中每一行就代表一个异常处理器。


- Nr. 代表异常处理器的序号
- Start PC (from)，代表异常处理器所监控范围的起始点
- End PC (to)，代表异常处理器所监控范围的结束点（该行不被包括在监控范围内，一般是 goto 指令）
- Handler PC (target)，指向异常处理器的起始位置，在这里就是 catch 代码块的起始位置
- Catch Type (type)，代表异常处理器所捕获的异常类型



如果程序触发了异常，Java 虚拟机会按照序号遍历异常表，当触发的异常在这条异常处理器的监控范围内（from 和 to），且异常类型（type）与该异常处理器一致时，Java 虚拟机就会跳转到该异常处理器的起始位置（target）开始执行字节码。

如果程序没有触发异常，那么虚拟机会使用 goto 指令跳过 catch 代码块，执行 finally 语句或者方法返回。




## 3. 字节码中的 finally


接下来在上述的代码中再加入一个 finally 代码块。


```java
// 源代码
public static void main(String[] args) {
  try {
    // dosomething
    System.out.println("enter try block");
  } catch (Exception e) {
    System.out.println("enter catch block");
  } finally {
    System.out.println("enter finally block");
  }
}
```

然后再次执行编译的命令，看看有什么不一样。


```java
// 字节码
 0 getstatic #2     <java/lang/System.out>
 3 ldc #3           <enter try block>
 5 invokevirtual #4 <java/io/PrintStream.println>
 8 getstatic #2     <java/lang/System.out>
11 ldc #5           <enter finally block>
13 invokevirtual #4 <java/io/PrintStream.println>
16 goto 50 (+34)
19 astore_1
20 getstatic #2     <java/lang/System.out>
23 ldc #7           <enter catch block>
25 invokevirtual #4 <java/io/PrintStream.println>
28 getstatic #2     <java/lang/System.out>
31 ldc #5           <enter finally block>
33 invokevirtual #4 <java/io/PrintStream.println>
36 goto 50 (+14)
39 astore_2
40 getstatic #2     <java/lang/System.out>
43 ldc #5           <enter finally block>
45 invokevirtual #4 <java/io/PrintStream.println>
48 aload_2
49 athrow
50 return
```


finally 代码块在当前版本（jdk 1.8）的 JVM 中的处理机制是比较特殊的。从上面的字节码行数中也可以明显看到，只是加了一个 finally 代码块而已，字节码指令增加了很多行。


如果再仔细观察一下，我们可以发现，在字节码指令中，有三块重复的字节码指令，分别是8~13行和40~45行，如果对字节码有些了解的同学或许已经知道了，这三块重复的字节码就是 finally 代码块对应的代码。


出现三块重复字节码指令的原因是：在 JVM 中，所有异常路径（如try、catch）以及所有正常执行路径的出口都会被附加一份 finally 代码块。也就是说，在上述的示例代码中，try 代码块后面会跟着一份 finally 的代码，catch 代码块后面也是如此，再加上原本正常流程会执行的 finally 代码块，在字节码中一共有三份 finally 代码块代码块。


而针对每一条可能出现的异常的路径，JVM 都会在异常表中多生成一条异常处理器，用来监控整个 try-catch 代码块，同时它会捕获所有种类的异常，并且在执行完 finally 代码块之后会重新抛出刚刚捕获的异常。


上述示例代码的异常表如下


```java
Exception table:
   from    to  target type
       0     8    19   Class java/lang/Exception
       0     8    39   any
      19    28    39   any
```


可以看到与原来相比异常表增加了两条，第2条异常处理器异常监控 try 代码块，第3条异常处理器监控 catch 代码块，如果出现异常则会跳转到第39行的 finally 代码块执行。

这就是 finally 一定会在 try-catch 代码块之后执行的原因了（某些能中断程序运行的操作除外）。




### 3.1 如果 finally 也抛出异常


上文说到虚拟机会对整个 try-catch 代码块生成一个或多个异常处理器，如果在 catch 代码块中抛出了异常，这个异常会被捕获，并且在执行完 finally 代码块之后被重新抛出。


那么在这里有一个额外的问题需要提及，假设在 catch 代码块中抛出了异常 A，当执行 finally 代码块时又抛出了异常 B，那么最后抛出的是什么异常呢？


如果有同学自己尝试过这个操作，就会知道最后抛出的异常 B。也就是说，在捕获了 catch 代码块中的异常后，如果 finally 代码块中也抛出了异常，那么最终将会抛出 finally 中抛出的异常，而原来 catch 代码块中的异常将会被忽略。



### 3.2 如果代码块中有 return

讲完了异常在各个代码块中的情况，接下来再来考虑一下 return 关键字吧，如果 try 或者 catch 中有 return，finally 还会执行吗？如果 finally 中也有 return，那么最终返回的值是什么？

为了说明这个问题，我编写了一段测试代码，然后找到它的字节码指令。


```java
public static int get() {
    try {
        return 1;
    } catch (Exception e) {
        return 2;
    } finally {
        return 3;
    }
}

// 字节码指令
 0 iconst_1
 1 istore_0
 2 iconst_3
 3 ireturn
 4 astore_0
 5 iconst_2
 6 istore_1
 7 iconst_3
 8 ireturn
 9 astore_2
10 iconst_3
11 ireturn
```


正如上文所述，finally 代码块会在所有正常及异常的路径上都复制一份。在这段字节码中，iconst_3 就是对应着 finally 代码块，共三份，所以即便在 try 或者 catch 代码块中有 return 语句，最终还是会会执行 finally 代码块中的内容。

也就是说，这个方法最终的返回结果是3。




## 4. finally 不执行的情况


1. 如果**未进入 try 代码块**或进入 try 代码块后一直没有执行完，比如 try 中有**无限循环**逻辑，那自然无法执行finally代码块。
1. 如果**虚拟机被终止**，比如 System.exit()，这个函数的作用是中止当前正在运行的 Java 虚拟机。如果虚拟机被中止，程序也会被终止，所以在它后面的代码都不会被执行，即便是 finally 代码块也同样不会执行。
1. **finally 代码块在被中止的守护线程内**，我们知道 Java 中的线程可以分为守护线程和用户线程。当程序中所有的用户线程都终止时，虚拟机会kill所有的守护线程来终止程序。当 try-finally 代码块存在于守护线程，而此守护线程因用户线程被销毁而终止时，该守护线程不会继续执行。假设用户线程 main 线程执行完毕后，作为守护线程的 thread 对象中的 try 代码块还没有执行结束，当用户线程执行结束后，此时已不存在用户线程，虚拟机会强制中止守护线程 thread，导致 finally 代码块不会被执行。



## 5. 小结


1. try-catch 语句的字节码指令
1. 异常表的介绍及 JVM 中异常处理流程
1. JVM 中关于 finally 代码块的特殊处理
1. 关于 return 关键字在 try-catch-finally 中的说明
1. finally不执行的几种情况

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。