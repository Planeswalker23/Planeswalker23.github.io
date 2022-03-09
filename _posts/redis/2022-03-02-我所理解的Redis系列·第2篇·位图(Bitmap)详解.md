---
layout: post
title: 我所理解的Redis系列·第2篇·位图(Bitmap)详解
categories: [Reids]
keywords: Reids, 位图
---



在前几天想要写个基于位图(Bitmap)实现的签到系统时，忽然发现自己不太清楚位图的具体操作命令，之前也从来没有有过相关的实操经验，于是才有了本文。本文记录了位图的实现原理、原生命令、应用场景以及包装后的位图工具类等。

我们知道 Redis 的常用数据结构有五种基础数据结构：字符串(string)、哈希(hash)、列表(list)、集合(set)、有序集合(zset)；以及三种高阶数据结构：位图(Bitmap)、基数估算(HyperLogLog)、地理位置(GEO)。

实际上位图并不是一种单独实现的数据结构，下面就让我们一起来揭晓位图的神秘面纱吧。

![redis-2-封面](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/redis/redis-2-封面.png)



## 1. 介绍

位图我们可以把它理解成是由二进制0和1组成的一个数组，通常用来判断某个数据存不存在的。例如，下图就表示了用户在最近七天的签到记录，1表示已签到，0表示未签到。

![redis-2-1](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/redis/redis-2-1.png)

这组位图数据就表示该用户在最近七天内，只有第二天没有签到，其他几天都签到了。



## 2. 实现原理

位图实际上并不是一种单独的数据结构，它底层其实是用 **Redis 字符串**通过**面向位操作**实现的，而 Redis 字符串则是使用动态字符串(SDS)来存储的。

SDS 的底层是用一个字节数组来存储的，下面就是它的底层代码：

```c
struct sdshdr {
  // SDS 长度，即 buf 数组中已使用字节数量
 int len;
 // buf 数组中未使用的字节数量
 int free;
 // 字节数组，用于保存字符串
 char buf[];
}
```

与传统C语言 char 类型不一样的是，Redis 字符串类型是二进制安全的。在传输数据时，Redis SDS API 都会以处理二进制的方式来处理存放在 buf 数组中的数据，程序不会对数据做任何限制、过滤或其他改动，传进来的是什么内容，传出去的还是什么内容。

例如，在传统C语言中，下面这串文本将会被识别成“Redis”，而不是“Redis SDS”；而 Redis 字符串则会将其识别为“Redis SDS”。

![redis-2-2](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/redis/redis-2-2.7n4hj1s2s8s.png)

这就是 Redis 二进制安全所带来的效果。事实上，Redis 二进制安全是因为 **SDS 使用 len 属性的值而不是空字符('\0')来判断字符串是否结束**。



## 3. 位图优势

### 3.1 节省空间

位图最大的优势就是**节省空间**，我们来做一个实验：存储用户一年的签到记录，假设用户全年满签。

如果用字符串来存储，键为用户ID，值为0-1字符串，那么值的长度就是365个'1'字符，换算成字节数就是 365 字节。

而如果用位图来存储，键为用户ID，值为0-1二进制位，那么值的长度就是365个二进制位，换算成字节数就是 365 / 8 =45.625 字节。

通过对纯字符串与位图的存储对比，我们可以知道，位图在存储用户一年签到记录的场景下的使用空间是纯字符串的1/45，非常节省空间。

除此之外，再来说说位图的存储上限，我们已经知道位图底层其实就是一个 Redis 字符串，而 Redis 字符串的最大存储限制是 512 MB，这换算成字节数就是 512 * 1024 * 1024  = 536870912 字节 = 2^29 字节，换算成二进制位就是 2^29 * 8 = **2^32** 位。



### 3.2 操作效率高

位图还有一个优点是**操作效率高**，这体现在 Redis 支持直接对二进制位进行操作上。

例如在 Redis 中有个 key 叫「bitmap」，其值为「a」，现在我想把它的值改成「b」，当然我可以直接通过 `set bitmap b` 来实现，但是这是将字符串底层的 char 数组全部修改了，如果我通过位图去修改，那么只需要两条命令，修改两个二进制位即可，如下：

![redis-2-3](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/redis/redis-2-3.77yx628hsxw0.png)

我们知道字符a的二进制表达是01100001，字符b的二进制表达是01100010，而上面的两个命令：

- setbit bitmap 6 1
- setbit bitmap 7 0

这两个命令正是将 bitmap 键的第7位二进制位改成1，第8位二进制位改成0，从而实现了将整个键由字符a更新成字符b的功能。

