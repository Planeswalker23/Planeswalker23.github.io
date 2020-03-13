---
layout: post
title: 装饰者模式
categories: [设计模式]
description: 设计模式：装饰者模式
keywords: 设计模式
---

# 装饰者模式

## 定义
> Attach additional responsibilities to an object dynamically keeping the same interface.Decorators provide a flexible alternative to subclassing for extending functionality.<br>
> 动态地给一个对象添加一些额外的职责。就增加功能来说，装饰模式相比生成子类更为灵活。<br>

## 类图
> 图片来自百度百科

![装饰者模式类图](https://user-gold-cdn.xitu.io/2019/8/31/16ce64aa0e67dd40?w=521&h=409&f=png&s=68245)

## 角色结构
> Component抽象构件:是一个接口或者是抽象类,就是定义我们最核心的对象,也就是最原始的对象。<br>
> ConcreteComponent具体构件:是最核心、最原始、最基本的接口或抽象类的实现,你要装饰的就是它。<br>
> Decorator装饰角色:一般是一个抽象类,做什么用呢?实现接口或者抽象方法,它里面可不一定有抽象的方法呀,在它的属性里必然有一个private变量指向Component抽象构件。<br>
> 具体装饰角色:ConcreteDecoratorA和ConcreteDecoratorB是两个具体的装饰类,你要把你最核心的、最原始的、最基本的东西装饰成其他东西。<br>

## 实例
> 现在很多人都喜欢喝奶茶，每天一杯有益身心健康(主要是心和嘴)。当我们在喝奶茶的时候，服务员总会问"要加什么?"，这跟每天的终极难题--吃什么--是一样，今天暂且不谈。而供选择的材料可能有波霸椰果布丁等，加了椰果的奶茶最后就被标为椰果奶茶售卖。今天就用买奶茶作为装饰者模式的例子。

1. 首先创建一个制作奶茶的原料抽象类`Material`。
````java
public abstract class Material {
    /**
     * 该奶茶原料的名字
     */
    private String name;

    /**
     * 获取已加入的原料
     */
    public String getFullName() {
        return this.name;
    }
    /**
     * 费用
     */
    public abstract double cost();

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
````

2. 然后创建配料抽象类`Burden`，继承于原谅抽象类。
````java
public abstract class Burden extends Material {

    /**
     * 配料类需要持有一个被修饰类，即奶茶
     */
    private Material material;

    /**
     * 配料类需要重写获取全部原料的方法，为了获得完成的原料
     * @return
     */
    public abstract String getFullName();

    public Material getMaterial() {
        return material;
    }

    public void setMaterial(Material material) {
        this.material = material;
    }
}
````

3. 分别创建具体的原料和配料
````java
public class MilkTea extends Material {

    public MilkTea() {
        this.setName("奶茶");
    }

    @Override
    public double cost() {
        return 5;
    }
}

public class Pudding extends Burden {

    public Pudding(Material material) {
        this.setMaterial(material);
    }

    @Override
    public String getFullName() {
        return this.getMaterial().getFullName() + " + 布丁";
    }

    @Override
    public double cost() {
        return 2 + this.getMaterial().cost();
    }
}
````

4. 创建测试类
````java
public class Test {

    public static void main(String[] args) {
        // 买一杯奶茶，不加任何配料
        Material milkTea = new MilkTea();
        System.out.println("奶茶费用 = " + milkTea.cost());

        // 买一被加椰果的奶茶
        Material milkTeaWithCoCo = new MilkTea();
        milkTeaWithCoCo = new NataDeCoco(milkTeaWithCoCo);
        System.out.println("椰果奶茶费用 = " + milkTeaWithCoCo.cost());

        // 买一被加布丁的玛奇朵
        Material MacchiatoWithPudding = new Macchiato();
        MacchiatoWithPudding = new Pudding(MacchiatoWithPudding);
        System.out.println("布丁玛奇朵费用 = " + MacchiatoWithPudding.cost());
    }
}
````
- 测试结果为
````$xslt
奶茶费用 = 5.0
椰果奶茶费用 = 6.0
布丁玛奇朵费用 = 12.0
````

## 装饰者模式的优缺点
### 优点
1. 在不影响其他对象的情况下,动态为单个对象新增功能。
2. 装饰类与被装饰类 (ConcreteComponent) 相互独立,互不耦合,易于扩展。
3. 代替继承方式的功能实现,减少继承类的存在。
   
### 缺点
1. 多层的装饰是比较复杂的，且不容易理解，参照Java的IO类。

## 使用场景
1. 需要扩展一个类的功能,或给一个类增加附加功能。
2. 需要动态地给一个对象增加功能,这些功能可以再动态地撤销。
3. 需要为一批的兄弟类进行改装或加装功能,当然是首选装饰模式。

## 源代码
[GitHub地址](https://github.com/Planeswalker23/all-in-one/tree/master/design-patterns/src/main/java/org/planeswalker/decorator)