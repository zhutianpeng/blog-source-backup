---
title: 牛客网>leetcode>二叉树算法 题目总结
date: 2019-02-28 16:58:47
tags:
	- 牛客网
    - 数组
---

牛客网上 [leetCode](https://www.nowcoder.com/ta/leetcode) 在线练习部分 全部二叉树算法汇总，索引到我的笔记

# 数组
- 数组是属于最基础的数据结构，比较灵活。
- 首先注意是否是排序？递增？非递减？乱序？的数组。 有序的数组的查找问题想到用“二分查找”，
- 数组中多用到双指针滑动窗口（和为S的连续正数序列），或者双指针夹逼(eg:和为S的两个数字)，或者带flag的双指针夹逼（eg:蓄水池问题）

# 例题
牛客网上 [剑指offer](https://www.nowcoder.com/ta/coding-interviews) 在线练习部分 全部 数组 算法汇总，索引到我的笔记

## 二叉树

| | 题目 | 考点 |
|:-----------:| :-------------:|:-------------:|
| 1 |  [数组中重复的数字(剑指3)](https://www.nowcoder.com/profile/1923750/note/detail/324621?tags=%E6%95%B0%E7%BB%84)  | 数组中的数字在0到n-1的范围内。ztp:hashset或hashmap辅助;  标准解法：利用数字大小的特点，众神归位，谁的位子被占了就是重复的（节省空间） | 
| 2 | [二维数组中的查找（剑指4）](https://www.nowcoder.com/profile/1923750/note/detail/307089?tags=%E6%95%B0%E7%BB%84)	 | 找到唯一方向的路径（从左下角开始，比input大则上移，比input小则右移。递归。）| 
| 3 | [旋转数组的最小数字(剑指11)](https://www.nowcoder.com/profile/1923750/note/detail/307177?tags=%E6%95%B0%E7%BB%84) | 对非减排序的数组的一个旋转，再找到最小值：二分查找 O(logn)| 
| 4 | [数组中出现次数超过一半的数字(剑指39)](https://www.nowcoder.com/profile/1923750/note/detail/316936?tags=%E6%95%B0%E7%BB%84)	 | ztp: hashmap辅助操作。 标准：摩尔投票法(每次从序列里选择两个不相同的数字删除掉（或称为“抵消”），最后剩下一个数字或几个相同的数字，就是出现次数大于总数一半的那个) | 
| 5 | [连续子数组的最大和(剑指42)](https://www.nowcoder.com/profile/1923750/note/detail/316964?tags=%E6%95%B0%E7%BB%84)	 | 关键问题是找起点，找终点。起点：若 sum+array[i] < array[i],即 sum<0, 前面的sum 加了不如不加，弃之，从i开始；终点：不好找，不用找。遍历一遍，把 sumMax 往上顶即可。 | 
| 6 | [把数组排成最小的数(45)](https://www.nowcoder.com/profile/1923750/note/detail/317008?tags=%E6%95%B0%E7%BB%84)	 | 重新定义Comparator排序规则（或者组合排序，例如商品的性质A和性质B同时决定商品的价格）。 变相的排序问题，1. 制定排序规则 2.排序（本题目用的是collection的匿名内部类来实现，算法复杂度 O(Nlog(N)) ）；需要注意：大数问题，用string解决。 | 
| 7 | [数组中的逆序对(剑指51)](https://www.nowcoder.com/profile/1923750/note/detail/317038?tags=%E6%95%B0%E7%BB%84)	 |消除逆序对，归并排序变化,算法复杂度为O(Nlog(N))。排序问题本质是消除逆序对，只通过交换相邻元素来消除逆序对的做法，算法复杂度为 O(n2);| 
| 8 | [数字在排序数组中出现的次数（剑指53）](https://www.nowcoder.com/profile/1923750/note/detail/322863?tags=%E6%95%B0%E7%BB%84)	 | 凡是排序数组，想到二分查找 | 
| 9| [数组中只出现一次的数字(剑指56)](https://www.nowcoder.com/profile/1923750/note/detail/323118?tags=%E6%95%B0%E7%BB%84) |	经典题目（1.只出现一次的1个数；2.只出现一次的两个数，其他数字出现偶数次）异或。 异或满足交换律和结合律。 异或的本质是不带进位的加法。 只出现一次的1个数，则异或全部数字； 只出现一次的2个数，则拆分成两个子数组。| 
| 10 | [和为S的两个数字（剑指57.1）](https://www.nowcoder.com/profile/1923750/note/detail/323206?tags=%E6%95%B0%E7%BB%84) |	经典题目（1.数组是递增排列；2.数组中可能有重复的数字）双指针夹逼| 
| 11 | [和为S的连续正数序列(剑指57.2)](https://www.nowcoder.com/profile/1923750/note/detail/323188?tags=%E6%95%B0%E7%BB%84)	 | 上面题目的变式，双指针滑动窗口| 
| 12	| [构建乘积数组（剑指66）](https://www.nowcoder.com/profile/1923750/note/detail/324691?tags=%E6%95%B0%E7%BB%84)	| 构建乘积但不允许使用除法。想到用两个数组来保存中间结果，如何初始化这两个数组，想到将通项公式转化为递推公式。|
|
