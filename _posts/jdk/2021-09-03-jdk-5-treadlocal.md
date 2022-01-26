---
layout: post
title: 我所理解的JDK系列·第5篇·ThreadLocal原理知多少
categories: [JDK]
description: 最早听说 ThreadLocal 是18年还在实习的时候，那时候有一个要用到线程池的任务，有人说并发的问题也可以通过 ThreadLocal 来解决。但当时没有用到这玩意，只留下了个“可以用它来解决并发问题”的模糊印象。
keywords: Java, JDK, ThreadLocal
---


最早听说 ThreadLocal 是18年还在实习的时候，那时候有一个要用到线程池的任务，有人说并发的问题也可以通过 ThreadLocal 来解决。但当时没有用到这玩意，只留下了个“可以用它来解决并发问题”的模糊印象。



![cover](https://cdn.nlark.com/yuque/0/2021/png/2331602/1630608650562-c325589d-52d2-443a-be5e-bf22d8f5301a.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_26%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



## 1. 开篇词

今天要总结的是 ThreadLocal，主要的议题有：
1. ThreadLocal 介绍
2. ThreadLocal 实现原理，包括底层数据结构、散列方式、哈希冲突、扩容机制等
3. ThreadLocal 内存泄漏分析
4. ThreadLocal 应用场景及示例

最早听说 ThreadLocal 是18年还在实习的时候，那时候有一个要用到线程池的任务，有人说并发的问题也可以通过 ThreadLocal 来解决。但当时没有用到这玩意，只留下了个“可以用它来解决并发问题”的模糊印象。

直到现在，我也会在项目中用到 ThreadLocal 了，但如果要详细的解释它的实现原理，我感觉好像还是有些模糊，所以就趁着有空看了看 ThreadLocal 的源码，把我对它的理解以及所有想象分享给大家。



## 2. ThreadLocal 介绍

正如 JDK 注释中所说的那样：”ThreadLocal 类提供线程局部变量，它通常是私有类中希望将状态与线程关联的静态字段。”

简而言之，就是 ThreadLocal 提供了**线程间数据隔离**的功能，从它的命名上也能知道这是属于一个线程的本地变量。也就是说，每个线程都会在 ThreadLocal 中保存一份该线程独有的数据，所以它是线程安全的。

熟悉 Spring 的同学可能知道 Bean 的作用域（Scope），而 ThreadLocal 的作用域就是线程。

我们可以通过一个简单示例来展示一下 ThreadLocal 的特性：

```java
public static void main(String[] args) {
  ThreadLocal<String> threadLocal = new ThreadLocal<>();
  // 创建一个有2个核心线程数的线程池
  ExecutorService threadPool = new ThreadPoolExecutor(2, 2, 1, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(10));
  // 线程池提交一个任务，将任务序号及执行该任务的子线程的线程名放到 ThreadLocal 中
  threadPool.execute(() -> threadLocal.set("任务1: " + Thread.currentThread().getName()));
  threadPool.execute(() -> threadLocal.set("任务2: " + Thread.currentThread().getName()));
  threadPool.execute(() -> threadLocal.set("任务3: " + Thread.currentThread().getName()));
  // 输出 ThreadLocal 中的内容
  for (int i = 0; i < 10; i++) {
    threadPool.execute(() -> System.out.println("ThreadLocal value of " + Thread.currentThread().getName() + " = " + threadLocal.get()));
  }
  // 线程池记得关闭
  threadPool.shutdown();
}
```

上面代码首先创建了一个有2个核心线程数的普通线程池，随后提交一个任务，将任务序号及执行该任务的子线程的线程名放到 ThreadLocal 中，最后在一个 for 循环中输出线程池中各个线程存储在 ThreadLocal 中的值。这个程序的输出结果是：

```java
ThreadLocal value of pool-1-thread-1 = 任务3: pool-1-thread-1
ThreadLocal value of pool-1-thread-2 = 任务2: pool-1-thread-2
ThreadLocal value of pool-1-thread-1 = 任务3: pool-1-thread-1
ThreadLocal value of pool-1-thread-2 = 任务2: pool-1-thread-2
ThreadLocal value of pool-1-thread-1 = 任务3: pool-1-thread-1
ThreadLocal value of pool-1-thread-2 = 任务2: pool-1-thread-2
ThreadLocal value of pool-1-thread-1 = 任务3: pool-1-thread-1
ThreadLocal value of pool-1-thread-2 = 任务2: pool-1-thread-2
ThreadLocal value of pool-1-thread-1 = 任务3: pool-1-thread-1
ThreadLocal value of pool-1-thread-2 = 任务2: pool-1-thread-2
```

由此可见，线程池中执行提交的任务的是名为 `pool-1-thread-1` 的线程，随后多次输出线程池核心线程在 ThreadLocal 变量中存储的的内容也表明：每个线程在 ThreadLocal 中存储的内容是当前线程独有的，在多线程环境下，能够有效防止自己的变量被其他线程修改（存储的内容是同一个引用类型对象的情况除外）。



## 2. ThreadLocal 实现原理

在 JDK1.8 版本中 ThreadLocal 类的源码总共723行，去掉注释大概有350行，应该算是 JDK 核心类库中代码量比较少的一个类了，相对来说它的源码还是挺容易理解的。

下面，就从 ThreadLocal 的数据结构开始聊聊它的实现原理吧。



### 2.1 底层数据结构

首先开章名义：**ThreadLocal 底层是通过 ThreadLocalMap 这个静态内部类来存储数据的，ThreadLocalMap 就是一个键值对的 Map，它的底层是 Entry 对象数组，Entry 对象中存放的键是 ThreadLocal 对象，值是 Object 类型的具体存储内容**。

除此之外，ThreadLocalMap 也是 Thread 类一个属性。

![ThreadLocal 底层数据结构](https://cdn.nlark.com/yuque/0/2020/png/2331602/1606744207960-d3b0ab14-cdc8-42ff-9eec-0cbcaf59060d.png#align=left&display=inline&height=327&margin=%5Bobject%20Object%5D&originHeight=327&originWidth=561&size=0&status=done&style=none&width=561)

如何证明上面给出的 ThreadLocal 类底层数据结构的正确性？我们可以从 `ThreadLocal#get()` 方法开始追踪代码，看看线程局部变量到底是从哪里被取出来的。

```java
public T get() {
  // 获取当前线程
  Thread t = Thread.currentThread();
  // 获取 Thread 类中 ThreadLocal.ThreadLocalMap 类型的 threadLocals 变量
  ThreadLocalMap map = getMap(t);
  // 若 threadLocals 变量不为空，根据 ThreadLocal 对象来获取 key 对应的 value
  if (map != null) {
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  // 若 threadLocals 变量是 NULL，初始化一个新的 ThreadLocalMap 对象
  return setInitialValue();
}

// ThreadLocal#setInitialValue
// 初始化一个新的 ThreadLocalMap 对象
private T setInitialValue() {
  // 初始化一个 NULL 值
  T value = initialValue();
  // 获取当前线程
  Thread t = Thread.currentThread();
  // 获取 Thread 类中 ThreadLocal.ThreadLocalMap 类型的 threadLocals 变量
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
  return value;
}

// ThreadLocalMap#createMap
void createMap(Thread t, T firstValue) {
  t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

通过 `ThreadLocal#get()` 方法可以很清晰的看到，我们根据 ThreadLocal 对象从 ThreadLocal 中读取数据时，首先会获取当前线程对象，然后得到当前线程对象中 ThreadLocal.ThreadLocalMap 类型的 threadLocals 属性，如果 threadLocals 属性不为空，会根据 ThreadLocal 对象作为 key 来获取 key 对应的 value；如果 threadLocals 变量是 NULL，就初始化一个新的 ThreadLocalMap 对象。

再看 ThreadLocalMap 的构造方法，也就是 Thread 类中 `ThreadLocal.ThreadLocalMap` 类型的 threadLocals 属性不为空时的执行逻辑。

```java
// ThreadLocalMap 构造方法
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
  table = new Entry[INITIAL_CAPACITY];
  int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
  table[i] = new Entry(firstKey, firstValue);
  size = 1;
  setThreshold(INITIAL_CAPACITY);
}
```

这个构造方法其实是将 ThreadLocal 对象作为 key，存储的具体内容 Object 对象作为 value，包装成一个 Entry 对象，放到 ThreadLocalMap 类中类型为 Entry 数组的 table 属性中，这样就完成了线程局部变量的存储。

所以说，**ThreadLocal 中的数据最终是存放在 ThreadLocalMap 这个类中的**。



### 2.2 散列方式

在 `ThreadLocalMap#set(ThreadLocal<?> key, Object value)` 方法中有这么一行代码：

```java
// 获取当前 ThreadLocal 对象的散列值
int i = key.threadLocalHashCode & (len-1);
```

这行代码得到的值其实是一个 ThreadLocal 对象的散列值，这就是 ThreadLocal 的散列方式，我们称之为**斐波那契散列**。

```javascript
// ThreadLocal#threadLocalHashCode
private final int threadLocalHashCode = nextHashCode();

// ThreadLocal#nextHashCode
private static int nextHashCode() {
	  return nextHashCode.getAndAdd(HASH_INCREMENT);
}

// ThreadLocal#nextHashCode
private static AtomicInteger nextHashCode = new AtomicInteger();

// AtomicInteger#getAndAdd
public final int getAndAdd(int delta) {
  	return unsafe.getAndAddInt(this, valueOffset, delta);
}

// 魔数 ThreadLocal#HASH_INCREMENT
private static final int HASH_INCREMENT = 0x61c88647;
```

`key.threadLocalHashCode` 所涉及的函数及属性如上所示，每一个 ThreadLocal 的 threadLocalHashCode 属性都是基于魔数 `0x61c88647` 来生成的。

这里就不讨论选择这个魔数的原因了（其实是我看不太懂），总之大量的实践证明：**使用`0x61c88647` 作为魔数生成的 threadLocalHashCode 再与2的幂取余，得到的结果分布很均匀**。

>  对 A 进行2的幂取余操作 `A % 2^N` 可以通过 `A & (2^n-1)` 来代替，两者计算结果相同，且位运算的效率比取模效率高很多，这个在 [HashMap八股文知多少？](https://www.yuque.com/planeswalker/bankend/hashmap) 一文中也做过相应讨论。



### 2.3 如何解决哈希冲突

我们已经知道 ThreadLocalMap 类的底层数据结构是一个 Entry 类型的数组，但与 HashMap 中的 Node 类数组+链表形式不同的是，Entry 类没有 next 属性来构成链表，所以它是一个单纯的数组。

就算上面所说的**斐波那契散列法**真的能够充分散列，但难免还是可能会发生哈希碰撞，那么问题来了，Entry 数组是如何解决哈希冲突的？

这就需要拿出 `ThreadLocal#set(T value)` 方法了，而具体处理哈希冲突的逻辑是在 `ThreadLocalMap#set(ThreadLocal<?> key, Object value)` 方法中的：

```java
public void set(T value) {
  // 获取当前线程
  Thread t = Thread.currentThread();
  // 获取 Thread 类中 ThreadLocal.ThreadLocalMap 类型的 threadLocals 变量
  ThreadLocalMap map = getMap(t);
  // 若 threadLocals 变量不为空，进行赋值；否则新建一个 ThreadLocalMap 对象来存储
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
}

// ThreadLocalMap#set
private void set(ThreadLocal<?> key, Object value) {
  // 获取 ThreadLocalMap 的 Entry 数组对象
  Entry[] tab = table;
  int len = tab.length;
  // 基于斐波那契散列法获取当前 ThreadLocal 对象的散列值
  int i = key.threadLocalHashCode & (len-1);
  // 解决哈希冲突，线性探测法
  for (Entry e = tab[i];
       e != null;
       e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();
		// 代码（1）
    if (k == key) {
      e.value = value;
      return;
    }
		// 代码（2）
    if (k == null) {
      replaceStaleEntry(key, value, i);
      return;
    }
  }
  // 代码（3）将 key-value 包装成 Entry 对象放在数组退出循环时的位置中
  tab[i] = new Entry(key, value);
  int sz = ++size;
  if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
}

// ThreadLocalMap#nextIndex
// Entry 数组的下一个索引，若超过数组大小则从0开始，相当于环形数组
private static int nextIndex(int i, int len) {
  return ((i + 1 < len) ? i + 1 : 0);
}
```

具体分析处理哈希冲突的 `ThreadLocalMap#set(ThreadLocal<?> key, Object value)` 方法，可以看到，在拿到 ThreadLocal 对象的散列值之后进入了一个 for 循环，循环的条件也很清楚：从 Entry 数组的 ThreadLocal 对象散列值处开始，每次向后挪一位，如果超过数组大小则从0开始继续遍历，直到 Entry 对象为 NULL 为止。

在循环过程中：

- 如代码（1），如果当前 ThreadLocal 对象正好等于 Entry 对象中的 key 属性，直接**更新 ThreadLocal 中 value 的值**；

- 如代码（2），如果当前 ThreadLocal 对象不等于 Entry 对象中的 key 属性，并且 Entry 对象的 key 是空的，这里进行的逻辑其实是**设置键值对，同时清理无效的 Entry**（基于**探测式清理**在一定程序上防止内存泄漏，下文会有详细介绍）；

- 如代码（3），如果在遍历中没有发现当前 TheadLocal 对象的散列值，也没有发现 Entry 对象的 key 为空的情况，而是满足了退出循环的条件（即找到了能够存放此键值对的位置），那么就会**创建一个新的 Entry 对象进行存储**，然后做一次**启发式清理**，在 while 循环中将本次索引下标位置开始不断的右移清理无效的元素将 Entry 数组中 key 为空，value 不为空的对象的 value 值全部释放；

至此，我们分析完了在向 ThreadLocal 中存储数据时，拿到 ThreadLocal 对象散列值之后的逻辑，回到本小节的主题—— ThreadLocal 是如何解决哈希冲突的？

由上面的代码可以知道，在基于斐波那契散列法获取当前 ThreadLocal 对象的散列值之后进入了一个循环，在循环中是处理具体处理哈希冲突的方法：

- 如果散列值已存在且 key 为同一个对象，直接更新 value
- 如果散列值已存在但 key 不是同一个对象，尝试在下一个空的位置进行存储

所以，来总结一下 ThreadLocal 处理哈希冲突的方式就是：**如果在 set 时遇到哈希冲突，ThreadLocal 会通过线性探测法尝试在数组下一个索引位置进行存储，同时在基于线性探测法解决哈希冲突时也会基于探测式清理释放 key 为 NULL，value 不为 NULL 的无效 Entry 对象的 value 属性来防止内存泄漏 **。



### 2.4 扩容机制

再来谈谈 ThreadLocal 的扩容机制，首先来介绍 ThreadLocal 的初始容量。在上文中有提到过 ThreadLocalMap 的构造方法，如下所示：

```java
// ThreadLocalMap 构造方法
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
  // 初始化 Entry 数组
  table = new Entry[INITIAL_CAPACITY];
  int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
  table[i] = new Entry(firstKey, firstValue);
  size = 1;
  // 设置扩容条件
  setThreshold(INITIAL_CAPACITY);
}
```

- **ThreadLocalMap 的初始容量是 16**

```java
// 初始化容量
private static final int INITIAL_CAPACITY = 16;
```

下面聊一下 **ThreadLocalMap 的扩容机制**，它在扩容前有两个判断的步骤，都满足后才会进行最终扩容：

- `ThreadLocalMap#set(ThreadLocal<?> key, Object value)` 方法中会触发**启发式清理**，在启发式清理无效 Entry 对象后，如果数组长度大于等于数组定义长度的 2/3，则首先进行 rehash；

```javascript
// 启发式清理后，若容量大于等于阈值，触发 rehash
if (!cleanSomeSlots(i, sz) && sz >= threshold)
  rehash();

// rehash 条件 sz >= threshold
private void setThreshold(int len) {
  threshold = len * 2 / 3;
}
```

- rehash 会触发**全量清理**，清理完成之后如果数组长度大于等于数组定义长度的 1/2，则进行扩容；

```java
// 扩容条件
private void rehash() {
  // 全量清理
  expungeStaleEntries();

  // Use lower threshold for doubling to avoid hysteresis
  // threshold = len*2/3
  // threshold - threshold*1/4 = len*2/3 - len*2/3*1/4 = 1/2
  if (size >= threshold - threshold / 4)
    resize();
}
```

- 进行扩容时，Entry 数组为**扩容为原来的2倍**，重新计算 key 的散列值，如果再次遇到 key 为 NULL 的情况，会将其 value 也置为 NULL，帮助虚拟机进行GC。

```java
// 具体的扩容函数
private void resize() {
  Entry[] oldTab = table;
  int oldLen = oldTab.length;
  // 2倍扩容
  int newLen = oldLen * 2;
  Entry[] newTab = new Entry[newLen];
  int count = 0;

  for (int j = 0; j < oldLen; ++j) {
    Entry e = oldTab[j];
    if (e != null) {
      ThreadLocal<?> k = e.get();
      if (k == null) {
        e.value = null; // Help the GC
      } else {
        int h = k.threadLocalHashCode & (newLen - 1);
        while (newTab[h] != null)
          h = nextIndex(h, newLen);
        newTab[h] = e;
        count++;
      }
    }
  }

  setThreshold(newLen);
  size = count;
  table = newTab;
}
```



### 2.5 父子线程间局部变量如何传递

我们已经知道 ThreadLocal 中存储的是线程的局部变量，那如果现在有个需求，想要实现线程间局部变量传递，这该如何实现呢？

其实大佬们早已料到会有这样的需求，于是设计出了 InheritableThreadLocal 类。

InheritableThreadLocal 类的源码除去注释之外一共不超过10行，因为它是继承于 ThreadLocal 类，很多东西在 ThreadLocal 类中已经实现了，InheritableThreadLocal 类只重写了其中三个方法：

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {

    protected T childValue(T parentValue) {
        return parentValue;
    }

    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }
  
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

我们先用一个简单的示例来实践一下父子线程间局部变量的传递功能。

```java
public static void main(String[] args) {
  ThreadLocal<String> threadLocal = new InheritableThreadLocal<>();
  threadLocal.set("这是父线程设置的值");

  new Thread(() -> System.out.println("子线程输出：" + threadLocal.get())).start();
}

// 输出内容
子线程输出：这是父线程设置的值
```

可以看到，在子线程中通过调用 `InheritableThreadLocal#get()` 方法，拿到了在父线程中设置的值。

那么，这是如何实现的呢？

实现父子线程间的局部变量共享需要追溯到 Thread 对象的构造方法：

```java
public Thread(Runnable target) {
  init(null, target, "Thread-" + nextThreadNum(), 0);
}

private void init(ThreadGroup g, Runnable target, String name, long stackSize) {
  init(g, target, name, stackSize, null, true);
}

private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  // 该参数一般默认是 true
                  boolean inheritThreadLocals) {
  // 省略大部分代码
  Thread parent = currentThread();
  
  // 复制父线程的 inheritableThreadLocals 属性，实现父子线程局部变量共享
  if (inheritThreadLocals && parent.inheritableThreadLocals != null) {
   	this.inheritableThreadLocals =
    ThreadLocal.createInheritedMap(parent.inheritableThreadLocals); 
  }
  
	// 省略部分代码
}
```

在最终执行的构造方法中，有这样一个判断：如果当前父线程（创建子线程的线程）的 inheritableThreadLocals 属性不为 NULL，就会将当下父线程的 inheritableThreadLocals 属性**复制**给子线程的 inheritableThreadLocals 属性。具体的复制方法如下：

```java
// ThreadLocal#createInheritedMap
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
  return new ThreadLocalMap(parentMap);
}

private ThreadLocalMap(ThreadLocalMap parentMap) {
  Entry[] parentTable = parentMap.table;
  int len = parentTable.length;
  setThreshold(len);
  table = new Entry[len];
	// 一个个复制父线程 ThreadLocalMap 中的数据
  for (int j = 0; j < len; j++) {
    Entry e = parentTable[j];
    if (e != null) {
      @SuppressWarnings("unchecked")
      ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
      if (key != null) {
        // childValue 方法调用的是 InheritableThreadLocal#childValue(T parentValue)
        Object value = key.childValue(e.value);
        Entry c = new Entry(key, value);
        int h = key.threadLocalHashCode & (len - 1);
        while (table[h] != null)
          h = nextIndex(h, len);
        table[h] = c;
        size++;
      }
    }
  }
}
```

需要注意的是，复制父线程共享变量的时机是在创建子线程时，如果在创建子线程后父线程再往 InheritableThreadLocal 类型的对象中设置内容，将不再对子线程可见（基于 new Entry(key, value) 得到的是两个不同的 Entry 对象）。



## 3. ThreadLocal 内存泄漏分析

最后再来说说 ThreadLocal 的内存泄漏问题，众所周知，如果使用不当，ThreadLocal 会导致内存泄漏。

**内存泄漏**是指程序中已动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。



### 3.1 发生内存泄漏的原因

而 ThreadLocal 发生内存泄漏的原因需要从 Entry 对象说起。

```java
// ThreadLocal->ThreadLocalMap->Entry
static class Entry extends WeakReference<ThreadLocal<?>> {
  /** The value associated with this ThreadLocal. */
  Object value;

  Entry(ThreadLocal<?> k, Object v) {
    super(k);
    value = v;
  }
}
```

Entry 对象的 key 即 ThreadLocal 类是继承于 **WeakReference 弱引用类**。具有弱引用的对象有更短暂的生命周期，**在发生 GC 活动时，无论内存空间是否足够，垃圾回收器都会回收具有弱引用的对象**。

由于 Entry 对象的 key 是继承于 WeakReference 弱引用类的，若 ThreadLocal 类没有外部强引用，当发生 GC 活动时就会将 ThreadLocal 对象回收。

而此时如果创建 ThreadLocal 类的线程依然活动，那么 Entry 对象中 ThreadLocal 对象对应的 value 就依旧具有强引用而不会被回收，从而导致内存泄漏。



### 3.2 如何解决内存泄漏问题

要想解决内存泄漏问题其实很简单，只需要记得在使用完 ThreadLocal 中存储的内容后将它 **remove** 掉就可以了。

这是主动防止发生内存泄漏问题的手段，但其实设计 ThreadLocal 的大神当然也发现了 ThreadLocal 可能引发内存泄漏的问题，所以他们也设计了相应的手段来防止内存泄漏。



### 3.3 ThreadLocal 内部如何防止内存泄漏

#### 3.3.1 set 方法

首先来说 set 方法，其实在上文中描述 `ThreadLocalMap#set(ThreadLocal<?> key, Object value)` 其实已经有涉及 ThreadLocal 内部清理无效 Entry 的逻辑了，：

```java
// ThreadLocalMap#set
private void set(ThreadLocal<?> key, Object value) {
  
  Entry[] tab = table;
  int len = tab.length;
  int i = key.threadLocalHashCode & (len-1);
  
  // 解决哈希冲突，线性探测法
  for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();
		// 代码（1）：更新 Threadlocal 值
    if (k == key) {
      e.value = value;
      return;
    }
		// 代码（2）：探测式清理无效 Entry
    if (k == null) {
      replaceStaleEntry(key, value, i);
      return;
    }
  }
  // 代码（3）：存放键值对，进行启发式清理
  tab[i] = new Entry(key, value);
  int sz = ++size;
  if (!cleanSomeSlots(i, sz) && sz >= threshold)
    // 代码（4）：rehash 中会进行全量清理
    rehash();
}
```

- 首先是**探测式清理**，即代码（2）的位置。在基于线性探测法解决哈希冲突的同时，如果遇到 key 为 null 的情况，会调用 ThreadLocalMap#replaceStaleEntry 方法清理无效 Entry

- 然后是**启发式清理**：在 ThreadLocal 解决完哈希冲突后，将键值对存放在退出线性探测后的位置。然后进行一次启发式清理，在 while 循环中将本次索引下标位置开始不断的右移清理无效的元素，需要注意的是这里会消耗 O(n) 的时间用于插入元素

```java
// 启发式清理：在 在 while 循环中将本次索引下标位置开始不断的右移清理无效的元素，需要注意这里会消耗 O(n) 的时间用于插入元素
private boolean cleanSomeSlots(int i, int n) {
  boolean removed = false;
  Entry[] tab = table;
  int len = tab.length;
  do {
    i = nextIndex(i, len);
    Entry e = tab[i];
    if (e != null && e.get() == null) {
      n = len;
      removed = true;
      i = expungeStaleEntry(i);
    }
  } while ( (n >>>= 1) != 0);
  return removed;
}
```

- 最后如果满足 rehash 条件，会在 rehash 中进行一次**全量清理**：在 for 循环中清理所有 key 为 null 且 value 不为 null 的无效 Entry 对象

```java
private void rehash() {
  // 全量清理
  expungeStaleEntries();

  // Use lower threshold for doubling to avoid hysteresis
  // 扩容
  if (size >= threshold - threshold / 4)
    resize();
}

// 全量清理
private void expungeStaleEntries() {
  Entry[] tab = table;
  int len = tab.length;
  for (int j = 0; j < len; j++) {
    Entry e = tab[j];
    if (e != null && e.get() == null)
      expungeStaleEntry(j);
  }
}
```



#### 3.3.2 get 方法

在 ThreadLocal#get 方法中也存在对无效 Entry 的清理逻辑，源码如下：

```java
// ThreadLocal#get
public T get() {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null) {
    // 实际获取 Entry 对象的方法
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  return setInitialValue();
}

// ThreadLocalMap#getEntry
private Entry getEntry(ThreadLocal<?> key) {
  // 根据寻址算法计算 key 的索引值
  int i = key.threadLocalHashCode & (table.length - 1);
  Entry e = table[i];
  if (e != null && e.get() == key)
    // 若 key 对应索引值的 Entry 对象不为空，且键与当前 ThreadLocal 对象相同，返回该 Entry 对象
    return e;
  else
    // 否则将按照解决哈希冲突时的线性探测法，向后遍历，尝试获取有效且 key 属性与当前 ThreadLocal 对象相同的 Entry 对象，同时若碰到无效 Entry 会进行清理
    return getEntryAfterMiss(key, i, e);
}
```

可以看到，在 ThreadLocalMap#getEntry 方法中，获取 ThreadLocal 中当前线程的局部变量的流程是：

- 首先根据寻址算法计算 key 的索引值
- 若 key 对应索引值的 Entry 对象不为空，且其 key 属性与当前 ThreadLocal 对象相同，返回该 Entry 对象（即返回了在 ThreadLocal 中当前线程存放的局部变量）
- 否则将按照解决哈希冲突时的线性探测法，向后遍历，尝试获取有效且 key 属性与当前 ThreadLocal 对象相同的 Entry 对象，同时若碰到无效 Entry 会进行清理

我们来看下 ThreadLocalMap#getEntryAfterMiss 的具体逻辑：

```java
// ThreadLocalMap#getEntryAfterMiss
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
  Entry[] tab = table;
  int len = tab.length;
 // 当 Entry 为空时结束循环
  while (e != null) {
    ThreadLocal<?> k = e.get();
    // 若 Entry 对象的 key 与当前 ThreadLocal 相同，返回该 Entry 对象
    if (k == key)
      return e;
    // 若 Entry 对象的 key 为空，进行清理
    if (k == null)
      expungeStaleEntry(i);
    else
      // 若 Entry 对象的 key 不为空，计算下一个索引值，继续 while 循环
      i = nextIndex(i, len);
    e = tab[i];
  }
  return null;
}
```

ThreadLocalMap#getEntryAfterMiss 中是一个 while 循环，循环结束条件是 Entry 对象为空，这意味着在 ThreadLocal 中当前线程并没有存放局部变量。在此之前，while 循环中，ThreadLocal 在寻找对应的 Entry 时会有如下逻辑：

- 若 Entry 对象的 key 与当前 ThreadLocal 相同，返回该 Entry 对象（即返回了在 ThreadLocal 中当前线程存放的局部变量）
- 若 Entry 对象的 key 为空，则这是一个 key 为 null 的无效 Entry 对象，可能导致内存泄漏，将执行 ThreadLocalMap#expungeStaleEntry 方法进行清理。
- 若 Entry 对象的 key 不为空，计算下一个索引值，继续 while 循环

最后我们来看一下在 ThreadLocal 中真正执行清理无效 Entry 的 ThreadLocal#expungeStaleEntry 方法的源码：

```java
// ThreadLocalMap#expungeStaleEntry
private int expungeStaleEntry(int staleSlot) {
  Entry[] tab = table;
  int len = tab.length;

  // expunge entry at staleSlot
  // 清理当前无效 Entry 对象
  tab[staleSlot].value = null;
  tab[staleSlot] = null;
  size--;

  // Rehash until we encounter null
  // 执行 rehash 直到遇到 Entry=null
  Entry e;
  int i;
  for (i = nextIndex(staleSlot, len);
       (e = tab[i]) != null;
       i = nextIndex(i, len)) {
    ThreadLocal<?> k = e.get();
    // rehash 过程中若碰到 key=null 的情况，进行清理
    if (k == null) {
      e.value = null;
      tab[i] = null;
      size--;
    } else {
      // 真正的 rehash 过程
      int h = k.threadLocalHashCode & (len - 1);
      if (h != i) {
        tab[i] = null;
        // Unlike Knuth 6.4 Algorithm R, we must scan until
        // null because multiple entries could have been stale.
        while (tab[h] != null)
          h = nextIndex(h, len);
        tab[h] = e;
      }
    }
  }
  return i;
}
```

ThreadLocalMap#expungeStaleEntry 的具体清理逻辑是：

- 首先将当前无效的 Entry 对象 value 置位 null，然后将无效 Entry 对象置位 null
- 随后进行一次 for 循环，for 循环的结束条件是 Entry 对象为 null
    - 执行 rehash 逻辑，在 rehash 过程中，如果碰到 key 为 null 的 Entry 对象，同样会将当前无效的 Entry 对象 value 置位 null，然后将无效 Entry 对象置位 null
    - 若是正常的 Entry 对象，进行 rehash，调整其索引值



#### 3.3.3 remove 方法

同样的，在 ThreadLocal#remove 方法中，也包含着主动清理无效 Entry 对象防止内存泄漏的逻辑，我们来直接看代码：

```java
// ThreadLocal#remove()
public void remove() {
  ThreadLocalMap m = getMap(Thread.currentThread());
  if (m != null)
    m.remove(this);
}

// ThreadLocalMap#remove(ThreadLocal)
private void remove(ThreadLocal<?> key) {
  Entry[] tab = table;
  int len = tab.length;
  // 获取 key 的索引下标值
  int i = key.threadLocalHashCode & (len-1);
  // 循环进行无效 Entry 对象的清理，直到 Entry 为空
  for (Entry e = tab[i];
       e != null;
       e = tab[i = nextIndex(i, len)]) {
    // 若 Entry 的 key 属性对应的正是当前 ThreadLocal 对象
    if (e.get() == key) {
      // 将 Entry 的 key 置为 null
      e.clear();
      // 实际执行清理 Entry 对象的方法
      expungeStaleEntry(i);
      return;
    }
  }
}
```

ThreadLocal#remove 方法在执行时有以下逻辑：

- 首先计算 key 的索引下标值
- 执行一次 for 循环，循环结束的条件是 Entry 为空。在循环过程中，如果找到了 Entry 的 key 属性对应的正是当前 ThreadLocal 对象时，将 key 置为 null，同时调用 ThreadLocal#expungeStaleEntry 方法清理 value



#### 3.3.4  ThreadLocal 内部防止内存泄漏小结

总结一下 ThreadLocal 内部如何防止内存泄漏：

- 在 **set** 元素时会基于线性探测法寻找适合存储 Entry 对象的索引位置，同时会进行**探测式清理**将本次索引下标位置到下一个 Entry 为 null 的位置中的无效 Entry 清理掉。随后会进行**启发式清理**，在 while 循环中将本次索引下标位置开始不断的右移清理无效的元素。最后如果满足 rehash 条件，会在 rehash 中进行一次**全量清理**：在 for 循环中清理所有无效 Entry 对象。
- 在 **get/remove** 方法中会基于**探测式清理**将本次索引下标位置到下一个 Entry 为 null 的位置中的无效 Entry 清理掉。

由此可以看到，无论是 set/get/remove 中哪一个主动对于无效 Entry 的清理动作都是有范围的，并不能完全避免内存泄漏问题，要彻底解决内存泄漏问题还得是养成**使用完就主动调用 remove 方法释放资源**的好习惯。



## 4. ThreadLocal 应用场景及示例

ThreadLocal 在很多开源框架中都有应用，比如：Spring 中的事务隔离级别的实现、MyBatis 分页插件 PageHelper 的实现等。

同时，我在项目中也有基于 ThreadLocal 与过滤器实现接口白名单的鉴权功能。



## 5. 小结

最后以面试题的形式来总结一下关于 ThreadLocal 本文所描述的内容：

- ThreadLocal 解决了哪些问题
- ThreadLocal 底层数据结构
- ThreadLocalMap 的散列方式
- ThreadLocalMap 如何处理哈希冲突
- ThreadLocalMap 扩容机制
- ThreadLocal 如何实现父子线程间局部变量共享
- ThreadLocal 为什么会发生内存泄漏
- ThreadLocal 内存泄漏如何解决
- ThreadLocal 内部如何防止内存泄漏，在哪些方法中存在
- ThreadLocal 应用场景



## 6. 参考资料

- [ThreadLocal源码解读](https://www.cnblogs.com/micrari/p/6790229.html)
- [一篇文章，从源码深入详解ThreadLocal内存泄漏问题](https://www.jianshu.com/p/dde92ec37bd1)

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。