如果说这个场景还不足以证明位图的高操作效率性的话，那么再来看一个场景：在用户签到场景下，对比字符串与位图修改第100天的签到状态操作复杂度，假设此行为是补签，当前是一年中的最后一天。

首先是字符串场景，假设用户全年仅第100天漏签了，那么这条字符串记录的值存储的就是99个1字符 + 1个0字符 + 265个1字符 ，当用户发生补签行为时，系统的操作过程如下：

- 读取用户签到记录
- 将字符数组的第100位由0改为1
- 将用户的签到记录更新位365个1字符

而如果使用位图来实现该功能，只需要一条命令：`setbit 键 99 1` 就实现了上述的过程(99是因为位图是从第0位开始计数的)。



## 4. 常用命令

### 4.1 SETBIT

该命令用于对指定 key 的某一二进制位进行赋值，并返回该位置上原来的二进制值，初始值为0。

```text
SETBIT key offset value
```

- key 表示键
- offset 表示二进制位偏移量，从0开始计数
- value 表示要赋的二进制值

```text
127.0.0.1:6379> set key a        #a字符的二进制表示为01100001
OK
127.0.0.1:6379> setbit key 6 1   #将二进制第7位改成1，即最终改为01100011，即字符c
(integer) 0
127.0.0.1:6379> get key
"c"
```



### 4.2 GETBIT

该命令用于获取指定 key 上指定位置的二进制值，若超出当前位图范围，返回0。

```text
GETBIT key offset
```

- key 表示键
- offset 表示二进制位偏移量，从0开始计数

```text
127.0.0.1:6379> get key         #c字符的二进制表示为01100011
"c"
127.0.0.1:6379> getbit key 0
(integer) 0
127.0.0.1:6379> getbit key 1
(integer) 1
127.0.0.1:6379> getbit key 2
(integer) 1
127.0.0.1:6379> getbit key 3
(integer) 0
127.0.0.1:6379> getbit key 4
(integer) 0
127.0.0.1:6379> getbit key 5
(integer) 0
127.0.0.1:6379> getbit key 6
(integer) 1
127.0.0.1:6379> getbit key 7
(integer) 1
```



### 4.3 BITCOUNT

该命令用于获取指定 key 中被设置为 1 的数量，可以通过 start 和 end 参数来指定区间，该数值是指字节数。

```text
BITCOUNT key [start end]
```

- key 表示键
- start 表示指定区间的起始字节数
- end 表示指定区间的终止字节数

```text
127.0.0.1:6379> get key               #c字符的二进制表示为01100011
"c"
127.0.0.1:6379> bitcount key
(integer) 4
127.0.0.1:6379> bitcount key 0 1
(integer) 4
127.0.0.1:6379> bitcount key 0 2
(integer) 4
127.0.0.1:6379> bitcount key 1 2
(integer) 0
127.0.0.1:6379> bitcount key 0 -1     #-1表示倒数第一个字节数
(integer) 4
```

start 和 end 值是指字节数，可以由上图中后两条命令中看出。在[0,1]区间中，有4位被设置为1，这意味着已经将 c 字符的所有为1的二进制位都算上了。



### 4.4 BITPOS

该命令用于获取指定 key 中被第一个二进制位为0或1的bit位，可以通过 start 和 end 参数来指定区间，该数值是指字节数。

```text
BITPOS key bit [start] [end]
```

- key 表示键
- bit 表示要目标二进制值是0还是1
- start 表示指定区间的起始字节数
- end 表示指定区间的终止字节数

```text
127.0.0.1:6379> get not_exist        # not_exist 是一个不存在的key
(nil)
127.0.0.1:6379> bitpos not_exist 1   # 对不存在的key查找首次出现1的位置，返回为-1
(integer) -1
127.0.0.1:6379> bitpos not_exist 0   # 对不存在的key查找首次出现0的位置，返回为0，因为二进制位默认为0
(integer) 0
127.0.0.1:6379> get zore             # zore 是一个不存在的key
(nil)
127.0.0.1:6379> setbit zore 0 1      
(integer) 0
127.0.0.1:6379> bitpos zore 0        # 任何key都会存在0，如果没有，那就会进行扩容，填充0
(integer) 1
```



## 5. 应用场景

位图适用于一些逻辑简单、重视空间占用小的特定场景，如：用户签到天数、登录次数、活跃天数各种只需要判断是与否的场景。



## 6. Bitmap工具类

下面是基于 StringRedisTemplate 实现的一个 Bitmap 工具类，包含了位图的 SETBIT, GETBIT, BITCOUNT, BITPOS 这四种基础操作，拿来即用。

