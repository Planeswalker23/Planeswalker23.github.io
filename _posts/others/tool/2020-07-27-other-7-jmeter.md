---
layout: post
title: 我所理解的其他问题·第7篇·基于Jmeter测试Web接口性能
categories: [Others]
keywords: Jmeter
---

最近接到一个需求，产品说要对一个接口做负载均衡。当时我听到这个需求的时候，我的内心是奔溃的——这接口只有一个，怎么做负载均衡，负载均衡起码得有两个才能做啊！

最后理解了产品想要做的东西：由于线上某接口请求量过大，导致程序宕机，他想要做的是扩大这个接口的健壮性。通俗点说就是不要让程序挂掉，就可以了。

在做这个需求的时候，我首先简单对这个接口进行了简单的性能测试，在此记录。

## 性能测试工具
- Jmeter
- JProfile

## 测试指标介绍
- 响应时间(RT, Response Time)：客户端从发起请求到接收最后一个字节数据为止所消耗的时间。
- 每秒查询数(QPS, Queries Per Second)：服务器在一秒内处理的请求次数。
- 吞吐量(throughput)：单位时间内系统处理用户的请求数
- 其他...

## Jmeter配置
对于此次接口性能测试，我在`Jmeter`中添加的所有元件及处理器如下

![所有元件及处理器](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642851371243-6291bf5a-90fc-42e4-885f-d272fac7af19.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_13%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

- 线程组：发起请求的线程数，循环次数等参数
- HTTP请求：请求的协议、服务器、端口、请求地址、请求参数等
- 结果树：记录了请求的响应信息
- 响应时间图：以图表的形式反应请求的响应时间
- 汇总报告：自动统计本次测试数据，包括样本数、平均响应时间、最大最小响应时间、异常率、吞吐量、数据包接收等
- 生成概要结果：主要用于统计请求总时间，其他指标与汇总报告有重叠，不再赘述

## 单线程请求的性能测试
首先测试的是接口的平均响应时间，我在线程组中配置了一个线程，循环100次。`ramp-up period`参数的作用是在此时间内建立全部的线程，这里不需要改动。

![线程组配置](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642851377123-185049ef-5a68-4edb-9ce9-0651d74b651d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_40%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

关于响应时间图的配置，首先需要设置一个输出文件路径，然后配置合适的时间间隔（不同的时间间隔显示的图表会不一致），我这里配置的是10毫秒。

![响应时间图配置](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642851382609-d68c5b00-d9cb-4317-a974-5a7de4e30a57.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_40%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

结果树在响应时间的测试中并不重要，汇总报告和概要结果不需要配置。

在完成了上述的配置之后，点击启动按钮，在等待一段时间后，整个测试流程结束，首先打开响应时间图中的**显示图表**。

![响应时间图](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642851387789-078b8a3f-abe9-4e67-a955-0adc62a48cd2.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_33%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

从上面的图形中可以粗略看到，每次请求的平均响应时间大概在40ms左右。

同时，可以在**汇总报告**界面看到本次测试的数据结果。

![汇总报告](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642851393581-1f965a61-e77c-4014-8794-f2530a72e25d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_41%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

从表格中可以看到，总共的样本数是100，平均响应时间为41ms，最小响应时间为28ms，最大响应时间为135毫秒，请求出错的概率是0.00%，吞吐量是每秒23.7次，数据包的统计值暂且不论。

由于本次测试是由一个线程顺序执行完成的，我么可以得出一个结论：在同一时间内只有一个用户对该接口发起请求时，该接口的平均响应时间为41ms。

以上对于单个用户请求的性能测试只是相当于一个试用`Jmeter`的`demo`工程，因为在正常情况下，线上系统的用户不可能只有一个，所以以上得出的结论不能代表这个接口的性能，我们需要找到一个合适的最佳并发数，然后进行并发测试。

## 并发请求的性能测试
将线程组中循环次数的配置项改为10次（为了缩短测试时间），然后通过改变线程数进行多次测试，得到的测试结果如下表。

|线程数|样本数|请求总时间(ms)|平均响应时间RT(ms)|吞吐量ThoughPut(个/s)|QPS(个/s)|
| ----- | -----  | -----  | ----- | ----- | ----- |
|1|10|1|60|16.6|10|
|10|100|2|145|64.5|50|
|20|200|3|270|67.7|66.7|
|30|300|4|349|79.4|75|
|40|400|6|511|68.7|66.7|
|50|500|8|735|64.4|62.5|
|60|600|11|970|57.6|54.5|

> QPS = 样本数/请求总时间，此接口为查询接口，故QPS与TPS可视为相同。

可以看到，当线程数为60个线程时，吞吐量和QPS都开始小于线程数，说明此时已经是一个瓶颈了，如果再往上加并发数，性能会越来越低，所以本接口的**最佳并发数**是50左右。

同时，观察到吞吐量呈一个先上升后下降的趋势，最高点即为**最大吞吐量**是79.4/s。

最后来测试**最大并发数**，继续增加线程数，当程序开始抛出`Error`异常，同时**汇总报告**中错误率不再为0%时，就达到了最大并发数，通过漫长的伪二分测试，我得到了本程序的最大并发数为160。

## 小结
基于 Jmeter 的性能测试到这里就结束了，主要涉及的指标有平均响应时间(RT)、吞吐量(Thoughput)、每秒查询数(QPS)、最佳并发数、最大吞吐量、最大并发数。

回到最初的需求——排查宕机原因——其实在日志里面说的很清楚了，是由于堆内存溢出。

而在上述的压测中，本程序的最大并发数只有160的原因是它的堆大小初始值只有256MB，使用`-Xms512m`启动附加命令将初始堆大小扩大为512MB，最大并发数也随之扩大，在相同的情况下就不会再出现堆内存溢出的情况了，后续就算并发数再扩大一倍也不会再出现宕机的情况（除非服务消费端循环调个几千次，并发数又增加好几倍...）。

## 再小结一次
在写完这篇性能测试小记之后，又有些新的内容做下分享：

首先是在测试之前，需要知道线上JVM、服务器的各种配置，本地程序的环境尽可能地模拟线上环境。

第二是并发数，最好是线上环境有PV的统计，这样就可以知道在宕机时系统的访问量，在`Jmeter`的线程数的配置时直接填访问量就可以，这样做的目标最终其实也是模拟线上的真实环境。

第三是考虑从代码上进行优化，比如说配置文件中的配置项可以解析为配置类在程序启动时就加载进来，而不是每次使用到的时候就去解析一次配置文件，这样做毫无意义（我也不知道项目里面为什么这么写，过几天把它优化掉）。

第四，也是比较重要的一点，进行性能测试的时候一定不要连生产环境的数据库（一般来说正常公司都会做生产环境与开发环境的隔离的）或者第三方接口啥的，否则把生产环境搞炸就完了。

## 参考
- [接口性能测试方案](https://www.cnblogs.com/abcd19880817/p/7206536.html)

> 版权声明：本文为[Planeswalker23](https://github.com/Planeswalker23)所创，转载请带上原文链接，感谢。