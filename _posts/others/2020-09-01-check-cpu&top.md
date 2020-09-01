---
layout: post
title: 记一次线上CPU警报排查过程&top命令
categories: [Others]
description: 记一次线上CPU警报排查过程&top命令
keywords: cpu, top
---

上周某天晚上上线，完事后当所有人准备高高兴兴下班的时候，群里报警机器人突然不合时宜地冒泡了：服务器CPU资源突然升高了，照理说这个点应该不会有很多用户使用我们系统，估计是某个定时任务或者开放API导致的。

![CPU警报](https://planeswalker23.github.io/images/posts/2020090101.png)

没办法，又得打开已经关闭了的显示器，开始排查到底是哪里的问题。

![封面](https://planeswalker23.github.io/images/posts/2020090100.png)

## 1. 排查过程记录
### 1.1 top

第一步是用`top`命令查看CPU资源分配情况及进程号。

![进程占用CPU情况](https://planeswalker23.github.io/images/posts/2020090101.png)

从图中可以看到，`PID=31439`的进程的`%CPU`值为100%，说明这条进程占用CPU的时间已经到达100%，线上的警报就是由这个进程发出的。

如果不确定，可以用`jps -l`命令查看线上服务器正在运行的`Java`项目，对比进程id来确定警报来源。

### 1.2 top -Hp 进程PID
第二步是用`top -Hp 进程PID`命令查看该进程下各线程CPU资源的使用情况。

![线程占用CPU情况](https://planeswalker23.github.io/images/posts/2020090102.png)

这条命令是为了确定是哪个线程执行的任务导致CPU警报的，从图上可以看到`线程PID=31440`，它占用了99%的CPU。

### 1.3 printf "0x%x\n" 线程PID
第三步是用`printf "0x%x\n" 线程PID`将线程PID转化为十六进制。

![线程PID十六进制](https://planeswalker23.github.io/images/posts/2020090103.png)

这一步也可以用其他方式代替，比如“在线十进制转十六进制”网站，获取线程PID的十六进制是因为在线程堆栈文件中，线程号就是用十六进制来表示的。

### 1.4 jstack 进程PID | vim +/十六进制线程PID -
第四步是用`jstack 进程PID | vim +/十六进制线程PID -`命令输出进程堆栈信息，同时通过十六进制线程号定位到指定线程号堆栈信息。

所以稳妥的方法就是把进程的堆栈信息都输出到一个文件中，然后根据十六进制的线程号定位到线程堆栈信息。

![线程堆栈文件](https://planeswalker23.github.io/images/posts/2020090104.png)

从上图可以看到，这个线程执行的代码是在`Demo.java:3`。

最后就是打开工程，查看这个类的源码，看看为什么在这里会消耗大量CPU。

> `Demo.java`文件是本文的实例代码，它是一个美丽的死循环。

也可以分两步完成以上操作：
- `jstack 进程PID > demo.txt`导出进程堆栈信息，导出到`demo.txt`文件
- 打开堆栈文件搜索十六进制线程ID号

## 2. top 执行结果
> 操作系统版本为：`CentOS Linux release 7.3.1611 (Core)`，可通过`cat /etc/redhat-release`命令查看。

然后这里记录一下 top 命令的各个参数的意义，平常只是用来看cpu和内存，其他指标还真啥都不知道，以后“查字典”也方便。

第一行：概况，与`uptime`命令的执行结果相同。
- HH:mm:ss：系统当前时间
- up x days,  HH:mm：从本次开机到当前时间的系统运行时间（单位：分钟）
- x user：当前登录到本机的用户数
- load average: x.xx, x.xx, x.xx：系统在1分钟、5分钟、15分钟内的系统平均负载值
    - 关于负载均衡值和算法的介绍可以看[这篇文章](http://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html)
    - 当然我也没看完上面那篇文章（全英文有点看不过来），但是文章中有一段解释对了解系统平均负载值是有帮助的：
        - 如果平均值为0.0，那么系统是闲置的
        - 如果1分钟平均值大于5分钟或15分钟的负载值，那么负载在增加
        - 如果1分钟平均值低于5分钟或15分钟的负载值，那么负载在减少
        - 如果这三个值高于CPU数，那么你可能有性能问题（视情况而定）

第二行：进程信息
- x total：所有进程数
- x running：正在运行的进程数
- x sleeping：休眠进程数
- x zombie：僵尸进程数
    - 僵尸进程是当子进程比父进程先结束，而父进程又没有回收子进程，释放子进程占用的资源，此时子进程将成为一个僵尸进程。

第三行：CPU状态
- x.x us：用户空间（user space）消耗CPU时间百分比
- x.x sy：内核空间（kernel space，system）消耗CPU时间百分比
- x.x ni：调整过用户态优先级的进程（niced user processes）消耗CPU时间百分比
- x.x wa：空闲（wait space）CPU时间百分比
- x.x hi：处理硬中断（hardware interrupt）的CPU时间百分比
- x.x si：处理软中断（software interrupt）的CPU时间百分比
- x.x st：等待CPU资源（steal time）的时间百分比

第四行：内存状态
- xxx total：内存总量
- xxx free：空闲内存量
- xxx used：已使用内存量
- xxx buff/cache：缓存和page cache占用的内存量

第五行：交换区状态
- xxx total：内存总量
- xxx free：空闲内存量
- xxx used：已使用内存量
- xxx avail Mem：可用于进程下一次分配的物理内存数量

第七行至末尾：进程详细信息
- PID：进程PID
- USER：进程所有者的用户名
- PR：进程调度优先级
- NI：从用户空间视角的进程优先级（niced值），值越低优先级越高
- VIRT：进程使用的虚拟内存（kb），VIRT=SWAP+RES
- RES：进程使用的（未被换出的）物理内存（kb），RES=CODE+DATA
- SHR：共享内存大小（kb）
- S：进程状态
    - D：不可中断的睡眠状态
    - R：运行
    - S：睡眠
    - T：跟踪/停止
    - Z：僵尸进程
- %CPU：进程占用CPU时间比例
- %MEM：进程占用的物理内存比例
- TIME+： 进程占用的CPU时间总量（ms）
- COMMAND：运行进程使用的命令

## 3. 参考资料
- [Linux top命令小结](https://www.jianshu.com/p/a6e96c102881)
- [Linux top命令详解](https://www.cnblogs.com/niuben/p/12017242.html)
- [僵尸进程](https://baike.baidu.com/item/%E5%83%B5%E5%B0%B8%E8%BF%9B%E7%A8%8B)
- [Linux “top” command: What are us, sy, ni, id, wa, hi, si and st (for CPU usage)?](https://unix.stackexchange.com/questions/18918/linux-top-command-what-are-us-sy-ni-id-wa-hi-si-and-st-for-cpu-usage)
- [Linux内存管理](https://segmentfault.com/a/1190000008125006)

> 版权声明：本文为[Planeswalker23](https://github.com/Planeswalker23)所创，转载请带上原文链接，感谢。