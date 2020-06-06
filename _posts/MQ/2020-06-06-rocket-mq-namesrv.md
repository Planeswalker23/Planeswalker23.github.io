---
layout: post
title: RocketMQ——1. 快速入门
categories: [MQ]
description: RocketMQ——1. 快速入门
keywords: MQ, RocketMQ, 消息队列
---

学习`RocketMQ`的第一天，应该从官网开始。在官网上有很多`QuickStart`的例子，在手打完官网上的例子后，相信就会对`RocketMQ`有了初步的认识，这一节就来介绍一下如何使用`RocketMQ`进行消息的收发。

## 0. 版本说明
使用`RocketMQ`需要如下有硬件的要求：
- 64位操作系统
- JDK 1.8+
- Maven 3.2.x
- Git
- 4GB+ 硬盘空间，broker 存储需要

## 1. 获取源码
打开`RocketMQ`在`Github`上的[主页](https://github.com/apache/rocketmq)，获取仓库地址。然后在本地电脑上克隆本仓库。

```git
git clone https://github.com/apache/rocketmq.git;
```

## 2. 启动 Namesrv 和 broker
打开项目后，第一步要做的是启动`nameserver`，这是`RocketMQ`的路由中心，它提供轻量级服务发现和路由，主要的作用是存储路由信息，管理`broker`节点，包括路由的查找、注册和删除。

在`RocketMQ`工程的`namesrv`包中找到入口类`org.apache.rocketmq.namesrv.NamesrvStartup`，运行这个类的`main`函数，发现报错了。

```
Please set the ROCKETMQ_HOME variable in your environment to match the location of the RocketMQ installation
```
这个报错是因为在为`nameserver`设置相关配置时没有设置成功。

```java
if (null == namesrvConfig.getRocketmqHome()) {
    System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation%n", MixAll.ROCKETMQ_HOME_ENV);
    System.exit(-2);
}
```

`ROCKETMQ_HOME`环境变量主要用于设置`nameserver`的配置，只需要将包含`conf`配置目录的这个路径赋值给环境变量`ROCKETMQ_HOME`即可，如下图。

![2020060601](https://planeswalker23.github.io/images/posts/2020060601.png)

再次运行`main`函数，就会发现启动成功。

```
The Name Server boot success. serializeType=JSON
```

接下来要启动的是`broker`，它主要用于消息存储，接收和发送。

在`RocketMQ`工程的`broker`包中找到入口类`org.apache.rocketmq.broker.BrokerStartup`，运行`main`函数，如果发现与上述一样的报错，重新设置环境变量即可，运行成功的输出如下。

```
The broker[fandeMac-mini.local, 172.16.26.129:10911] boot success. serializeType=JSON
```

至此，`RocketMQ`的路由中心和接收发消息的服务器就启动成功了，我们可以通过`nameserver`和`broker`来进行消息传递了。

## 3. 启动 Producer 发送消息
找到`example`包的`org.apache.rocketmq.example.quickstart.Producer`类，这是一个最简单的消息生产者，我们来看一下它的源码。

```java
public class Producer {
    public static void main(String[] args) throws MQClientException, InterruptedException {

        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        producer.setNamesrvAddr("127.0.0.1:9876");
        producer.start();

        for (int i = 0; i < 1000; i++) {
            try {
                Message msg = new Message("TopicTest","TagA",("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));

                SendResult sendResult = producer.send(msg);
                System.out.printf("%s%n", sendResult);
            } catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }

        producer.shutdown();
    }
}
```

1. 使用`DefaultMQProducer`类来创建生产者实例，并指定消息组`Group`和路由中心地址。
2. 启动生产者实例。
3. 创建消息，指定`Topic`和`Tag`（用于区分消息的类别）。
4. 发送消息。
5. 关闭生产者实例。

这样就实现了`RocketMQ`的发送消息。

## 4. 启动 Consumer 消费消息
找到`example`包的`org.apache.rocketmq.example.quickstart.Consumer`类，这是一个最简单的消息消费者，我们来看一下它的源码。

```java
public class Consumer {

    public static void main(String[] args) throws InterruptedException, MQClientException {

        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");
        consumer.setNamesrvAddr("127.0.0.1:9876");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.subscribe("TopicTest", "*");
        consumer.registerMessageListener(new MessageListenerConcurrently() {

            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        consumer.start();
        System.out.printf("Consumer Started.%n");
    }
}
```

1. 使用`DefaultMQPushConsumer`类来创建消费者实例，并指定消息组`Group`、路由中心地址、消费模式、消息类别。
2. 注册消息监听器，监听消息，消费消息，返回消费成功标识。
3. 启动生产者实例。

至此，我们就完成了`RocketMQ`的快速入门。

> 版权声明：本文为[Planeswalker23](https://github.com/Planeswalker23)所创，转载请带上原文链接，感谢