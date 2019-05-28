---
title: 牛客网 > leetcode > 链表算法 题目总结
date: 2019-01-13 19:41:13
tags:
	- 牛客网
    - 链表
---



## 链表

* 长度的游戏，多用快慢指针来实现，因为 2X-X = △k， 即X=△k，找到△k的意义，再在慢指针x上面接着做文章即可。举例 ： “链表中倒数第k个结点”，“两个链表的第一个公共结点”，“链表中环的入口结点”
* 指针的游戏，多加几个指针来解决问题，而不是用hashset, ArrayList 等容器来装。举例： “反转链表”，“合并两个排序的链表”，“复杂链表的复制”，“删除链表中重复的节点”


## 例题
牛客网上 [leetCode](https://www.nowcoder.com/ta/leetcode) 在线练习部分 全部 链表 算法汇总，索引到我的笔记

| | 题目 | 考点 |
|:-----------:| :-------------|:-------------|
| 1 |  [从尾到头打印链表](https://www.nowcoder.com/profile/1923750/note/detail/307114?noteType=0)  | 使用递归的方法，入栈，从尾到头打印连表 | 
| 2 | [链表中倒数第k个结点](https://www.nowcoder.com/profile/1923750/note/detail/311842?noteType=0)	 | 链表，一个长度的问题，怎么找到那个 “△k” 成为了用“双点”的思路来源 | 
| 3 | [反转链表](https://www.nowcoder.com/profile/1923750/note/detail/311879?noteType=0) | 想到的是用一个容器来装链表，那就不太是 最佳思路； 应当用链表来解决链表 | 
| 4 | [合并两个排序的链表](https://www.nowcoder.com/profile/1923750/note/detail/316419?noteType=0)	 | 设置3个指针，注意应用变量的问题 | 
| 5 | [复杂链表的复制](https://www.nowcoder.com/profile/1923750/note/detail/316560?noteType=0)	 | 用链表自己的属性来解决，先拼成一个长链表，再拆分 | 
| 6 | [两个链表的第一个公共结点](https://www.nowcoder.com/profile/1923750/note/detail/316578?noteType=0)	 | 长度法：list1与list2的长度差很关键。先让长的走△步，然后一起遍历，第一个相同的就是要找的；时间复杂度 O(n+m)| 
| 7 | [链表中环的入口结点](https://www.nowcoder.com/profile/1923750/note/detail/316582?noteType=0)	 | 龟兔赛跑法（快慢指针，本质是玩长度的游戏） | 
| 8 | [删除链表中重复的节点](https://www.nowcoder.com/profile/1923750/note/detail/316634?noteType=0)	 | 设置 pre ，last 指针， pre指针指向当前确定不重复的那个节点，而last指针相当于工作指针，一直往后面搜索。 | 
|