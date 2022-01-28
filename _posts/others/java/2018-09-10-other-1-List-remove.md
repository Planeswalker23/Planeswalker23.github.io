---
layout: post
title: 我所理解的其他问题·第1篇·List#remove方法失效
categories: [Java]
keywords: Java, List
---



## 1. 开篇词
最近在写增删改查的时候遇到一个问题，苦思冥想了很久，最后旁边的小哥哥一句“看源码就知道了”，遂恍然大悟，翻阅源码才终于搞清楚这个问题。晚上趁着还早，花点时间记录下来。



### 1.1 实体类

```java
public class User {
    private String name;
    private Integer age;
    ……
    //省略setter&&getter
}
```



### 1.2 测试代码

List 容器中泛型为 User 类型，存在5个不同年龄 User 对象。需求是这样的：需要按照 List 容器中元素的年龄属性删除匹配的元素。

```java
public static void main(String[] args) {
        List<User> userList = new ArrayList<>();
        userList.add(new User("小王", 18));
        userList.add(new User("小赵", 19));
        userList.add(new User("小钱", 20));
        userList.add(new User("小孙", 21));
        userList.add(new User("小李", 22));

        List<Integer> ageList = new ArrayList<>();//需要遍历匹配的属性
        ageList.add(18);
        ageList.add(20);
        ageList.add(22);

        List<Integer> matchList = new ArrayList<>();//匹配结果对应list索引的集合
        
        for (int i = 0; i < userList.size(); i++)
            if (ageList.contains(userList.get(i).getAge()))
                matchList.add(i);//遍历用户List进行匹配，将匹配的索引加入matchList
        
        for (int i = matchList.size()-1; i >=0; i--)
            userList.remove(matchList.get(i));//遍历matchList对userList执行remove方法

        System.out.println("userList.size() = " + userList.size());

    }
```



## 2. 问题

 - 测试代码的执行结果为 `userList.size() = 5`，即 remove 方法没有将相应的索引值删掉！
 - 在 remove 之前我还考虑过如果先将容器前面的元素删除则会使它的 size 发生变化，可能会导致删错。这个问题的解决方案是：
  1. 在每次执行remove方法后，对i减1
  2. 从后往前执行remove



## 3. 深究源码

 - List接口的remove方法有两个重载，一个参数是Object，一个参数是int。

```java
public interface List<E> extends Collection<E> {

	......
	boolean remove(Object o);
	......
	E remove(int index);
	......
}
```

 - 很显然在测试代码中 remove 没有将匹配的元素删除的原因就是重载到了 `boolean remove(Object o)` 这个方法。

 - 那么，为什么会重载到参数为 Object 类型的 remove 方法？



## 4. 原因

我将匹配结果的索引值存放在 List 容器中，而容器是无法存放基本数据类型的。所以只能存放 Integer 类型的参数，我将 Integer 类型的参数从容器 matchList 中取出来，get 方法的返回值是 Integer 类型，所以把一个 Integer 类型的参数放到 remove 方法中，重载的自然是参数为 Object 类型的方法，所以也就出现了 remove 方法没有按照预期发展的结果。



## 5. 解决方案



### 5.1 方案一

在 get 方法后加上 intValue() 方法将 Integer 类型转化为 int

```java
userList.remove(matchList.get(i).intValue());//遍历matchList对userList执行remove方法
```



## 5.2 方案二

不需要将匹配到需要删除的元素索引存在 ArrayList 中，直接删除，然后将索引值-1就可以

```java
for (int i = 0; i < userList.size(); i++)
    if (ageList.contains(userList.get(i).getAge()))
        userList.remove(i--);//先删除匹配的元素，然后将索引值i减1
```

然后打印的结果就是 userList.size() = 2 了。



## 6. 总结

以后如果想要实现的效果与预期不符，可以尝试开始看源码了，毕竟那么多方法，记不全的。搞不好一个参数类型不对，全满皆输。

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。