```java
@Slf4j
@Component
public class BitmapUtil {

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 根据key获取值
     *
     * @param key 键
     * @return String
     */
    private String get(String key) {
        String result = redisTemplate.opsForValue().get(key);
        log.info("Bitmap GET operation successfully. Result following: key is {{}}, value is {{}}.", key, result);
        return result;
    }

    /**
     * 为key设置值
     *
     * @param key   键
     * @param value 值
     */
    public void set(String key, String value) {
        redisTemplate.opsForValue().set(key, value);
        log.info("Bitmap SET operation successfully. Result following: key is {{}}, value is {{}}.", key, value);
    }

    /**
     * 对位图中某一二进制位进行赋值
     *
     * @param key    键
     * @param offset 偏移量，二进制位
     * @param value  值
     * @return Boolean 该位置上原来的值
     */
    public Boolean setBit(String key, long offset, boolean value) {
        Boolean result = redisTemplate.opsForValue().setBit(key, offset, value);
        log.info("Bitmap SETBIT operation successfully. Result following: key is {{}}, offset is {{}}, value is {{}}. Former value is {{}}", key, offset, value, result);
        return result;
    }

    /**
     * 获取位图中某一指定偏移量的二进制值
     *
     * @param key    键
     * @param offset 偏移量，二进制位
     * @return Boolean
     */
    public Boolean getBit(String key, long offset) {
        Boolean result = redisTemplate.opsForValue().getBit(key, offset);
        log.info("Bitmap GETBIT operation successfully. Result following: key is {{}}, offset is {{}}, value is {{}}.", key, offset, result);
        return result;
    }


    /**
     * 统计位图值对应位为1的数量
     *
     * @param key 键
     * @return Long
     */
    public Long bitCountTrue(String key) {
        Long result = redisTemplate.execute((RedisCallback<Long>) con -> con.bitCount(key.getBytes()));
        log.info("Bitmap BITCOUNT operation successfully. Result following: key is {{}}, result is {{}}.", key, result);
        return result;
    }

    /**
     * 统计位图值指定范围(范围为字节范围)对应位为1的数量
     *
     * @param key   键
     * @param start 开始字节位置(包含)
     * @param end   结束字节位置(包含)
     * @return Long
     */
    public Long bitCountTrue(String key, long start, long end) {
        Long result = redisTemplate.execute((RedisCallback<Long>) con -> con.bitCount(key.getBytes(), start, end));
        log.info("Bitmap BITCOUNT operation successfully. Result following: key is {{}}, result is {{}}, startByteIndex is {}, endByteIndex is {}.", key, result, start, end);
        return result;
    }


    /**
     * 统计位图值指定二进制值的数量
     *
     * @param key   键
     * @param value 指定二进制值
     * @return Long
     */
    public Long bitPos(String key, boolean value) {
        Long result = redisTemplate.execute((RedisCallback<Long>) con -> con.bitPos(key.getBytes(), value));
        log.info("Bitmap BITPOS operation successfully. Result following: key is {{}}, value is {{}}, result is {{}}.", key, value, result);
        return result;
    }


    /**
     * 统计位图值指定范围(范围为字节范围)指定二进制值的数量
     *
     * @param key   键
     * @param value 指定二进制值
     * @param start 开始字节位置(包含)
     * @param end   结束字节位置(包含)
     * @return Long
     */
    public Long bitPos(String key, boolean value, long start, long end) {
        Long result = redisTemplate.execute((RedisCallback<Long>) con -> con.bitPos(key.getBytes(), value, Range.open(start, end)));
        log.info("Bitmap BITPOS operation successfully. Result following: key is {{}}, value is {{}}, result is {{}}, startByteIndex is {}, endByteIndex is {}.", key, value, result, start, end);
        return result;
    }

    /**
     * 根据key获取整个位图的值，并转化为8位 0-1 String
     *
     * @param key 键
     * @return String
     */
    public String getBitmapUsingString(String key) {
        String redisResult = redisTemplate.opsForValue().get(key);
        byte[] byteResult = redisResult == null ? null : redisResult.getBytes();
        String result = ByteUtil.getStringFormByteArray(byteResult);
        log.info("Bitmap GET operation successfully. Result following: key is {{}}, value is {{}}.", key, result);
        return result;
    }
}

```



## 7. 参考资料

- [Redis bitmap位图操作(图解)](http://c.biancheng.net/redis/bitmap.html)
- [Redis修行 — 位图实战](https://juejin.cn/post/6844904049909694477)
- [汉字二进制转换器](https://www.qqxiuzi.cn/bianma/erjinzhi.php)