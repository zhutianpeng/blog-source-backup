---
title: 动态规划
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


`从前往后用循环，从后往前用递归。`


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

参见： [程序员小灰漫画：什么是动态规划？（整合版）](https://mp.weixin.qq.com/s/3h9iqU4rdH3EIy5m6AzXsg)

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

解答1. 数学公式法：（这种写起来比较清爽，但是不太容易想到）
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

解答3： 递归写法( 这种写法比较直观，但是会有超时的风险)
```java
 public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String s = sc.next();
        System.out.print(find(s,s.length()));
    }

    public static int find(String s, int i){
        if(i==0){
            return 1;
        }
        if(i==1){
            return s.charAt(0)=='0' ? 0 : 1;
        }

        int first = Integer.valueOf(s.substring(i-1,i));
        int twins = Integer.valueOf(s.substring(i-2,i));
        Boolean flag1=false, flag2=false;
        if(first>=1 && first<=9){
            flag1=true;
        }
        if(twins>=10 && twins<=26){
            flag2=true;
        }
        if(flag1&&flag2){
            return find(s,i-1)+find(s,i-2);
        }else if(flag1){
            return find(s,i-1);
        }else if(flag2){
            return find(s,i-2);
        }
        return 0;
    }
```


## 5.1. Jump Game  
题目： [LeetCode: 55.Jump Game](https://leetcode.com/problems/jump-game/)
```java
//探索法
class Solution {
    public boolean canJump(int[] nums) {
        int reach=0;
        for(int i=0;i<=reach;i++){
            if(reach<=i+nums[i])    reach = i+nums[i];
            if(reach>=nums.length-1)    return true;
        }
        return false;
    }
}
```

## 5.2 Jump Game2
题目 [LeetCode: 45.Jump Game II](https://leetcode.com/problems/jump-game-ii/)
```java
//探索法，O(n)，每一次都要走最远才算一步
class Solution {
    public int jump(int[] nums) {
        if(nums.length==1)  return 0;
        int reach=0, nextreach=0, step=0;
        int i=0;
        while(i<=reach){
            for(int j=i; j<=reach;j++){
                if(nextreach<=j+nums[j]){
                    nextreach=j+nums[j];
                }
                if(nextreach>=nums.length-1){
                    return step+1;
                }
            }
            i=reach+1;
            reach=nextreach;
            step++;
        }
        return -1;
    }
}
```

## 6. Minimum Path Sum
题目 [LeetCode: 64. Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum/)

```java
// 动态规划，由于只能向下走或者向右走，所以每一个格子只能由上一个格子或者左一个格子得到，取较小即可。
class Solution {
    public int minPathSum(int[][] grid) {
        for(int i=1;i<grid[0].length;i++){
            grid[0][i]=grid[0][i-1]+grid[0][i]; //边界
        }
        for(int i=1;i<grid.length;i++){
            grid[i][0]=grid[i-1][0]+grid[i][0]; //边界
        }
        for(int i=1;i<grid.length;i++){
            for(int j=1;j<grid[0].length;j++){
                grid[i][j]=grid[i][j]+ Math.min(grid[i-1][j],grid[i][j-1]); //最优子结构
            }
        }
        return grid[grid.length-1][grid[0].length-1];
    }
}
```

## 7. Triangle
题目 [LeetCode: 120. Triangle](https://leetcode.com/problems/triangle/)
```java
//这个题目是典型的DP
// 最优子结构： dp[i][j] = min{dp[i+1][j],dp[i+1][j+1]}+triangle.get[i].get[j]; 其中dp[i][j]的含义是从[i][j]位置 到最底层的路径和。
// 注意边界条件涵盖在 循环当中了。

class Solution {
        public int minimumTotal(List<List<Integer>> triangle) {
        int m=triangle.size();
        int n=triangle.get(triangle.size()-1).size();
        int[][] dp = new int[m+1][n+1];
        for(int i=m-1;i>=0;i--){
            for(int j=0;j<triangle.get(i).size();j++){
                dp[i][j]= Math.min(dp[i+1][j],dp[i+1][j+1])+triangle.get(i).get(j);
            }
        }
        return dp[0][0];
    }
}
```