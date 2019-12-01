---
layout: post
title: 单例模式
categories: [设计模式]
description: 设计模式：单例模式
keywords: 设计模式, singleton
---

## 背景
> 最近开始学习设计模式，准备从最简单的单例模式开始写博客。<br>
> 单例模式可以细分为懒汉式、饿汉式、静态内部类、枚举、容器五种创建方法。这篇文章就介绍单例模式的各种写法。

## 懒汉式
> 单例模式-懒汉式，顾名思义就是懒人的创建对象方式，只有当用到这个类的时候才去实例化对象，如果不用到，只声明对象，并不会实例化对象。

### 1、懒汉式：线程不安全，适用于单线程
```java
public class Singleton1 {

    // 声明单例对象为private，并不进行实例化
    private static Singleton1 instance = null;

    // 同时声明类的构造方法为private，使得无法在此类外进行实例化
    private Singleton1() { }

    // 获得单例对象的方法
    public static Singleton1 getInstance() {
        // 判断对象是否已实例化，没有实例化才new一个新的对象
        if (null == instance) {
            instance = new Singleton1();
        }
        return instance;
    }
}
```
- 缺点：线程不安全，当两个线程同时运行到`if (null == instance)`时，可能出现两个线程同时判断为true的情况，可能获取到两个不同的Singleton1实例，不满足单例条件。

### 2、懒汉式，使用synchronized同步锁：线程安全，适用于多线程，但效率低
> 为了防止多线程不安全的情况，可以使用synchronized关键字修饰获取实例的静态方法，但这个方法会引起线程阻塞，影响程序性能，效率不高。

```java
public class Singleton2 {

    // 声明单例对象为private，并不进行实例化
    private static Singleton2 instance = null;

    // 同时声明类的构造方法为private，使得无法在此类外进行实例化
    private Singleton2() { }

    // 获得单例对象的方法
    public static synchronized Singleton2 getInstance() {
        // 判断对象是否已实例化，没有实例化才new一个新的对象
        if (null == instance) {
            instance = new Singleton2();
        }
        return instance;
    }
}
```
- 总结：适用于多线程，但加锁后开销加大，获取实例的方法效率低

### 3、懒汉式-双重检验锁：线程安全，适用于多线程，且效率高
> 效率低的问题可以通过修改获取实例静态方法实现：只有在实例未被初始化时才加锁，加锁后再次判断实例是否已被初始化。

```java
public class Singleton3 {

    // 声明单例对象为private，并用volatile关键字修饰，保证可见性
    private static volatile Singleton3 instance = null;

    // 同时声明类的构造方法为private，使得无法在此类外进行实例化
    private Singleton3() { }

    // 获得单例对象的方法
    public static Singleton3 getInstance() {
        // 判断对象是否已实例化，没有实例化才new一个新的对象，第一次判断是为了不必要的同步
        if (null == instance) {
            // 加锁，准备创建实例
            synchronized (Singleton3.class) {
                // 再次判断实例是否已被实例化
                if (null == instance) {
                    instance = new Singleton3();
                }
            }
        }
        return instance;
    }
}
```
- 总结：资源利用率高，第一次加载时才创建实例，效率高，但第一次获取实例稍慢，且会可能出现DCL失效

## 饿汉式
> 饿汉式的意思是，类加载器加载类的时候就初始化对象创建实例，不论后面是否用到这个类。

### 4、饿汉式
````java
public class Singleton4 {

    private static Singleton4 instance = new Singleton4();

    private Singleton4() {
    }

    public static Singleton4 getInstance() {
        return instance;
    }
}
````
- 优点：基于类加载机制避免了多线程的同步问题，获取对象速度快（直接返回对象）
- 缺点：在类加载时就创建实例导致类加载慢，没有达到懒加载的效果，同时不能确定是否有其他方法导致二次类加载

## 静态内部类
> 在单例类的静态内部类中声明单例对象，并实例化。<br>
> 属于懒汉式。

### 5、静态内部类：推荐的写法
````java
public class Singleton5 {

    // 同时声明类的构造方法为private，使得无法在此类外进行实例化
    private Singleton5() { }

    // 获得单例对象的方法
    public static Singleton5 getInstance(){
        return SingletonHolder.instance;
    }

    // 静态内部类，调用getInstance方法时，加载静态内部类，创建实例
    private static class SingletonHolder {
        private static final Singleton5 instance = new Singleton5();
    }
}
````
- 总结：第一次加载类时并不会初始化instance，只有第一次调用获取实例方法时虚拟机加载静态内部类并初始化instance,这样不仅能确保线程安全也能保证类的唯一性，所以推荐使用静态内部类单例模式。

## 枚举类
### 6、枚举类单例
```java
public enum Singleton6 {

    INSTANCE;

    // 业务方法
    public void serviceMethod() { }
}
```
- 属于饿汉式
- 优点：简单，枚举实例默认线程安全，单例，反序列化不会生成新的对象
- 缺点：少用，可读性不高

### 验证单例模式的反序列化
```java
public class SingletonSerializableCheck {

    /**
    * 序列化单例对象
    * 把对象转换为字节序列的过程称为对象的序列化
    * @param instance
    * @param filePath
    * @throws Exception
    */
    private static void writeEnum(Object instance, String filePath) throws Exception {
        ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream(new File(filePath)));
        outputStream.writeObject(instance);
        outputStream.close();
    }

    /**
    * 单例对象反序列化
    * 把字节序列恢复为对象的过程称为对象的反序列化
    * @param filePath
    * @return
    * @throws Exception
    */
    private static Object readEnum(String filePath) throws Exception {
        ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream(new File(filePath)));
        return inputStream.readObject();
    }
}
```
- 其他的单例模式写法会出现反序列化生成两个单例对象的情况，将上面验证代码声明单例对象改成`Object checkedSingletonObj = Singleton1.getInstance();`，输出false，说明反序列化生成了新对象。

### 如何避免反序列化生成新对象？
- 单例类应提供`readResolve()`方法以控制对象的反序列化
```text
    /**
     * 控制反序列化，防止反序列化生成新对象
     * @return singleton.Singleton1
     * @throws ObjectStreamException
     */
    private Object readResolve() throws ObjectStreamException {
        return instance;
    }
```

## 基于容器实现
> 如HashMap，因为HashMap中的key是不可重复的

### 7、基于容器实现单例
```java
public class Singleton7 {

    private static Map<String, Object> objMap = new HashMap<>();

    public static void registerService(String key, Object instance) {
        if (!objMap.containsKey(key)) {
            objMap.put(key, instance);
        }
    }

    public static Object getService(String key) {
        return objMap.get(key);
    }
}
```

源代码地址 [GitHub](https://github.com/Planeswalker23/java-day-day-up/tree/master/design_patterns/singleton)