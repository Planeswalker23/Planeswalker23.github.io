---
layout: post
title: 组合模式
categories: [设计模式]
description: 设计模式：组合模式
keywords: 设计模式
---

# 组合模式
> HeadFirst上面的组合模式有点看不大懂啊，于是又去看了看《设计模式之禅》，还是先理解这个模式怎么用，哪种情况用合适，优缺点，然后在日常工作中运用到，才能真正掌握。

## 定义
> Compose objects into tree structures to represent part-whole hirearchies.Composite lets clients treat individual objects and compositions of objects uniformly.
> 将对象组合成树形结构，以表示“部分-整体”的层次结构，使得用户对单个对象和组合对象的使用具有一致性。

## 类图
> 图片来自百度

![组合模式类图](https://gss0.bdstatic.com/94o3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike92%2C5%2C5%2C92%2C30/sign=14709b4c5db5c9ea76fe0bb1b450dd65/c8177f3e6709c93d5024a3b0993df8dcd00054a6.jpg)

## 角色结构
> Component抽象构建角色：可以定义属性和方法
> Leaf叶子构件：遍历的最小单位
> Composite树枝构件：中间节点或根节点

## 实例
> 当下大学中有各种各样的组织，大的如学生会权力大人多，小的如社团，都会有完整的组织架构。
> 就拿学生会来举个例子吧，最大的头头叫主席，下面有各个部门，最底层的是部门的干事。

1. 创建学生的抽象类，它有两本基本属性，一个构造方法，还有一个展示的方法。

````java
public abstract class Student {
    /**
     * 姓名
     */
    private String name;
    /**
     * 身份
     */
    private String identity;

    /**
     * 构造方法
     * @param name
     * @param identity
     */
    public Student(String name, String identity) {
        this.name = name;
        this.identity = identity;
    }
    /**
     * 该学生的职责
     * 个体和整体都具有的共享方法
     */
    public void job() {
        System.out.println(this.name + "的职责是" + this.identity);
    }
}
````

2. 创建部长类，继承学生类，他表示拥有下级的部长及以上等级的学生，他还有增减干事的方法以及展示下级的方法。

````java
public class Minister extends Student {

    /**
     * 部长级干部拥有自己的干事，负责某一部门的运作
     * 主席的下级是部长，负责整个协会的运作
     */
    private List<Student> followers = new ArrayList<>();
    /**
     * 构造方法
     * @param name
     * @param identity
     */
    public Minister(String name, String identity) {
        super(name, identity);
    }

    /**
     * 给部长增加干事
     * @param student
     */
    public void add(Student student) {
        this.followers.add(student);
    }

    /**
     * 给部长减少干事
     * @param student
     */
    public void del(Student student) {
        this.followers.add(student);
    }

    public void getChildren() {
        Iterator<Student> iterator = this.followers.iterator();
        while (iterator.hasNext()) {
            iterator.next().job();
        }
    }
}
````

3. 创建干事类，他没有下级，只有普通的方法。
````java
public class NormalStudent extends Student {
    /**
     * 构造方法
     * @param name
     * @param identity
     */
    public NormalStudent(String name, String identity) {
        super(name, identity);
    }
}
````

4. 创建测试类

````java
public class Test {
    public static void main(String[] args) {
        Minister chairman = new Minister("大王","主席");
        Minister minister = new Minister("小王","宣传部部长");
        NormalStudent student1 = new NormalStudent("一一","宣传部干事1");
        NormalStudent student2 = new NormalStudent("二二","宣传部干事2");
        // 构建组织架构树
        chairman.add(minister);
        minister.add(student1);
        minister.add(student2);
        // 展示树状结构
        chairman.getChildren();
        minister.getChildren();
    }
}
````

- 测试结果为

````$xslt
小王的职责是宣传部部长
一一的职责是宣传部干事1
二二的职责是宣传部干事2
````

## 组合模式的优缺点
### 优点
1. 高层模块调用简单
2. 节点自由增加

### 缺点
1. 场景类直接使用了树叶节点和树枝节点，这在面向接口编程上是很不恰当的，与依赖原则冲突
   

## 使用场景
1. 维护和展示“部分-整体”关系的场景——树形菜单、文件和文件夹管理
2. 从一个整体中能够独立出部分模块或功能的场景

## 源代码
[GitHub地址](https://github.com/Planeswalker23/all-in-one/tree/master/design-patterns/src/main/java/org/planeswalker/composite)