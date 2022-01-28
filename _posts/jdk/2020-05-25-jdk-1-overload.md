---
layout: post
title: 我所理解的JDK系列·第1篇·编译器是如何选择重载方法的
categories: [JDK]
keywords: Java, JDK, 重载
---



本文介绍了重载、方法签名的概念，由几个示例入手介绍了编译器选择重载方法的规则与优先级，其中还向大家介绍了对象的声明类型与实际类型的概念，希望能够帮助到大家。

![cover](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643204028729-022c9216-f000-469f-b442-9ff892d4b987.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_26%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



## 1. 开篇词

今天鄙人在日常搬砖时进行了一次很普通的操作：方法调用。这时候我利用强大的 IDEA 编译器看到了所有可供选择的同名方法（重载方法），如下：

![1](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643204039599-f1462d1c-cfc4-4807-bf64-6d629e9de16a.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_31%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

看上去好像很简单的样子，当参数传入 Father 类型时调用第一个重载方法、当参数传入 Grandpa 时调第二个重载方法、当参数传入 Son 时调第三个重载方法。

但是如果我告诉你 Father、Grandpa、Son 这三个类是有继承关系的，同时我在声明入参的时候声明类型都是父类，在这种情况下，会调用哪个重载方法呢？比如：

![2](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643204045972-9ba0ac2a-0040-44ff-a710-20bd9cfa53f2.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_26%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

又比如，当重载方法为可变长参数（不要告诉我你不知道啥是可变长参数）及 Object 时，会选择哪一个呢？

![3](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643204054635-9d23f557-4714-44ea-891d-5d3b00752d9e.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_24%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

还有另外几种情况，本文也会逐一介绍，下面就一起徜徉在知识的深渊中吧！




## 2. 重载与方法签名


首先来明确一下重载的概念：重载(Overload)就是指**在同一个类中定义相同名称、不同参数类型或参数个数的方法**。而重载方法的返回类型，可以相同也可以不相同。

或者我们可以说，重载就是指在一个类中的两个方法具有不同的方法签名。




> The Java programming language supports _overloading_ methods, and Java can distinguish between methods with different _method signatures_.
> 译文：Java 语言支持方法重载，它可以区分具有不同方法签名的方法。
> 原文来源：[Defining Methods ](https://docs.oracle.com/javase/tutorial/java/javaOO/methods.html)



那么上文中所说的方法签名是什么意思呢？在 Oracle 官网文档中关于 [Defining Methods](https://docs.oracle.com/javase/tutorial/java/javaOO/methods.html) 的文章中也定义了方法签名的描述。




> **Definition:** Two of the components of a method declaration comprise the _method signature_—the method's name and the parameter types.
> 译文：方法声明的两个组件组成了方法签名——方法名和参数类型。



也就是说，一个方法的**方法名和参数类型（参数列表）**构成了它的方法签名。对于一个类中的两个方法，如果它们拥有相同的方法名和不同的参数个数或不同的参数类型，那么它们就是重载方法。


例如，下面的 Test 类的两个 overloadMethod 方法就是重载方法。


```java
public class Test {
    public void overloadMethod(int s) {
        System.out.println("int");
    }

    public Object overloadMethod(Object obj) {
        System.out.println("Object");
        return obj;
    }
}
```





## 3. 如何选择重载方法

现在我们已经知道重载方法的概念，以及编译器会根据入参选择合适的重载方法来执行，那么回到本文开始的问题，在那几种情况下，编译器会选择哪个重载方法来执行呢？我们逐一来进行实操与分析。

### 3.1 方法重载示例1——声明类型

第一种情况，声明类型与实际类型不同。


我在 OverloadDemo2 这个类中声明了两个重载方法 test，它们分别接收 Object 类和 String 类的参数。在 main 方法中，我又声明一个 Object 对象，并创建一个 String 对象赋值给它。

![4](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643204061411-71d68422-319d-4326-b910-8c7bbd71814d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_22%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)


执行这个程序，结果输出了：`Object`。也就是说，对于一个声明为 Object 类但被赋予 String 类的对象，编译器在选择重载方法时会将它视为 Object 类。


这就告诉我们，在选取重载方法的时候，对于参数类型的判定是基于参数的**声明类型**的。


#### 3.1.1 声明类型


为了介绍什么是声明类型，我们先来看一行代码。


```text
Object obj = new String("1");
```


对于 obj 对象而言，它被声明为 Object 类，然后又创建了一个 String 类型的对象并赋值给它。在这其中，Object 类型被称为 obj 变量的声明类型（或静态类型），而 String 类型则被称为它的实际类型。


变量本身的声明类型是不会发生改变的，声明类型是在编译器就能确定的。而变量的实际类型在编译器是不确定的，在运行期才能确定。


而**编译器在重载时是根据参数的声明类型而不是实际类型作为判定依据的**，我们可以通过字节码来验证这个结论。


```text
public static void main(java.lang.String[]);
    Code:
       0: new           #6  // class java/lang/String
       3: dup
       4: ldc           #7  // String 1
       6: invokespecial #8  // Method java/lang/String."<init>":(Ljava/lang/String;)V
       9: astore_1
      10: aload_1
      11: invokestatic  #9  // Method test:(Ljava/lang/Object;)V
      14: return
```


上面通过 `javap -c 类名` 的反编译命令得到的字节码就是主方法的字节码，看到第11行通过 invokestatic 指令调用了静态方法 test，而这个 test 方法的参数类型就是 Object。


> 示例代码中将 test 方法设置为静态方法，所以会用 invokestatic 指令，它的作用就是调用静态方法。
>
> `(Ljava/lang/Object;)V` 是 test 方法的方法描述符，括号中的 Object 类是方法的参数，V 是指方法的返回值 void。

#### 3.1.2 静态分派


现在我们已经知道编译器在编译时就可以确定要调用的重载方法，同时它是根据参数的声明类型来确定具体调用的重载方法的。


我们又把所有依赖静态类型来定位方法执行版本的分派动作成为静态分派。


> 引自《深入理解 Java 虚拟机》（第二版）8.3.2 分派

#### 3.1.3 开篇词场景解析

在知道声明类型的相关概念后，我想在开篇词中的第一个场景大家心里应该都已经有答案。

```text
Grandpa fatherAsGrandpa = new Father();
Grandpa sonAsGrandpa = new Son();
```

这两个对象的声明类型都是 Grandpa 类型，所以在执行重载方法时会寻找以 Grandpa 类型为入参的重载方法，即图1中的第二个重载方法。



### 3.2 方法重载示例2——继承与自动拆装箱

#### 3.2.1 继承示例

在了解了静态分派的概念后，接下来我们来看第三个示例代码，在 OverloadDemo3 类中，我声明了几个重载方法，并且在调用 test 方法时传入了一个 `new Integer(1)` 类型的值。

```java
public class OverloadDemo3 {

    public static void test(Integer o) {
        System.out.println("Integer");
    }

    public static void test(Object s) {
        System.out.println("Object");
    }

    public static void main(String[] args) {
        test(new Integer(1));
    }
}
```

当存在两个重载方法，入参一个是 Integer 类型，一个是 Object 类型时，执行 main 方法后，输出的内容是：`Integer`。这证明虽然 `Integer is a Object`，但是在 Integer 作为入参的重载方法存在时，该重载方法还是会优先于父类 Object 作为入参的重载方法被调用。

当去掉以 Integer 作为入参的重载方法后，再次运行 mian 方法，输出的内容是：`Obejct`。证明当以 Integer 作为入参的重载方法不存在时，编译器会将 Integer 入参转型成父类 Object 类型，从而找到了入参为 Object 类型的重载方法。

如果父类还有父类，那么将在继承关系中向上递归地去查找符合类型的重载方法。

#### 3.3.3 自动拆装箱

考虑到 Integer 类型的特殊性（作为原始类型的包装类），我不得不再做一个实验，即再新增一个原始类型的重载方法，并且将入参改成 int 类型：


```java
public class OverloadDemo3 {

    public static void test(int o) {
        System.out.println("int");
    }

    public static void test(Integer o) {
        System.out.println("Integer");
    }

    public static void main(String[] args) {
        test(1);
    }
}
```


当执行 main 方法后，输出的值是：`int`。毫无疑问这是正确的，传入 int 类型的参数，执行需要 int 类型参数的重载方法。

而如果注释掉需要 int 类型参数的重载方法，再次运行程序，会发现这次的输出变成了：`Integer`。这是因为在编译时发生了自动装箱，int 类型的值被自动装箱成 Integer 类型，这样就会去调用需要 Integer 类型参数的重载方法。

所以当原始类型的入参不存在重载方法时，编译器会考虑将原始类型进行自动装箱成包装类，再去寻找重载方法。

#### 3.3.3 可变长参数

还记得在开篇词中我还提出了一个场景，就是当可变长参数也存在于重载方法中时，编译器该如何选择？下面来进行实验，我们已经知道在原始类型、包装类型、父类这三种场景中，父类作为入参的重载优先级是最低的，所以我们先将父类作为重载方法入参与将可变长参数作为重载方法入参进行，也就是开篇词中提到的第二个场景：

```java
public class OverloadDemo3 {

    public static void overloadMethod(int... s) {
        System.out.println("int...");
    }

    public static void test(Object o) {
        System.out.println("Object");
    }

    public static void main(String[] args) {
        test(1);
    }
}
```

当执行 main 方法后，输出的值是：`Object`。这证明父类作为入参的重载方法优先于高于可变长参数作为入参的重载方法。

如果注释掉需要 Object 类型参数的重载方法，再次运行 main 方法，会发现这次的输出变成了：`int...`。

这其实是因为编译器在进行自动装箱后也无法找到符合类型的重载方法，然后将参数转化为一个数组，即寻找可变长参数的重载方法。

由此我们可以知道，可变长参数的重载优先级是最低的。


### 3.3 选取重载方法优先级


综上所述，我们可以得到编译器寻找重载方法的规则与优先级：


1. 根据参数的声明类型（静态类型）寻找重载方法
1. 若为原始类型先考虑自动拆装箱
1. 考虑参数的父类型（在继承关系中向上递归地去查找符合类型的重载方法）
1. 考虑可变长参数

最后，如果还是没有找到重载方法，那么就会发生编译期错误。



## 4. 小结

本文介绍了重载、方法签名的概念，由几个示例入手介绍了编译器选择重载方法的规则与优先级，其中还向大家介绍了对象的声明类型与实际类型的概念，希望能够帮助到大家。

在日常工作中能避免类似重载方法的坑。



## 5. 参考资料

- 深入理解 Java 虚拟机（第三版）

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。