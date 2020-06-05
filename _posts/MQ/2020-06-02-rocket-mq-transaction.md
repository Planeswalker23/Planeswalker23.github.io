---
layout: post
title: 「RocketMQ-1」如何实现分布式事务
categories: [MQ]
description: 「RocketMQ-1」如何实现分布式事务
keywords: MQ, RocketMQ, 消息队列
---

相信小伙伴们在日常开发学习中都听说过分布式事务，今天就来聊聊 RocketMQ 如何实现分布式事务。

在RocketMQ 官方给出的[Transaction example](https://rocketmq.apache.org/docs/transaction-example/)示例中向大家介绍了事务消息，可以将它视为一个两阶段提交的消息（半消息，`half message`）从而实现了分布式系统中的最终一致性，也就是要么都成功，要么都失败。

首先来看一个例子。

假设在一个分布式系统中有两个子服务，分别用于下单和支付。


> 版权声明：本文为[Planeswalker23](https://github.com/Planeswalker23)所创，转载请带上原文链接，感谢