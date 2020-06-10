---
layout: post
title: RocketMQ.2-NameServer不就是个注册中心
categories: [MQ]
description: RocketMQ.2-NameServer不就是个注册中心
keywords: MQ, RocketMQ, 消息队列
---

上一篇文章讲述了以`RocketMQ`源码的方式启动`NameServer`和`broker`进行单机部署及收发消息的流程，其实就是简单的`quickstart`，后端君在实际操作过之后就已经能够基于`RocketMQ`进行简单业务的消息传递，完成诸如异步消费收集日志这样的小功能了。

在`RocketMQ`的官网上还有很多不同种类的消息示例，建议想要学习`RocketMQ`的同学们先去手写一下这些`Demo`，了解过如顺序消息、广播消息、定时消息、批消息等消息类型的特性之后再去看源码。

接下来就是学习`RocketMQ`的第二天正文内容，今天我们来聊聊`NameServer`的启动流程以`及NameServer`主要功能的源码分析。

## NameServer的综述
`NameServer`是一个提供轻量级服务发现和路由的服务器，它主要包括两个功能：
- 代理管理，`NameServer`从`Broker`集群接受注册，并提供心跳机制来检查`Broker`是否活动。
- 路由管理，每个名称服务器将保存关于`Broker`集群的整个路由信息和用于客户机查询的队列信息。

我们可以将`NameServer`就当成是一个轻量级的注册中心。事实上，曾经`RocketMQ`就用过`Zookeeper`来作为注册中心，后来因为`RocketMQ`本身架构的原因不需要像`Zookeeper`那样的选举机制来选择`master`节点，所以移除了`Zookeeper`依赖，并使用`NameServer`来替代。

作为`RocketMQ`的注册中心，`NameServer`接收集群总所有`Broker`的注册，并每隔10s提供心跳机制来检查`Broker`的是否，如果`Broker`有超过120s没有更新，那么将被视为失效并从集群中移除。

了解了`NameServer`的功能之后，后端君不禁会想，`NameServer`的这些功能是如何实现的？这就需要翻阅源码了。

![2020052403](https://planeswalker23.github.io/images/posts/2020052403.png)

## Nameserver启动流程分析
上一篇文章《快速入门》中也提到过，启动`NameServer`需要找到`namesrv`包中的启动类`NamesrcStartup`类，而研究`NameServer`的启动流程也需要从这个类的`main`方法开始。
### 解析配置
启动`NameServer`的第一步是构造一个`NamesrvController`实例，这个类是`NameServer`的核心类。

```java
public static NamesrvController main0(String[] args) {
    try {
        // 构造 NamesrvController 类
        NamesrvController controller = createNamesrvController(args);
        // 初始化、启动 NamesrvController 类
        start(controller);
        String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
        log.info(tip);
        System.out.printf("%s%n", tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }
    return null;
}
```

而`createNamesrvController`方法，就是从命令行接收参数，然后将解析成配置类`NamesrvConfig`和`NettyServerConfig`。

```java
final NamesrvConfig namesrvConfig = new NamesrvConfig();
final NettyServerConfig nettyServerConfig = new NettyServerConfig();
// RocketMQ 默认端口为9876
nettyServerConfig.setListenPort(9876);
// 通过 -c 参数指定配置文件
if (commandLine.hasOption('c')) {
    String file = commandLine.getOptionValue('c');
    if (file != null) {
        InputStream in = new BufferedInputStream(new FileInputStream(file));
        properties = new Properties();
        properties.load(in);
        MixAll.properties2Object(properties, namesrvConfig);
        MixAll.properties2Object(properties, nettyServerConfig);

        namesrvConfig.setConfigStorePath(file);

        System.out.printf("load config properties file OK, %s%n", file);
        in.close();
    }
}
// 通过 -p 参数打印当前配置，并退出程序
if (commandLine.hasOption('p')) {
    InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
    MixAll.printObjectProperties(console, namesrvConfig);
    MixAll.printObjectProperties(console, nettyServerConfig);
    System.exit(0);
}
// 通过"--具体属性名 属性值"指定属性值
MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);
```

我们知道在命令行中运行`RocketMQ`是可以指定参数的，它的原理就是上面代码展示的那样。

通过`-c`命令可以指定配置文件，将配置文件中的内容解析成`java.util.Properties`类，然后赋值给`NamesrvConfig`和`NettyServerConfig`类完成配置文件的解析与映射。

如果指定了`-p`命令，则会在控制台打印配置信息，然后程序直接退出。

除此之外还可以使用`-n`参数指定`namesrvAddr`的值，这是在`org.apache.rocketmq.srvutil.ServerUtil#buildCommandlineOptions`方法中指定的参数，不在本节的讨论范围内，有兴趣的同学可以自己去翻阅源码调试下看看。

当完成配置属性的映射，就会根据配置类`NamesrvConfig`和`NettyServerConfig`构造一个`NamesrvController`实例。
```java
final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);
```

### 初始化及心跳机制
启动`NameServer`的第二步是通过`NamesrvController#initialize`完成初始化。

```java
public boolean initialize() {
    // 加载`kvConfig.json`配置文件中的`KV`配置，然后将这些配置放到`KVConfigManager#configTable`属性中
    this.kvConfigManager.load();
    // 根据`NettyServerConfig`启动一个`Netty`服务器
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
    // 初始化负责处理`Netty`网络交互数据的线程池
    this.remotingExecutor = Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
    this.registerProcessor();
    
    // 注册心跳机制线程池
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);
    
    // 注册打印KV配置线程池
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            NamesrvController.this.kvConfigManager.printAllPeriodically();
        }
    }, 1, 10, TimeUnit.MINUTES);

   // 省略以下代码...

    return true;
}
```

在初始化`NamesrvController`过程中，会注册一个心跳机制的线程池，它会在启动后5秒开始每隔10秒扫描一次不活跃的`broker`。
```java
public void scanNotActiveBroker() {
    Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        long last = next.getValue().getLastUpdateTimestamp();
        // private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;
        if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
            RemotingUtil.closeChannel(next.getValue().getChannel());
            // 将该 broker 从 brokerLiveTable 中移除
            it.remove();
            log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
        }
    }
}
```

若`broker`的`lastUpdateTimestamp`超过120秒未更新，则该`broker`会被视为失效并从集群中移除。

除了心跳机制的线程池外，还会注册另外一个线程池，它会每隔10秒打印一次所有的`KV`配置信息。

### 优雅停机
`NameServer`启动的最后一步，是注册了一个`JVM`的钩子函数，它会在`JVM`关闭之前执行。这个钩子函数的作用是释放资源，如关闭`Netty`服务器，关闭线程池等。
```java
Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
    @Override
    public Void call() throws Exception {
        controller.shutdown();
        return null;
    }
}));
```

## 小结
本文讲述了`NameServer`的作用，同时基于其启动类`NamesrvStartup`类分析了启动流程，以及心跳机制和优雅停机的实现原理。

希望可以帮助到大家。


## 参考文献
- [RocketMQ Architecture](http://rocketmq.apache.org/docs/rmq-arc/)
- [RocketMQ Namesrv启动流程](https://www.jianshu.com/p/fbbce22b7c92)


> 版权声明：本文为[Planeswalker23](https://github.com/Planeswalker23)所创，转载请带上原文链接，感谢。
> 本文已上传个人公众号，欢迎扫码关注。

![wechat](https://planeswalker23.github.io/images/wechat.png)。