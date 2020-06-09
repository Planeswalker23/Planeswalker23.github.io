---
layout: post
title: 什么情况下 finally 代码块不会执行
categories: [Java]
description: 什么情况下 finally 代码块不会执行
keywords: Java, finally
---

## finally 的作用
finally 是 Java 异常处理机制的组成部分，它保证了代码块中的代码一定要被执行。通常我们会将资源释放的代码写在 finally 代码块中，如 I/O 的关闭、数据库连接的关闭等。

但是也会存在 finally 代码块中的代码不会执行的情况，本文就是总结了这些情况。

## finally 代码块不执行的情况
> 之前在网上的帖子看到有一种情况是叫做”不进入 try 代码块“，这不废话吗，不进入 try 当然不会执行 finally 代码块。所以本文讨论的是进入 try 代码块后 finally 代码块不执行的情况。

### 1. 虚拟机被终止 `System.exit()；`
`System.exit(0)`的作用是中止当前正在运行的 Java 虚拟机。如果虚拟机被中止，程序也会被终止，所以在它后面的代码都不会被执行，即便是 finally 代码块也同样不会执行。

```java
private static void demo1() {
    try {
        System.out.println("执行 try 代码块");
        System.exit(0);
    } finally {
        System.out.println("开始执行 finally 代码块");
    }
}

/**控制台输出：

执行 try 代码块

**/
```

### 2. try 代码块无限循环
这是由于业务逻辑的原因，一直在执行 try 代码块的业务并且永远无法终止，try 代码块都没执行完，自然无法执行 finally 代码块。

```java
private static void demo2() {
    try {
        while (true) {
            // do something
            System.out.println("执行 try 代码块");
        }
    } finally {
        System.out.println("开始执行 finally 代码块");
    }
}

/**控制台输出：

执行 try 代码块
执行 try 代码块
...（一直输出）

**/
```

### 3. 在被中止的守护线程内
Java 中的线程可以分为守护线程和用户线程。当程序中所有的用户线程都终止时，虚拟机会 kill 所有的守护线程来终止程序。

当 try-finally 代码块存在于守护线程，而此守护线程因用户线程被销毁而终止时，该守护线程不会继续执行。

假设用户线程（main线程）执行完毕后，作为守护线程的 thread 对象中的 try 代码块还没有执行结束，当用户线程执行结束后，此时已不存在用户线程，虚拟机会强制中止守护线程 thread，导致 finally 代码块不会被执行。

```java
private static void demo3() {
    Thread thread = new Thread(() -> {
            try {
                System.out.println("执行 try 代码块，等待2s，等待主线程执行结束...");
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.println("开始执行 finally 代码块");
            }
        });
        thread.setDaemon(true);
        thread.start();
}

/**控制台输出：

执行 try 代码块，等待2s，等待主线程执行结束...

**/
```

## 参考博客
[finally块不被执行的情况总结](https://www.cnblogs.com/yadiel-cc/p/11296567.html)

> 版权声明：本文为[Planeswalker23](https://github.com/Planeswalker23)所创，转载请带上原文链接，感谢。<br>
> 本文已上传个人公众号，欢迎扫码关注。

![wechat](https://planeswalker23.github.io/images/wechat.png)