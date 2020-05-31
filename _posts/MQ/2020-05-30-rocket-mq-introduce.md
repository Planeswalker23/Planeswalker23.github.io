---
layout: post
title: RocketMQ 实战(0)与君初相识
categories: [MQ]
description: RocketMQ 实战(0)与君初相识
keywords: MQ, RocketMQ, 消息队列
---

## 1. 什么是 RocketMQ
官方对于`RocketMQ`的定义是
> Apache RocketMQ is a distributed messaging and streaming platform with low latency, high performance and reliability, trillion-level capacity and flexible scalability.——[RocketMQ Architecture](https://rocketmq.apache.org/docs/rmq-arc/)

`RocketMQ`是一个具有低延迟、高性能、高可用、万亿级容量和灵活的可伸缩性的分布式消息传递和流平台。

![2020053101](https://planeswalker23.github.io/images/posts/2020053101.png)

从`RocketMQ`的架构图可以看到，它是由`NameServer, Broker, Producer, Consumer`四种角色组成的，每一种角色都可以进行水平扩展而不会出现单点故障，所以它天然的支持分布式。

## 2. RocketMQ 角色介绍
在刚开始使用`RocketMQ`时相信大家都会跟着官网的 [Quick Start](https://rocketmq.apache.org/docs/quick-start/) 案例来实现一个`demo`，我也一样。但是完事之后，感觉好像跟之前没有什么区别，根本不知道`demo`里面每个步骤中启动的是个什么玩意儿。

所以在这里先记录一下`RocketMQ`各个角色的介绍和功能，在知道这个之后，我对`RocketMQ`的架构和执行流程有了更好的理解。

### 2.0 案例
在介绍`RocketMQ`之前，先来看一个例子。

假设我在京东商城买了一个东西，仓库会把相应的商品打包好，交给京东物流，次日京东物流会指派快递点的某个快递员把商品交到我手上。

在这个过程中有这样几个角色：京东物流、快递点、寄件人仓库、收件人我。

就分别代表了`NameServer, Broker, Producer, Consumer`。

### 2.1 NameServer
先来说说京东物流`NameServer`，京东物流负责的工作是管理所有的快递点，可以把它看成是一个指挥部，它知道所有快递点的信息，它会把商品分发给适合的快递点，让快递点负责交付货物。

### 2.2 Broker
而作为快递点的`Broker`，负责的就是商品的暂存和传输。

即消息暂存和消息传输。

### 2.3 Producer
消息生产者，即寄件人仓库。

### 2.4 Consumer
消息消费者，即收件人我。

## 3. 消息队列的使用场景
下面再来说说消息队列`MQ(Message Queue)`的主要使用场景。
### 3.1 异步处理
当一个流程十分臃肿，而其中决定性的节点比较少的时候，就可以考虑将非决定性的节点改为使用消息队列发送异步消息实现，这样可以提高整个流程的效率。除此之外，还能将优先的服务器资源用来处理更多的决定性节点业务。

### 3.2 流量削峰
不知道大家有没有经历过这样的场景：大学时选课，经常会出现进不去选课页面的情况。

这就是流量超过了程序上限，最终导致整个系统都不可用。而消息队列就可以很大程度地防止出现这样的情况，它可以降低并发的请求，把流量控制在服务器能够接受的范围。

虽然整个系统瘫痪的情况基本是不会出现了，但是选课响应慢，还是会发生...

### 3.3 服务解耦
在一个微服务化的程序中，主业务可能会有很多个下游业务，而每个下游业务需要的参数可能是不同的，并且下游业务可能会经常更新，这样就存在了服务间耦合性过于紧密的情况。

如果在此时引入消息队列，主业务只在一个队列中发送一个完整的数据，而其他下游业务通过订阅这个主题来完成自己的业务，这样无论下游业务做怎样的修改，参数如何变化，只要队列中数据时完整的，就不需要修改主业务。就实现了服务间的解耦。

## 4. 小结
本文记录了我在学习`RocketMQ`前不明白、不清楚的一些点，比如各个角色的作用，以及消息队列能够解决的问题。

不过当我多写`demo`之后，这些问题也就迎刃而解，这里只是做个记录，权当课前预习了。
