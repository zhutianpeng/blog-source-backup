---
title: 牛客网 > leetcode > 二叉树算法 题目总结
date: 2019-01-08 16:58:47
tags:
	- 牛客网
    - 二叉树
---

牛客网上 [leetCode](https://www.nowcoder.com/ta/leetcode) 在线练习部分 全部二叉树算法汇总，索引到我的笔记

## 数据结构
![数据结构](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20190830161228.png)


## 二叉树遍历：
```java
public class Traverse {

    //前序
    public void preorder_traverse(TreeNode root) {
        System.out.println(root.val);
        if(root.left!=null) preorder_traverse(root.left);
        if(root.right!=null)preorder_traverse(root.right);
    }
    //中序
    public static void inorder_traverse(TreeNode root) {
        if(root==null) return;
        inorder_traverse(root.left);
        System.out.println(root.val);
        inorder_traverse(root.right);
    }
    //后序
    public static void poster_traverse(TreeNode root) {
        if(root==null) return;
        poster_traverse(root.left);
        poster_traverse(root.right);
        System.out.println(root.val);
    }

//    前序遍历，存回溯的第一个节点（右节点）
    public static void preorder_traverse2(TreeNode root){
        if(root==null) return;
        Stack<TreeNode> stack = new Stack<TreeNode>();
        while(!stack.empty() || root!=null){
            if(root!=null){
                System.out.println(root.val);
                if(root.right!=null) stack.push(root.right);
                root=root.left;
            }else{
                root=stack.pop();
            }
        }
    }

//    中序遍历，存回溯的第一个节点（根节点）
    public static void inorder_traverse2(TreeNode root){
        if(root==null) return;
        Stack<TreeNode> stack = new Stack<TreeNode>();
        while(!stack.empty() || root!=null){
            if(root!=null){
                stack.push(root);
                root=root.left;
            }else{
                root = stack.pop();
                System.out.println(root.val);
                root = root.right;
            }
        }
    }

//    后序遍历，按照先序遍历：根+右+左，然后再用stack倒序
    public static void poster_traverse2(TreeNode root){
        if(root==null) return;
        Stack<TreeNode> stack = new Stack<TreeNode>();
        Stack<Integer> result = new Stack<Integer>();
        while(!stack.empty() || root!=null){
            if(root!=null){
                result.push(root.val);
                if(root.left!=null) stack.push(root.left);
                root=root.right;
            }else{
                root = stack.pop();
            }
        }
        while(!result.empty()){
            System.out.println(result.pop());
        }
    }

    //    层序遍历，用queue实现
    public static void layer_traverse(TreeNode root){
        if(root==null) return;
        Queue<TreeNode> queue = new LinkedList<TreeNode>();
        queue.offer(root);
        while(!queue.isEmpty()){
            int qlength = queue.size(); //关键，只记录每一行个数
            for(int i=0;i<qlength;i++){
                TreeNode t = queue.poll();
                System.out.println(t.val);
                if(t.left!=null) queue.offer(t.left);
                if(t.right!=null) queue.offer(t.right);
            }
        }
    }
}
```


## 二叉树题目：

| | 题目 | 考点 |
|:-----------:| :-------------:|:-------------:|
| 1 |  [Minimum Depth of Binary Tree](https://www.nowcoder.com/profile/1923750/note/detail/271328?tags=tree)  | 最小深度：层序遍历，循环 | 
| 2 | [binary-tree-postorder-traversal](https://www.nowcoder.com/profile/1923750/note/detail/271327?tags=tree)	 | 后序遍历：递归 | 
| 3 | [binary-tree-preorder-traversal](https://www.nowcoder.com/profile/1923750/note/detail/271326?tags=tree) | 先序遍历：递归 | 
| 4 | [sum-root-to-leaf-numbers](www.nowcoder.com/profile/1923750/note/detail/271329?tags=tree)	 | 遍历，数学，递归 | 
| 5 | [binary-tree-maximum-path-sum](https://www.nowcoder.com/profile/1923750/note/detail/271325?tags=tree)	 | 先序遍历，循环，栈 | 
| 6 | [populating-next-right-pointers-in-each-node-ii](https://www.nowcoder.com/profile/1923750/note/detail/271324?noteType=0)	 | 层序遍历，循环，队列 | 
| 7 | [populating-next-right-pointers-in-each-node](https://www.nowcoder.com/profile/1923750/note/detail/271324?noteType=0)	 | 层序遍历，循环，队列 | 
| 8 | [path-sum-ii](https://www.nowcoder.com/profile/1923750/note/detail/271394?noteType=0)	 | 递归，数学计算 | 
| 9| [path-sum](https://www.nowcoder.com/profile/1923750/note/detail/271394?noteType=0) |	递归，数学计算 | 
| 10 | [balanced-binary-tree](https://www.nowcoder.com/profile/1923750/note/detail/271543?noteType=0) |	递归，后序遍历 | 
| 11 | [binary-tree-level-order-traversal-ii](https://www.nowcoder.com/profile/1923750/note/detail/272708?noteType=0)	 | 从下往上的层序遍历，循环，栈，ArrayList逆序 | 
| 12	| [construct-binary-tree-from-inorder-and-postorder-traversal](https://www.nowcoder.com/profile/1923750/note/detail/272823?noteType=0 )	| 中序遍历+后序遍历 构造二叉树：后序 list找根，中序List划分。 递归。 | 
| 13	| [construct-binary-tree-from-preorder-and-inorder-traversal](https://www.nowcoder.com/profile/1923750/note/detail/272786?noteType=0 )	| 先序遍历+后序遍历 构造二叉树：前序list找根，中序List划分。 递归。 | 
| 14	| [maximum-depth-of-binary-tree]( https://www.nowcoder.com/profile/1923750/note/detail/272921?noteType=0)	| 最大深度 | 
| 15	| [binary-tree-zigzag-level-order-traversal](https://www.nowcoder.com/profile/1923750/note/detail/272982?noteType=0 )	|  层序遍历循环+ArrayList的逆序 | 
| 16	| [binary-tree-level-order-traversal](https://www.nowcoder.com/profile/1923750/note/detail/272984?noteType=0 )	| 层序遍历，循环 | 
| 17 | [symmetric-tree]( https://www.nowcoder.com/profile/1923750/note/detail/273198?noteType=0)	| 层序遍历（左右）左子树，层序遍历（右左）右子树，直接比较每一个元素 | 
| 18	| [same-tree](https://www.nowcoder.com/profile/1923750/note/detail/273233?noteType=0 ) | 	先序遍历，递归两棵树。同时比较相同节点的值 | 
| 19| 	[recover-binary-search-tree](https://www.nowcoder.com/profile/1923750/note/detail/273788?noteType=0 )	|  中性遍历，递归，延续 validate-binary-search-tree，注意交换节点的两种情况，相邻，不相邻 | 
| 20	| [validate-binary-search-tree](https://www.nowcoder.com/profile/1923750/note/detail/273289?noteType=0) | 	中序遍历，递归，法1：先变成List; 法2：直接比较 | 
| 21	| [unique-binary-search-trees-ii](https://www.nowcoder.com/profile/1923750/note/detail/274545?noteType=0) | 	动态规划，递归，延续上题，unique-binary-search-trees | 
| 22 | [unique-binary-search-trees]( https://www.nowcoder.com/profile/1923750/note/detail/274521?noteType=0)	| 动态规划，递归 | 
| 23 | [binary-tree-inorder-traversal](https://www.nowcoder.com/profile/1923750/note/detail/274346?noteType=0)	| 中序遍历，递归 | 
|
