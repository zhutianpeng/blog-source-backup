---
title: java OOM异常
date: 2019-04-23  16:58:47
tags:
	- java
    - 动态规划
---

# 方法论
把DP看作是数列，有3个关键因素，2种求解思路，3种实现方式，3个难点

## 3个关键因素：
- 最优子结构：最终成功的条件的子集，通常有几种情况组成，或是求和，或是求max
- 边界 ：初始条件，不要遗漏
- 状态转移公式 ：递推公式

## 2种求解思路：
- top-down： 递归，从A(n),A(n-1)往下求值
- bottom-up： 循环，从A(1),A(2)…往上求值

## 3种实现的方式：
具体的实现方式有三种，时间复杂度要看具体情况而论（不一定数学的方式就最快），一般采用 “简单递归 + 备忘录算法” ：

- top-down
    - 简单递归 ： 时间复杂度是指数级，底数要看最优子结构有几项
    - 简单递归 + 备忘录算法 ： 时间复杂度和空间复杂度相同，为HashMap中的key的数量
- bottom-up
    - 循环求解 ： 时间复杂度为 O(N M …) 其中 N,M 均为维度上面的衡量指标

## 3个难点：
- 知道这个题目要用动态规划
- 最优子结构不好想
- 边界条件容易 遗漏情况

