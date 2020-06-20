---
layout: post
title: RocketMQ.4-基于事务消息解决分布式事务
categories: [MQ]
description: RocketMQ.4-基于事务消息解决分布式事务
keywords: MQ, RocketMQ, 消息队列
---

相信很多小伙伴们在日常开发学习过程中或多或少都听说过分布式事务，恰巧后端君之前在看`RocketMQ`源码，今天就来聊聊`RocketMQ`基于事务消息解决分布式事务的原理。

![](https://planeswalker23.github.io/images/posts/2020060701.png)

## 1. 分布式事务简述
事务相信大家应该都知道，它是指逻辑上的一组操作，组成这组操作的各个单元，**要么都成功，要么都失败**，同时事务应该满足`ACID`四个特性。

- `A, 原子性(Atomicity)`：一个事务是一个不可分割的工作单位，事务中包括的操作要么都做，要么都不做。
- `C, 一致性(Consistency)`：事务应确保数据库的状态从一个一致状态转变为另一个一致状态。
- `I, 隔离性(Isolation)`：多个事务并发执行时，一个事务的执行不应影响其他事务的执行。
- `D, 持久性(Durability)`：已被提交的事务对数据库的修改应该永久保存在数据库中。

在分布式系统中，一个流程可能是由多个系统共同完成的，这些系统部署在不同的服务器上，同时这些系统可能对应着多个数据库，在这样的情况下，如果其中一个系统出现了故障，就应该导致整个流程失败，所有已更新的操作全部回滚，只有所有系统都成功了，这次操作才能够被持久化。

而分布式事务，就是为了保证在分布式系统中数据的一致性。

## 2. 分布式事务产生的原因
首先我们通过一个例子来说明一下分布式事务产生的原因。

假设在一个分布式系统中有两个子服务，分别用于下单和新增用户积分。当用户选择了商品，下单支付成功后，会依据商品价格定量的增加用户积分。在这个过程中，增加加分这一步并不是下单流程必须的，所以通过消息队列来进行异步执行是比较合理的。

![](https://planeswalker23.github.io/images/posts/2020062001.png)

在上述的整个流程中，基于下单和新增积分这两个步骤一共可能会有四种情况发生。
1. 下单成功，新增积分成功。
2. 下单失败，新增积分失败。
3. 下单成功，新增积分失败。
4. 下单失败，新增积分成功。

情况1和2自然不必多说，那么3和4是我们需要进一步分析的情况。

首先来说说下单成功，新增积分失败的情况，当用户下单后，订单系统发送一个消息到消息队列，然后积分系统收到这个消息后进行消费，但是此时发生消费失败的情况，这就产生了第3种情况。

在这种情况下，订单系统发送的消息是没有被确认的，所以只需要重新消费即可。

如果你问我如果再次消费还是失败怎么办？答案就是不断重试，当失败超过一定次数后，这个消息就会被列入死信队列，这时候就需要开发手动处理了。

接下来是第4种情况，下单失败，新增积分成功。用户都没下单就给他加积分，用户肯定爽歪歪，老板就头疼了，而作为开发，这个月绩效肯定是没了，严重的还可能被辞退，背上刑事责任。

![](https://planeswalker23.github.io/images/posts/2020060703.png)

这种情况的发生是由于用户下单与发送消息到消息队列这两个步骤没有同时失败。比如说下单需要扣除余额，假设在程序中是下单就发消息，当消息已发送但程序处理到扣除余额时发余额不足的情况，所以出现了下单失败，却新增了积分的情况。

在这样的情况下，我们就需要将整个下单流程放在同一个事务中，只有所有的下单流程都成功了之后，才能发送消息，进行用户积分的增减，这样就可以保证整个流程**要么都成功，要么都失败**。

而**事务消息**正是实现这个方案的方法之一。

## 3. 事务消息的执行流程
在`RocketMQ`官方给出的[Transaction example](https://rocketmq.apache.org/docs/transaction-example/)示例中向大家介绍了事务消息，可以将它视为一个两阶段提交的消息（半消息，`half message`），事务消息能够保证分布式系统中的最终一致性，即要么都成功，要么都失败。

事务消息的执行流程是这样的。

![事务消息的执行流程，图片来源于百度](https://planeswalker23.github.io/images/posts/2020060702.png)

1. 消息生产者发送一个半消息到`Broker`，如果半消息发送成功，`Broker`将返回一个发送成功的标识，生产者开始执行本地事务。如果半消息发送失败，整个消息也就崩于起始，发送失败。
2. 在生产者执行本地事务时或断网等特殊情况下，`Broker`未收到本地事务的最终执行状态，`Broker`会定时发起消息回查来检查本地事务的执行状态。
3. 当生产者执行本地事务结束后，生产者会发送一个二次确认的标识给`Broker`，如果是`Commit`那代表本地事务执行成功，半消息将被标记为可投递，消费者将收到该消息；如果是`Rollback`则代表本地事务执行失败，半消息将被删除，消费者也就不会收到这条消息。
4. 消费者最终将收到被标记为可投递的半消息进行消费。

## 4. 源码分析
### 4.1 发送事务消息示例
根据`RocketMQ`官方给出的[Transaction example](https://rocketmq.apache.org/docs/transaction-example/)示例代码，我们知道可以通过`TransactionMQProducer.sendMessageInTransaction`方法来发送事务消息。

```java
public class TransactionProducer {
    public static void main(String[] args) throws MQClientException, InterruptedException, UnsupportedEncodingException {
        // 1. 创建事务监听器
        TransactionListener transactionListener = new TransactionListenerImpl();
        // 2. 创建生产者
        TransactionMQProducer producer = new TransactionMQProducer("please_rename_unique_group_name");
        // 3. 创建事务状态回查异步执行线程池
        ExecutorService executorService = new ThreadPoolExecutor(2, 
                5, 
                100,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<Runnable>(2000), 
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread thread = new Thread(r);
                        thread.setName("client-transaction-msg-check-thread");
                        return thread;
                    }
                });

        producer.setExecutorService(executorService);
        producer.setTransactionListener(transactionListener);
        // 4. 启动生产者
        producer.start();
        // 5. 发送事务消息
        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 10; i++) {
            Message msg = new Message("TopicTest1234", tags[i % tags.length], "KEY" + i,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.sendMessageInTransaction(msg, null);
            System.out.printf("%s%n", sendResult);
            Thread.sleep(10);
        }
        // 6. 等待本地事务执行
        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
        }
        // 7. 关闭生产者
        producer.shutdown();
    }
}
```

1. 创建消息监听器，消息监听器是一个实现`TransactionListener`接口的类，它主要功能是执行本地事务`TransactionListener#executeLocalTransaction`和本地事务状态回查`TransactionListener#checkLocalTransaction`。
2. 创建事务消息的生产者实例，需要注意的是事务消息的生产者是一个`TransactionMQProducer`类。
3. 创建一个线程池，这个线程池主要用于异步执行事务状态的回查。
4. 启动生产者并通过`TransactionMQProducer#sendMessageInTransaction`方法发送事务消息。

### 4.2 事务监听器简析
与普通消息不一样的是，发送事务消息还需要设置事务监听器，来看代码。

```java
public class TransactionListenerImpl implements TransactionListener {
    private AtomicInteger transactionIndex = new AtomicInteger(0);

    private ConcurrentHashMap<String, Integer> localTrans = new ConcurrentHashMap<>();

    // 执行本地事务，提交本地事务状态
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        // 原子整数类自增，为每个消息设置一个整数值，用于事务回查的逻辑
        int value = transactionIndex.getAndIncrement();
        int status = value % 3;
        localTrans.put(msg.getTransactionId(), status);
        // 这里本地事务返回UNKNOW状态是为了让所有的消息都走事务回查的逻辑
        return LocalTransactionState.UNKNOW;
    }

    // 事务回查
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        // 根据TransactionId获取该消息在执行本地事务时设置的对应的整数值，返回不同的事务最终执行状态
        Integer status = localTrans.get(msg.getTransactionId());
        if (null != status) {
            switch (status) {
                case 0:
                    return LocalTransactionState.UNKNOW;
                case 1:
                    return LocalTransactionState.COMMIT_MESSAGE;
                case 2:
                    return LocalTransactionState.ROLLBACK_MESSAGE;
                default:
                    return LocalTransactionState.COMMIT_MESSAGE;
            }
        }
        return LocalTransactionState.COMMIT_MESSAGE;
    }
}
```

`executeLocalTransaction`方法是用于执行本地事务，它的返回值是一个`LocalTransactionState`枚举类，包括`COMMIT_MESSAGE, ROLLBACK_MESSAGE, UNKNOW`，分别代表提交（本地事务执行成功）、回滚（本地事务执行失败）和未知（需要消息回查验证）。

`checkLocalTransaction`方法是用于消息回查，来确定本地事务的执行状态，它的返回值同样是`LocalTransactionState`枚举类。对于本地事务状态为`UNKNOW`的消息，消息生产者会使用创建的消息回查线程池来定时调用消息回查方法来确认本地事务的最终状态。

以上就是发送事务消息的流程，接下来就是激动人心的源码分析时刻。

![](https://planeswalker23.github.io/images/posts/2020060704.png)

从前后端君认为一个框架只要能用的好就可以了，干嘛非得看源码呢，后来我才知道，看源码不光是学习前辈的代码的机会，同时也是了以后可能会遇到的“坑”做准备，毕竟是开源啊...

### 4.3 如何标记为一个事务消息
在发送事务消息的方法`TransactionMQProducer.sendMessageInTransaction`，最终执行的是`DefaultMQProducerImpl#sendMessageInTransaction`。在这个方法发送消息前，会给消息实体类`Message`设置一个属性。
```java
MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, "true");
```

这个属性标识了该消息是事务消息，同时这个属性决定了消息对消费者是不可见的。

至于为什么不可见……首先在`DefaultMQProducerImpl#sendKernelImpl`方法中会判断刚刚设置的`TRAN_MSG`标识，然后通过设置消息系统标记的方式将`sysFlag`设置为`MessageSysFlag.TRANSACTION_PREPARED_TYPE`。
```java
int sysFlag = 0;
final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
    sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
}
```

最终在发送消息的方法`SendMessageProcessor#sendMessage`中会判断是否有`MessageSysFlag.TRANSACTION_PREPARED_TYPE`的标识，如果有将调用`prepareMessage`方法来发送事务消息。
```java
String traFlag = oriProps.get(MessageConst.PROPERTY_TRANSACTION_PREPARED);
// 判断是否有事务消息的标记
if (traFlag != null && Boolean.parseBoolean(traFlag) && !(msgInner.getReconsumeTimes() > 0 && msgInner.getDelayTimeLevel() > 0)) {
    // 判断该生产者是否不支持发送事务消息，若不支持不会发送
    if (this.brokerController.getBrokerConfig().isRejectTransactionMessage()) {
        response.setCode(ResponseCode.NO_PERMISSION);
        response.setRemark("the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1()+ "] sending transaction message is forbidden");
        return response;
    }
    // 发送事务消息
    putMessageResult = this.brokerController.getTransactionalMessageService().prepareMessage(msgInner);
} // 省略其他代码...
```

我们再来跟进`prepareMessage`方法，它的整个调用链是这样的：
```java
TransactionalMessageService#prepareMessage
-> TransactionalMessageServiceImpl#prepareMessage
-> TransactionalMessageBridge#putHalfMessage

public PutMessageResult putHalfMessage(MessageExtBrokerInner messageInner) {
    return store.putMessage(parseHalfMessageInner(messageInner));
}

private MessageExtBrokerInner parseHalfMessageInner(MessageExtBrokerInner msgInner) {
    // 备份消息的原主题和队列
    MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_TOPIC, msgInner.getTopic());
    MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msgInner.getQueueId()));
    // 设置
    msgInner.setSysFlag(MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), MessageSysFlag.TRANSACTION_NOT_TYPE));
    // 重新设置Topic为事务半消息的Topic
    msgInner.setTopic(TransactionalMessageUtil.buildHalfTopic());
    // 消费队列更改为0
    msgInner.setQueueId(0);
    msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.getProperties()));
    return msgInner;
}

// TransactionalMessageUtil.buildHalfTopic()
public static String buildHalfTopic() {
    return MixAll.RMQ_SYS_TRANS_HALF_TOPIC;
}
```

在`putHalfMessage`方法中会重新包装事务消息，首先将此事务消息的主题和消费队列备份到`Massage#properties`这个`HashMap`属性中。然后将消息主题更改为`RMQ_SYS_TRANS_HALF_TOPIC`，这是专门存放事务消息的主题。等到经过消息回查确定了本地事务的状态为`COMMIT_MESSAGE`之后，就会把这个消息的主题恢复到原先的主题，从而能够让消费者消费。

## 5. 小结
本文描述了分布式事务产生的原因、`RocketMQ`执行事务消息的流程、官方给出的`Transaction`例子以及分析源码中是如何讲一个消息标记为事务消息的。

> 版权声明：本文为[Planeswalker23](https://github.com/Planeswalker23)所创，转载请带上原文链接，感谢

## 6. 参考资料
- RocketMQ 技术内幕
- [Transaction example](https://rocketmq.apache.org/docs/transaction-example/)
- [数据库事务](https://zh.wikipedia.org/wiki/数据库事务)