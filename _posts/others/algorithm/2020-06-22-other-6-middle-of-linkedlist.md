---
layout: post
title: 我所理解的其他问题·第6篇·如何找到链表的中间节点
categories: [算法]
keywords: 算法
---



## 1. 碎碎念

遥想后端君当年，曾经也是学校ACM队的一员，但参加过级别最高的比赛，同时也是ACM方面获得的最大成就，不过是天梯赛三等奖（当时天梯赛在浙江还只是省B级别的，现在已经算国赛了），犹记得当时的分数是120多分，满分是200还是160来着我忘记了，反正成绩比平均分高了大概20分。

现在回想起来，总体还是感觉自己是个打酱油的，普通的算法题做做还行，难的比如动态规划、贪心、搜索、图论啥的，就不太会了。

![1](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642854508203-97f99423-fa0b-4158-a2fd-68f4d6f89f50.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_10%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

而如今的大厂面试，几乎都会有笔试轮，让现场手写算法题，要是笔试都没过，那与面试官手撕HashMap、JVM、各种框架原理都没有机会上演。由此可见算法的重要性，所以后端君决定开始重学数据结构与算法，每天都会抽出一个小时时间来练习。

![2](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642854514416-9fc49177-9bbd-43a4-b210-312a893514e5.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_11%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



## 2. 题目

今天练习的主题是快慢指针法，对应LeetCode上的题目是[第876题——链表的中间结点](https://leetcode-cn.com/problems/middle-of-the-linked-list/)。

题目的要求是给定一个链表的头结点head，返回这个链表的中间节点。

给定的链表数据结构为

```java
public class ListNode {
     int val;
     ListNode next;
     ListNode(int x) { val = x; }
}
```

例如，当链表为`[1,2,3,4,5]`时，返回的是值为3的节点；当链表为`[1,2,3,4,5,6]`时，返回的是值为4的节点。

当然我们可以先遍历一遍链表，得到链表的长度，然后计算出中间节点的索引值，最终通过迭代去获得中间节点。

这样虽然最终也能得到结果，但怎么说呢，不优雅。

![3](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642854520720-8ef5853e-029b-4bbe-b6ca-198ed96286b4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_11%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

而下面要讲的快慢指针法，只需要通过一次遍历就能够得到答案，相对来说就比较优雅且高效。



## 3. 快慢指针法

快慢指针法在寻找链表中间节点这道题上的的思路是这样：通过两个指针遍历链表，快指针每次走2步，慢指针每次走1步，那么当快指针走到链表尾部的时候，慢指针恰好走到链表的中间节点，然后遍历结束，返回慢指针即为链表中间节点。

先来看第一个示例，当链表为`[1,2,3,4,5]`时，灵魂画手上线，这时链表是这样子的（我们用白色的箭头来表示慢指针，黑色的箭头来表示快指针）：

![第1次移动后的链表](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642854526046-5a26bd61-7d7a-41bf-918a-b962453ff6e8.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_14%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

当两个指针经过第1次移动后，快指针走到3的位置，慢指针走到2的位置，这时链表是这样子的：

![第1次移动后的链表](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642854562160-dc80c807-036f-42d5-bf85-c2ce71821b2c.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_14%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

当两个指针经过第2次移动后，快指针走到5的位置，慢指针走到3的位置，这时链表是这样子的：

![第2次移动后的链表](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642854567517-5f9ebe0e-bf10-4634-97aa-c7abbc2eae9b.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_14%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

当要进行第3次移动到，发现快指针已经走到了链表尾部节点，即`fast.next == null`，于是退出遍历，直接返回慢指针，也就是值为3的节点。

这时当链表节点数为奇数的时候的题解步骤，若节点数为偶数，就有一些不同的，比如链表为`[1,2,3,4,5,6]`时，在第3次移动后快指针走到了NULL指针的位置，慢指针走到了4的位置，如下图所示：

![第3次移动后的链表](https://cdn.nlark.com/yuque/0/2022/png/2331602/1642854576003-33228533-f108-4e4d-ab0f-c53659580a32.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_19%2Ctext_6K-t6ZuA77ya5oiR5omA55CG6Kej55qE5ZCO56uv5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

这时候的判断就不是快指针走到链表尾部节点了，而是快指针本身就是一个空节点。

所以综合链表长度为奇数和偶数的两种情况，遍历链表结束的条件应该是：快指针为空或快指针走到链表尾部节点（即快指针的下一个阶段为空）。

所以，这道题的快慢指针解法应该是下面这样子。

```java
public ListNode middleNode(ListNode head) {
    // 初始化快慢指针，全部指向头结点
    ListNode fast = head,slow = head;
    // 遍历结束的条件是快指针为空或快指针走到链表尾部节点
    while (fast !=null && fast.next!=null) {
        // 快指针走2步，慢指针走1步
        fast = fast.next.next;
        slow = slow.next;
    }
    // 返回慢指针
    return slow;
}
```



## 4. 拓展

寻找链表中间节点只是快慢指针法的一种应用，快慢指针法还能够用在很多算法题中，比如寻找[链表中倒数第k个节点](https://leetcode-cn.com/problems/lian-biao-zhong-dao-shu-di-kge-jie-dian-lcof)、判断一个链表是不是[环形链表](https://leetcode-cn.com/problems/linked-list-cycle/)等等。

这两题的分别是剑指Offer第22题和Leetcode第141题，有兴趣的同学可以去做做。



## 5. 小结

本文通过LeetCode上的一道寻找链表中间节点的题目来描述了一下快慢指针法，这是一个普通却又并不简单的算法。后端君认为，在应用快慢指针法时，我们需要注意的是遍历结束条件以及快慢指针分别如何移动，在确定了这两个步骤之后才能够真正运用这个算法来解决问题。

今天的快慢指针法，你学废了吗？

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。