# 例题集
## 1. 跳台阶：
[一只青蛙一次可以跳上1级台阶，也可以跳上2级。求该青蛙跳上一个n级的台阶总共有多少种跳法（先后次序不同算不同的结果）。](https://www.nowcoder.com/profile/1923750/note/detail/307633?noteType=0)

简单递归： 时间复杂度O（2^N）
```java
//动态规划
//求递推公式是最终目标（递归），思考方式有两种：bottom up, top down.
//bottom up：走上第三级台阶的种数等于走上第一级台阶+第二级台阶 f(3)=f(2)+f(1);
//top down: 走完全部台阶种数= 走完倒数第一级+走完倒数第二级:  f(n)=f(n-1)+f(n-2); 
public class Solution {
    public int JumpFloor(int target) {
        if(target<=0){
            return 0;
        }
        if(target==1)
            return 1;
        if(target==2)
            return 2;
        return JumpFloor(target-1) + JumpFloor(target-2);
    }
}
```
简单递归+备忘录算法： 时间复杂度 O(n)， 空间复杂度 O(n)
```java
public class Solution {
    public int JumpFloor(int target, HashMap<Integer,Integer> map) {
        if(target<=0){
            return 0;
        }
        if(target==1)
            return 1;
        if(target==2)
            return 2;
        if(map.contains(target)){
            return map.get(target);
        }else{
            int value = JumpFloor(target-1) + JumpFloor(target-2);
            map.put(target,value);
            return value;
        }
    }
}
```
数学法: 时间复杂度O(n), 空间复杂度O(1)
```java
public class Solution {
    public int JumpFloor(int target) {
        if(target<=0) return 0;
        if(target==1) return 1;
        if(target==2)  return 2;

       int a=1; int b=2; int temp =0;
       for(int i=3;i<=n;i++){
           temp=a+b;
           a=b;
           b=temp;
       }
       return temp;
    }
}
```

## 2.变态跳台阶
[一只青蛙一次可以跳上1级台阶，也可以跳上2级……它也可以跳上n级。求该青蛙跳上一个n级的台阶总共有多少种跳法。](https://www.nowcoder.com/profile/1923750/note/detail/309821?noteType=0)

数学法：
```java
import java.lang.Math;
//思路：动态规划
//递推公式： f(n)= f(n-1)+ f(n-2)+f(n-3)+... +f(1)+f(0), 千万不要漏掉 f(0),以后还是建议用buttom up的思路
//递推公式2：f(n)=2f(n-1)
//通项公式： f(n) = Math.pow(2,n-1);
public class Solution {
    public int JumpFloorII(int target) {
        if(target<1){
            return 0;
        }else if(target==1){
            return 1;
        }else{
            return (int)Math.pow(2,target-1);
        }
    }
}
```
## 3. 挖金矿
有一个国家发现了5座金矿，每座金矿的黄金储量不同，需要参与挖掘的工人数也不同。参与挖矿工人的总数是10人。每座金矿要么全挖，要么不挖，不能派出一半人挖取一半金矿。要求用程序求解出，要想得到尽可能多的黄金数目。

简单递归：时间复杂度 O(2^n)

```java 
//人数:w    金矿:n    G[]:金矿中金子数目     P[]:挖矿需要的人数
//简单递归, 时间复杂度O(2^n)
public static int goldDigger1(int w, int n, int[] G, int[] P){
    if(n<=1 && w<P[0]){
        return 0;
    }else if(n==1 && w>=P[0]){
        return G[0];
    }else if(n>1 && w<P[n-1]){
        return goldDigger1(w, n-1, G, P);
    }else{
        return Math.max(goldDigger1(w,n-1,G,P) , goldDigger1((w-P[n-1]),n-1,G,P)+G[n-1]);
    }
}
```

简单递归 + 备忘录算法： 时间复杂度 O(key), 空间复杂度 O(key)
```java
//人数:w    金矿:n    G[]:金矿中金子数目     P[]:挖矿需要的人数
//简单递归 + 备忘录算法, 时间复杂度 O(key), 空间复杂度 O(key)
public static int goldDigger2(int w, int n, int[] G, int[] P, HashMap<String,Integer> map){
    if(n<=1 && w<P[0]){
        return 0;
    }else if(n==1 && w>=P[0]){
        return G[0];
    }else if(map.containsKey(w+"_"+n)) {
        return map.get(w+"_"+n);
    }else{
        int value=0;
        if(n>1 && w<P[n-1]){
            value = goldDigger2(w, n-1, G, P,map);
        }else{
            value =  Math.max(goldDigger2(w,n-1,G,P,map) , goldDigger2((w-P[n-1]),n-1,G,Pmap)+G[n-1]);
        }
        map.put(w+"_"+n,value);
        return value;
    } 
}
```

## 4. Decode Ways
题目： [leetcode91](https://leetcode.com/problems/decode-ways/)

解答1. 数学公式法：（这种写起来比较清爽）
```java
public static int decodeWays(String s){
    if(s==null||s.length()==0){
        return 0;
    }
    int n=s.length();
    int[] dp = new int[n+1];
    //init
    dp[0]=1; //empty String only has 1 way to decode
    dp[1]=s.charAt(0)=='0'? 0:1; //0:cannot decode; not 0: 1 way
    for(int i=2;i<=n;i++){
        int first = Integer.valueOf(s.substring(i-1,i));
        int second = Integer.valueOf(s.substring(i-2,i));
        if(first>=1 && first<=9){dp[i]+=dp[i-1];}
        if(second>=10 && second<=26){dp[i]+=dp[i-2];}
    }
    return dp[n];
}
```

解答2. 简单递归+备忘录算法 (很直观，但是不建议写)
```java
public static int decodeWay(String s, int n, HashMap<Integer,Integer> map){
    if(s==null||s.length()==0){
        return 0;
    }
    if(map.containsKey(n)){
        return map.get(n);
    }else{
        int value=0;
        if(n==0){
            value = s.charAt(0)=='0'? 0:1;
        }else if(n==1){
            value =  s.charAt(0)=='0'? 0:1;
        }else{
            int first = Integer.valueOf(s.substring(n-1,n));
            int second = Integer.valueOf(s.substring(n-2,n));
            if(first>=1 && first<=9 && second>=10 && second<=26){
                value= decodeWay(s,n-1,map)+decodeWay(s,n-2,map);
            }else if(first==0 && second>=10 && second<=26){
                value= decodeWay(s,n-2,map);
            }else if(first>=1 && first<=9 && (second<10 || second>26)){
                value= decodeWay(s,n-1,map);
            }else{
                value= 0;
            }
        }
        map.put(n,value);
        return value;
    }
}
```