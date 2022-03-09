---
layout: post
title: 我所理解的Redis系列·第b篇·Redis位图如何以二进制字符串的形式展示
categories: [Reids]
keywords: Reids, 位图
---



上一篇 [我所理解的Redis系列·第2篇·位图(Bitmap)详解 ](https://www.yuque.com/planeswalker/bankend/lamqao)中介绍了 Redis 位图的基本命令以及基于 Spring 的 RedisTemplate 实现的 Bitmap 工具类。但是在基于 Redis 位图设计通用的签到系统过程中，还发现了另一个问题：怎么才能让前端展示一周/一个月中用户的签到记录？

![redis-b-封面](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/redis/redis-b-封面.png)



## 1. 背景

用户签到记录的页面，一般会以当月月历的形式来呈现（如下图就是支付宝会员的签到页面）。前端会根据后端的响应数据来进行实际签到行为的渲染。

<img src="https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/redis/redis-b-1.jpg" alt="redis-b-1" style="zoom:50%;" />

一般来说，后端的响应数据有两种实现方案：

1. 返回用户当月已签到的具体日期，如上图的返回结果就是仅返回前六天的日期
2. 返回用户当月的完整签到记录，如「3.1已签到、3.2未签到」这样固定形式的 json 格式数据结构

总之需要知道用户哪天签到了，哪天没签到。

那么这里问题就来了，存储用户签到记录的数据结构是 Redis 位图，但实际上位图的底层数据结构还是 Redis 字符串，也就是说，后端程序拿到 Redis 中的数据是字符类型的。

例如用户只有第2、3、8天签到了，通过位图将用户前七天签到记录存在 Redis 中，而去获取该 key 时，得到的却是「a」。

```text
127.0.0.1:6379> setbit user:sign:1 0 0
(integer) 0
127.0.0.1:6379> setbit user:sign:1 1 1
(integer) 1
127.0.0.1:6379> setbit user:sign:1 2 1
(integer) 1
127.0.0.1:6379> setbit user:sign:1 3 0
(integer) 0
127.0.0.1:6379> setbit user:sign:1 4 0
(integer) 0
127.0.0.1:6379> setbit user:sign:1 5 0
(integer) 0
127.0.0.1:6379> setbit user:sign:1 6 0
(integer) 0
127.0.0.1:6379> setbit user:sign:1 7 1
(integer) 1
127.0.0.1:6379> get user:sign:1
"a"
```

要想前端能够成功解析用户的签到记录，我们返回的响应数据就应该是类似「01100001」这种有规律的字符，而不是只有一个「a」。



## 2. 解决思路

为了让前端能够成功渲染用户签到记录月历，我想到了两个方案来进行数据转化。



### 2.1 GETBIT

第一种方案就很粗暴，直接通过 GETBIT 来获取当月每天的签到状态，最后拼成一个完整的字符串进行响应。

```java
public String getBitString(Long days, String key) {
  // 拼接返回结果
  StringBuilder result = new StringBuilder();
  for (long offset = 0; offset < days; offset++) {
    // 获取指定偏移位上的二进制值
    Boolean booleanResult = redisTemplate.opsForValue().getBit(key, offset);
    // 进行字符串0-1赋值
    result.append(Boolean.TRUE.equals(booleanResult) ? "1" : "0");
  }
  return result.toString();
}
```

当然这种方法不太可取，因为这里要进行30次的与 Redis 的交互操作。一般 Redis 一次简单的取值耗时在几毫秒左右，假设是1毫秒吧，30次也得是30毫秒，或许这也不是非常慢，但是这30次交互是会占用 Redis CPU 的，如果数据多了，请求也多了，就可能会造成严重的后果。

所以这种方式不可取。



### 2.2 字符转为二进制

第二种方案只需要一次 Redis 交互就能完整取出当月用户的所有签到记录，但是在程序中会多一步字符转化为二进制的处理，当然，在逻辑上稍微复杂一点。

```java
public String getBitString2(String key) {
  // 获取 Redis 中对应 KEY 的值
  String redisResult = redisTemplate.opsForValue().get(key);
  // 将字符串转化为字节数组
  byte[] byteResult = redisResult == null ? new byte[0] : redisResult.getBytes();
  StringBuilder result = new StringBuilder();
  for (int i = 0; i < byteResult.length; i++) {
    byte sourceByte = byteResult[i];
    // 将字节数组中每一位转化为二进制
    char[] charArray = new char[8];
    for (int j = 7; j >= 0; j--) {
      // 判定byte的最后一位是否为1，若为1，则是true；否则是false
      charArray[j] = (sourceByte & 1) == 1 ? '1' : '0';
      // byte右移一位
      sourceByte = (byte) (sourceByte >> 1);
    }
    // 将转化结果二进制字符数组拼接到返回结果中
    result.append(charArray);
  }
  return result.toString();
}
```

将字符转化为二进制值，再转化为字符串的逻辑如下：

1. 首先将字符串转化为字节数组，转化时需要做NULL判断，防止空指针
2. 然后对字节数组的每一位进行转化二进制字符数组的操作
   1. 声明一个长度为8的for循环（因为一个字节等于8个二进制位）
   2. 每次循环通过 & 运算符判断 byte 的最后一位是否为1
   3. 判断完成后字节右移一位进行下一次循环
   4. 8次循环结束后将得到一个8位长度的字符数组，每一位的数据要么是'0'，要么是'1'
3. 将得到的8位字符数组拼接到最终返回结果中



以上两种方案的效率我也通过实际运行进行了对比，结果如下：

```java
start1: 1646570941960
  end1: 1646570942024 // 方案1:64毫秒
start2: 1646570942024
  end2: 1646570942029 // 方案2:5毫秒
```

结果也正如开始时说的那样，方案1相对而言更耗时。



## 3. 解决方案

在解决了如何将位图中的字符转化为0-1形式的二进制字符的核心问题后，本文开篇提出的问题也就迎刃而解。

假设 Redis 位图的 key 是某ID为1的用户2022年3月份的签到记录，key 为「user:1:sign:2022:3」。

前端想要获取该用户当月的签到记录，后端只需要先从 redis 中直接获取此键的值，然后使用上文中方案2的代码进行解析，将内容转化为通过0和1表示未签到和已签到的字符数组，最后返回给前端即可。

如果用户只有第2、3、8天签到了，那么前端收到的响应数据应该就是「01100001」，随后前端在渲染月历时就可以根据该字符串进行每个字节的判断。

> 我不是前端，如果 JS 中字符串不能转化为字节数组的话，那么后端可以使用字符数组或者其他前端可接受的形式返回，以供遍历，当然这不是重点，在此就不多做讨论了。

本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。