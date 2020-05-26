---
layout: post
title: 「JVM」原始类型 boolean 在 JVM 中的讨论
categories: [JVM]
description: 「JVM」原始类型 boolean 在 JVM 中的讨论
keywords: JVM, 字节码，boolean
---

## 导言
在开始学习`JVM`字节码之后，遇到了一个有意思的问题，下面这段代码，会输出什么：
````
public class Foo {
    public static void main(String[] args) {
        boolean flag = true;
        if (flag) {
            System.out.print("A");
        }
        if (flag == true) {
            System.out.print("B");
        }
    }
}
````
这个问题的答案很显然——会输出`AB`。

接下来重点来了，如果将 2 赋值给`flag`变量，会输出什么呢？如果将`flag`赋值为 3 呢？

要知道这两个问题的答案，我们得知道在`JVM`中`boolean`类型的变量是如何表示的，以及在这两个`if`语句中到底进行了怎样的判断。

## boolean 类型在 JVM 中是如何表示的
在命令行中输入`javap -c Foo`，得到反编译的字节码如下:

````
Compiled from "Foo.java"
public class geektime.part1.Foo {
  public geektime.part1.Foo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: iconst_1
       1: istore_1
       2: iload_1
       3: ifeq          14
       6: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       9: ldc           #3                  // String Hello, Java!
      11: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      14: iload_1
      15: iconst_1
      16: if_icmpne     27
      19: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      22: ldc           #5                  // String Hello, JVM!
      24: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      27: return
}
````

在字节码文件的`main`方法中，第0~2行完成了将一个`int`类型的常量赋值给了第一个变量`flag`的操作。于是我们得到了结论：
- **在`JVM`中`boolean`类型的变量是用数字 0 和 1 来表示的。`false`用 0 表示，`true`用 1 表示**。

### if (flag) 分析
然后继续看字节码文件，接下来的字节码就是代表第一个判断语句`if (flag)` 语句，从3~11行都属于第一个`if`代码块，首先是`ifeq`。

![7A29389B-DD73-4675-9DF2-B36FB06BD56E.png](../../images/posts/2020-05-26-3.png)

>`ifeq`助记符的作用是：当栈顶 int 型数值等于0时跳转。这里跳转的意思就是不执行语句块中的代码。

此时的栈顶 int 型数值就是刚刚被赋值给 flag 的1，所以在这里`ifeq 14`的意思就是当 flag 等于 0 的时候跳转到14行，由于第14已经不属于 if 语句的范围了，所以这里的跳转是不执行 if 语句的意思。**也就是说，`if(flag)`中是判断 flag 的值，当 flag 值不等于0的时候才执行 if 中的语句**。

### if (flag == true) 分析
接下来继续看第二个判断语句`if (flag == true)`语句，是在字节码的14~24行。前面两行是将一个`int`类型的数值1和`flag`变量推送到栈顶，可以理解为把接下来将要进行比较的`true`放入将要进行比较的一个“容器”，然后是`if_icmpne`助记符。

![7A29389B-DD73-4675-9DF2-B36FB06BD56E.png](../../images/posts/2020-05-26-4.png)

>`if_icmpne`助记符的作用是：比较栈顶两int型数值大小，当结果不相等时跳转。

现在栈顶两`int`型的数值是刚刚推送的`true`也就是1，以及`flag`变量，所以`if_icmpne`助记符是比较这两个数值，如果他们相等，执行`if`语句的内容。**也就是说，`if(flag == true)`中是进行`flag`和`true`的判断，当它们相等时执行`if`中的语句**。

## 为什么编译器提示可以简化
关于上述的示例代码，编译器或许会在第二个`if`语句提醒你可以`Simplify`，当你执行简化以后你会发现它将`if (flag == true)`变成了`if (flag)`，如下图所示。

![559A0D7D-7954-4A7E-8D64-EC6A352F7AD1.png](../../images/posts/2020-05-26-5.png)

在没有`Simplify`之前，这两个 if 语句如果要执行，第一个的条件是 flag 不等0，第二个的条件是 flag 等于1。这是两个完全不同的比较嘛！但是为什么在编译器中可以画上等号呢？

![timg.jpeg](../../images/posts/2020-05-26-6.jpeg)

当我们已经了解过`boolean`类型的变量在虚拟机中的表现形式之后，答案显而易见。首先这里的`flag`是一个`boolean`类型的变量，它只有`true`和`false`两种值，即在虚拟机中只有0和1两种值。所以对于一个`boolean`类型的变量，不等0就代表了它一定等1，所以编译器才会发出可以简化的提示。

## 如果将 flag 的值赋为其他整数型值
我们知道在正常情况下编译器不会接受将2这一个数字赋值给`boolean`类型变量的这么一个操作，但是我们可以通过一些其他的工具（如`asmtools`）来实现这个操作。

1. 将`flag`的值赋为2。在字节码中就是将`iconst_2`赋值给`flag`时，输出为空。

2. 将`flag`的值赋为3。在字节码中就是将`iconst_3`赋值给`flag`时，输出为`AB`。

3. 如果再多做几个实验，将`flag`的值赋为4，输出为空；将`flag`的值赋为5，输出为`AB`......
如果我们将这些整数都转化为二进制，即2=0010，3=0011，4=0100，5=0101。
当二进制末尾为0时，无输出，当二进制末尾为1时，输出`AB`。

由此我们可以得出结论：如果将其他整数类型的值赋值给一个`boolean`类型的变量，虚拟机会取此整数值二进制的最后一位。

## 总结
1. `boolean`类型的变量在虚拟机中有两种表现形式：`true`用1表示，`false`用0表示。
2. 当虚拟机执行`if(boolean)`的条件判断时，实际进行的判断是该布尔值是否不等于0，当该布尔值不等于0时执行`if`语句中的内容。
3. 当虚拟机执行`if(boolean==true)`的条件判断时，实际进行的判断时该布尔值是否等于1，当该布尔值等于1时执行`if`语句中的内容。
4. 编译器提示`if(boolean==true)`可以简化为`if(boolean)`的原因是在虚拟机中布尔值只有两种值，执行不等于0的判断与执行等于1的判断是等价的。
5. 若在虚拟机中将其他`int`值赋值给一个布尔类型的变量，虚拟机会取此整数值二进制的最后一位。