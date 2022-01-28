---
layout: post
title: 我所理解的其他问题·第12篇·问了三位候选人反转链表这道题
categories: [算法]
keywords: 算法
---



公司出台了一项新政策，后面的面试都需要候选人进行实际编码，可以是算法题，也可以是业务架构类题，反正面试的结果中得有代码产出。对于众多像我这样平庸的一面面试官来说，leetcode上随便扒一道算法题或许就是最简单的应对措施了。

恰巧曾经在某个技术论坛上看到过一段话：“反转链表这道题目，看似简单，实际上能好好写出来的人没几个。”抱着将信将疑的态度，我连续三次将这道题目作为候选人的一面算法题，结果确实不假。

第一位候选人磨了很久，最终写完后，连编译都没有通过。第二位同学可能是有算法底子，很快写出了迭代解法。第三位同学没有用传统的迭代或递归，而是用空间换时间，以一种类似暴力的解法解了出来，我觉得他属于出奇制胜的类型。

其实之前我也做过这道题目，对于不常写算法的同学来说，确实比较难解。下面就来讨论一下反转链表的三种解法：**暴力、迭代、递归**。



## 1. 题目

首先来看下 leetcode 上的原题：[反转链表](https://leetcode-cn.com/problems/fan-zhuan-lian-biao-lcof/)。

定义一个函数，输入一个链表的头节点，反转该链表并输出反转后链表的头节点。

示例：

```text
输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL
```

链表节点的数据结构为：

```java
public class ListNode {
  int val;
  ListNode next;
  ListNode(int x) { val = x; }
}
```



## 2. 解法1：暴力

我不知道这算不算暴力，我觉得挺暴力的。第三位候选人的思路是以时间换空间，先把所有节点存储在列表中，然后反向遍历，同时更改链表节点的 next 属性，最后返回。



### 2.1 代码实现

其代码实现如下：

```java
public static ListNode reverseList(ListNode head) {
    // 先定义一个初始节点，最终应返回该节点的 next 节点
    ListNode result = new ListNode(0);
    ListNode cur = head;
    ListNode temp = result;
    // 缓存链表中的每一个节点
    List<ListNode> list = new ArrayList<>();
    while (cur != null) {
        list.add(cur);
        cur = cur.next;
    }
    // 反向遍历
    for (int i = list.size() - 1; i >= 0; i--) {
        ListNode node = list.get(i);
        // 新建节点，修改引用关系
        temp.next = new ListNode(node.val);
        temp = temp.next;
    }
    return result.next;
}
```

这种解法值得一提的就是**用空间换时间**，在反向遍历过程中，通过 new 一个新的 ListNode 节点来承载原来节点的 val 属性，next 属性在下一次循环中进行变更，同时使用 new 新的 ListNode 节点也可以也可以较好的解决迭代法中较难理解的变更指针问题。

**时间复杂度：O(n)，空间复杂度：O(n)，对链表进行2次循环。**

该解法在 leetcode 上的执行结果如下：

![解法1执行结果](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123404176-7c8e385e-6a81-4b9d-bb94-1c3677152747.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_46%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fresize%2Cw_1500%2Climit_0)



### 2.2 图解

我们用图的方式来分析一下反向遍历中每一次循环节点的对应关系：

开始循环前，上方为暂存于 list 列表的 ListNode 节点，下方是准备返回的 result 链表，初始节点为0。

![反转链表-暴力](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123409089-2520fccb-addf-467b-9279-27f903fb157d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_33%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

第1次循环，从 list 列表的最后一个元素5开始遍历，新建一个 ListNode 节点，val 值为5，暂未指定 next 属性，将 temp 节点的 next 属性指向新建的节点，然后 temp 节点向后移动一位（temp = temp.next）。

![反转链表-暴力](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123415245-db1f7b4d-0704-466f-ad6f-cb0294337fad.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_33%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

第2次循环，遍历到 list 列表的倒数第二个元素4，新建一个 ListNode 节点，val 值为4，将 temp 的 next 属性指向新建的节点，然后 temp 节点向后移动一位。

![反转链表-暴力2](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123419701-3a998370-1b20-4abf-a59f-4b490a90e307.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_33%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

第3次循环，遍历到 list 列表的倒数第三个元素3，新建一个 ListNode 节点，val 值为3，将 temp 的 next 属性指向新建的节点，然后 temp 节点向后移动一位。

![反转链表-暴力3](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123426189-1cdecd53-94a8-41f6-920d-40e66da70d2c.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_33%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

第4次循环，遍历到 list 列表的倒数第四个元素2，新建一个 ListNode 节点，val 值为2，将 temp 的 next 属性指向新建的节点，然后 temp 节点向后移动一位。

![反转链表-暴力4](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123431857-cf60b099-11a5-440b-b55d-0afef55c8a47.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_33%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

第5次循环，遍历到 list 列表的倒数第五个元素1，新建一个 ListNode 节点，val 值为2，将 temp 的 next 属性指向新建的节点，然后 temp 节点向后移动一位。

![反转链表-暴力5](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123436581-e7c2445f-7863-4fa7-9d0d-da41364f94f2.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_33%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

最后，遍历结束，返回 result 对象的 next 属性，将下方链表中值为 0 的后一位对应的链表作为返回，最终完成了链表的反转。



## 3. 解法2：迭代

第二种解法是迭代法，其实也用到了双指针。在进行链表遍历时，借助双指针修改各节点的地址指向。



### 3.1 代码实现

其代码实现如下：

```java
public ListNode reverseList(ListNode head) {
    ListNode cur = head;
    ListNode pre = null;
    while(cur != null) {
      // temp 暂存 cur.next，防止后继节点在 cur.next 地址变更后从循环中消失
      ListNode temp = cur.next;
      // 修改 cur.next 引用指向
      cur.next = pre;
      // pre 暂存当前遍历节点 cur
      pre = cur;
      // cur 访问下一节点，使得循环向下一节点进行
      cur = temp;
    }
    return pre;
}
```

迭代法是比较通用的解法，但是它比较难想，如果对链表不甚了解的同学很难想出这样的解法。

**时间复杂度：O(n)，空间复杂度：O(1)，对链表进行1次循环。**

该解法在 leetcode 上的执行结果如下：

![解法2执行结果](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123443115-e7546111-eaca-4609-b2f6-fb74899f6a4f.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_45%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fresize%2Cw_1500%2Climit_0)



### 3.2 图解

我们用图的方式来分析一下反向遍历中每一次循环节点的对应关系。

开始循环前，声明两个“指针”，一个 cur 节点指向头节点 head，另一个 pre 节点为 null，也可以把它看成是尾节点指向的位置，表示当前节点的前继节点。

![反转链表-迭代](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123448685-6c840a35-4f74-492a-93a6-482e7edb2d5e.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_35%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

第1次循环，cur 节点为头节点，即值为1的节点。声明一个新的变量 temp 用于暂存链表的后继节点，防止在双指针变更指向时丢失后继链表，然后将 cur 节点的后继节点改为 pre。此时指针指向变更完成，将 cur 与 pre 向后移动一位，先将 pre 赋值为 cur，再将 cur 赋值为之前暂存原始链表第二个节点的 temp 变量。

至此，第一次循环结束，此时链表及双指针状态如下：

![反转链表-迭代1](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123452845-4a12148f-e39e-465f-b9a4-ca955c2c34ee.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_35%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

第2次循环，cur 节点是值为2的节点，将 cur 后继节点赋值给变量 temp，然后将然后将 cur 节点的后继节点改为 pre，然后将 cur 与 pre 向后移动一位，先将 pre 赋值为 cur，再将 cur 赋值为之前暂存原始链表第二个节点的 temp 变量。

至此，第二次循环结束，此时链表及双指针状态如下：

![反转链表-迭代2](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123459498-50163412-bef2-428a-b1f0-bde969f35950.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_35%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

同样道理，第3次循环结束后，链表状态、各指针地址如下：

![反转链表-迭代3](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123464772-86aab66b-6352-4044-b1bf-087aa6f355ba.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_35%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

第4次循环结束后，链表状态、各指针地址如下：

![反转链表-迭代4](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643123472968-e4e28f80-79a0-4143-b7bc-442662ff56fb.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_35%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

第5次循环结束后，链表状态、各指针地址如下：

![反转链表-迭代5](https://cdn.nlark.com/yuque/0/2022/png/2331602/1643369090468-c1c781f4-0f9c-4dd5-a814-86c6582cda25.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_35%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

最后，cur 为 null，循环结束，返回 pre 对象，完成了链表的反转。



## 4.  解法3：递归

第三种解法是递归法，这种解法更难想到，本人不才，也是看了 leetcode 的题解和图解才看懂的。

递归中最重要的就是找到递归的终止条件以及对节点的处理，在反转链表中前者无疑是当节点为空时终止递归，后者的要点就是将当前节点的后继节点改为前继节点就可以。



### 4.1 代码实现

其代码实现如下：

```java
public ListNode reverseList(ListNode head) {
  	return recur(head, null);    // 调用递归并返回
}
private ListNode recur(ListNode cur, ListNode pre) {
    if (cur == null) return pre; // 终止条件
    ListNode res = recur(cur.next, cur);  // 递归后继节点
    cur.next = pre;              // 修改节点引用指向
    return res;                  // 返回反转链表的头节点
}
```

**时间复杂度：O(n)，空间复杂度：O(N)，对链表进行1次循环。**



### 3.2 图解

递归法图解[在 leetcode 上排行第二的精选题解的 PPT 图解](https://leetcode-cn.com/problems/fan-zhuan-lian-biao-lcof/solution/jian-zhi-offer-24-fan-zhuan-lian-biao-die-dai-di-2/)就描绘得十分清楚明白，这里就不再继续画了，没别的，主要是画这图解挺费时间的。

总结一下，递归就是：看着挺简单的，想出来挺难的，复杂度也挺高的。



## 5. 小结

本文总结了反转链表的暴力、迭代、递归三种解法，并作出图解。反转链表这道题，解出来其实不复杂，就是迭代、递归这种基于链表的“指针”式处理比较烧脑。

总的来说，算法还是不能丢！

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://link.juejin.cn/?target=https%3A%2F%2Fwww.yuque.com%2Fplaneswalker%2Fbankend)，欢迎来访。