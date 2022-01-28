---
layout: post
title: 我所理解的其他问题·第9篇·由Dubbo直连引出的File路径问题
categories: [Java]
keywords: Java, File
---


你好，有幸相见。

从九月开始，我决定发起「每周一博」的目标：每周至少发布一篇博客，可以是各种源码分析研读，也可以是记录工作中遇到的难题。

在经过了一段时间漫无目的的学习之后，我发现那样用处好像不大，看过的东西过段时间就忘了，而且也没有做什么笔记。

“凡所学，必有所输出。”我认为这才是最适合我的学习方式，这也是「每周一博」活动的来由，朋友们，如果你也觉得经常会忘记以前看过的东西，一起加入这个活动吧。

----

## 1. Dubbo 直连的几种方式
在 Dubbo 官网的[直连提供者](http://dubbo.apache.org/zh-cn/docs/user/demos/explicit-target.html)这篇文章中向开发者介绍了绕过注册中心，直接通过ip+端口指定服务提供者的直连方式。

需要注意的是：官方建议直连方式仅**开发测试**时使用！

Dubbo 直连有4种实现方式：
### 1.1 通过 XML 配置
```java
<dubbo:reference id="xxxService" interface="com.alibaba.XxxService" url="dubbo://localhost:20890" />
```

### 1.2 通过注解方式配置
```java
    @Reference(check = false, timeout = 60000, url = "localhost:20890")
    private XxxService xxxService;
```

### 1.3 通过 -D 参数指定接口
```java
java -Dcom.alibaba.XxxService=dubbo://localhost:20890 -jar Demo.jar
```
### 1.4 通过 -D 参数指定映射文件
```java
java -Ddubbo.resolve.file=xxx.properties  -jar Demo.jar
```
然后在 `xxx.properties` 文件中加入如下配置

```java
com.alibaba.xxx.XxxService=dubbo://localhost:20890
```

## 2. 直连调试
在项目中有很多个 Dubbo 接口，而且在线上的某种环境中是没有 zk 注册中心的，出于切换 Dubbo 接口调用方式以及方便管理所有 Dubbo 接口的考虑，我选用的是第四种——通过映射文件实现直连。

> 注意：本项目使用的 Dubbo 版本是2.6.0。

### 2.1 @Reference注入为null
刚开始调试的时候，我发现在有注册中心的环境下发现在消费者中无法通过 `@Reference` 注解注入 Dubbo 接口，开始我以为是 Dubbo 版本的问题，后来将生产者与消费者的 Dubbo 版本号改为一致才发现不是这个原因。自己研究了好久，实在找不到原因了于是寻求身为 `Dubbo committer` 的同事的帮助才知道原来是在启动类中没有配置 `@EnableDubboConfiguration` 注解启用 Dubbo 导致的，惭愧惭愧。

### 2.2 映射文件路径问题排查
在解决了上面的问题之后，首先我在消费者的配置文件中将 `Spring.dubbo.registry.address` 改为 `N/A`，表明不使用注册中心，然后在 `resource` 目录下新增映射文件`dubbo.properties`，同时在映射文件中指定直连接口：

```java
com.alibaba.xxx.XxxService=dubbo://localhost:20891
```

最后在编译器的 `Run/Debug Configuration` 中添加如下的启动参数配置：

![1](https://www.yuque.com/api/filetransfer/images?url=http%3A%2F%2Fjavageekers.club%2Fupload%2F2020%2F09%2F2020090901-ae4780e37c6148a3a9b6c41e7e5f5c7d.png&sign=cc793da9f38546e60b70d27b4e98583e58dcff17b3c5b264d2acdf3f8475980f)

然后在启动之后却出现了让我差异的报错：

![启动报错](https://www.yuque.com/api/filetransfer/images?url=http%3A%2F%2Fjavageekers.club%2Fupload%2F2020%2F09%2F2020090902-db0ab7b65f6d40fcb18f165700b42a86.png&sign=e4c9f91e2b98907e6e8a4f01dbc8fa34c4944b47978a53358cee04ffa25ad377)

`dubbo.properties` 文件不存在，开始我以为是 target 目录中没有把这个文件编译进去，因为之前也发生过 idea 编译后某些类和资源文件没有更新的情况，但事实却不是如此，target目录中 `dubbo.properties` 好端端的在那里。

然后我把编译器 `Run/Debug Configuration` 配置中的文件名改成了它的绝对路径

```java
-Ddubbo.resolve.file=/Users/fan/workspace/Windfall/src/main/resources/dubbo.properties
```

发现程序可以正常启动，这样问题就很明显了，绝对是文件路径的锅！

我循着报错的异常栈，在报错的地方打了一个断点进行调试，在新建 `FileInputStream` 对象打开文件的地方执行了获取 `File` 对象绝对路径的方法。

![Debug1](https://www.yuque.com/api/filetransfer/images?url=http%3A%2F%2Fjavageekers.club%2Fupload%2F2020%2F09%2F2020090903-6a46783a549d467391f986b3532eff7d.png&sign=f6fc8cccbb2216a28bddf63e6da370cc8394bd4f65e35a15fe38ce109e1c7292)

`dubbo.properties` 的绝对路径竟然是这么个玩意，本项目的项目名是 `Windfall`，相当于程序会从当前工程目录下寻找映射文件，而我创建文件的路径是 `Windfall->src->main->resource`，这当然会找不到。

于是我把启动参数改成了：

```java
-Ddubbo.resolve.file=src/main/resources/dubbo.properties
```

果然，在编译器中程序正常启动了。

问题就这样解决了，但是让我疑惑的是，为什么我指定了映射文件名之后，它的路径却变成了从当前工程目录下进行寻找。然后我翻了翻 `File#getAbsolutePath()` 方法，发现这个方法中执行的逻辑其实是这样的：

```java
public String resolve(File f) {
    if (isAbsolute(f)) return f.getPath();
    return resolve(System.getProperty("user.dir"), f.getPath());
}
```

如果文件本来就是以绝对路径创建的，直接返回它的 path，否则在 path 前加上 `user.dir` 作为该文件的绝对路径，而 `user.dir` 环境变量就是指用户当前工作目录。

### 2.3 关于 user.dir
还有需要注意的是，在命令行中启动 jar 包，`user.dir` 的路径就是执行启动命令的目录。

为此，我写了一个测试程序。

```java
public class demo{
    public static void main(String[] args) {
        System.out.println(System.getProperties("user.dir"));
    }
}
```

先用 `javac demo.java` 命令编译程序，然后执行`java demo`命令启动程序，打印 `user.dir` 的信息，我在多个目录中都尝试了这个程序，发现输出的结果都不相同。

![user.dir测试程序](https://www.yuque.com/api/filetransfer/images?url=http%3A%2F%2Fjavageekers.club%2Fupload%2F2020%2F09%2F2020090904-e9f9cd731b98401da74aa928278ec44a.png&sign=d0ac4e472cdd59c44bb92d02dec95c1146007347e063e7e40df820b04f9ed990)

从程序执行的结果来看，`user.dir` 的结果就是执行启动命令的路径。所以，在服务器上使用 Dubbo 进行直连的时候，需要注意 `dubbo.properties` 文件需要放在执行启动脚本的目录。

## 3. 小结
本文讲述了 Dubbo 直连的几种实现方式以及我在进行直连调试的时候遇到的几个问题：
- `@EnableDubboConfiguration` 注解启用 Dubbo
- `new File()` 路径问题
- `user.dir` 值问题

## 4. 参考资料
- [Dubbo 官网直连提供者](http://dubbo.apache.org/zh-cn/docs/user/demos/explicit-target.html)
- [java中user.dir到底是啥？](https://www.jianshu.com/p/28693aad491b)

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。