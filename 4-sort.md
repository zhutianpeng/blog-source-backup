---
title: java实现的排序算法
date: 2019-03-17 16:58:47
tags:
	- java
    - sort
---

```java
package sort;

/**
 * Created by AndrewKing on 3/7/2019.
 */
public class Sort {
    public static void main(String[] args) {
        int[] nums = {2, 3, 1, 5, 4, 7, 9, 6, 8};
        quick_sort(nums,0,nums.length-1);
        for (int i = 0; i < nums.length; i++) {
            System.out.println(nums[i]);
        }

    }

    //选择排序：从第i个元素后面的所有元素中找到最大的，与i交换位置。
    //算法复杂度：O(n2)，稳定
    public static void selection(int[] nums) {
        int N = nums.length;
        for (int i = 0; i < N - 1; i++) {
            int min = i;
            for (int j = i + 1; j < N; j++) {
                if (nums[j] < nums[min]) {
                    min = j;
                }
            }
            int temp = nums[i];
            nums[i] = nums[min];
            nums[min] = temp;
        }
    }

    //冒泡排序：扫一次数组，两两交换相邻元素，把最大的元素移到了最右边；数组长度减1后，继续该操作。
    //算法复杂度：O(n2)，稳定
    public static void bubble(int[] nums) {
        int N = nums.length;
        for (int i = 0; i < N; i++) { //统计N次
            for (int j = 0; j < N - i - 1; j++) {
                if (nums[j] > nums[j + 1]) {
                    int temp = nums[j];
                    nums[j] = nums[j + 1];
                    nums[j + 1] = temp;
                }
            }
        }
    }

    //插入排序：i:每次找到一个元素，归位到左边他的位置;  i++
    public static void insert(int[] nums) {
        int N = nums.length;
        for (int i = 1; i < N; i++) {
            for (int j = i; j > 0 && nums[j] < nums[j - 1]; j--) {
                int temp = nums[j];
                nums[j] = nums[j - 1];
                nums[j - 1] = temp;
            }
        }
    }

    //希尔排序：特殊的插入排序, 有可能一次性消除多个逆序对，提高效率
    public static void shell(int nums[]){
        int N = nums.length;
        int h =1;
        while(h<N/3){
            h = h*3+1;
        }
        while(h>=1){
            for(int i=h;i<N;i++){
                for(int j=i;j>=h && nums[j]<nums[j-h];j=j-h){
                    int temp = nums[j];
                    nums[j] = nums[j - h];
                    nums[j - h] = temp;
                }
            }
            h=h/3;
        }
    }

    //归并排序

    public static void m_merge(int nums[],int b, int mid, int e){
        int[] aux = new int[nums.length]; //这里可以只申请一次
        int i=b, j=mid+1; //双指针
        for(int k=b;k<=e;k++){ //辅助空间
            aux[k] = nums[k];
        }
        for(int k=b;k<=e;k++){
            if(i>mid){
                nums[k]= aux[j];
                j++;
            }else if(j>e){
                nums[k] =aux[i];
                i++;
            }else if(aux[i]>=aux[j]){
                nums[k] = aux[j];
                j++;
            }else{
                nums[k] = aux[i];
                i++;
            }
        }
    }
    //递归写法：top-down  m_sort先分， m_merge后合，复杂度: O(logN)
    public static void m_sort_topDown(int nums[], int b, int e){
        if(e<=1 || b>=e){
            return;
        }
        int mid = b+ (e-b)/2; // 不能写成  (b+e)/2 会有溢出危险。
        m_sort_topDown(nums, b,mid);
        m_sort_topDown(nums, mid+1, e);
        m_merge(nums,b,mid,e);
    }

    //循环写法：bottom-up,先分后合，复杂度: O(logN)
    public static void m_sort_bottomUp(int nums[]){
        int len = nums.length;
        for(int sz=1;sz<len;sz=sz*2){
            for(int b=0;b<len-sz;b=b+2*sz){
                m_merge(nums,b,b+sz-1,Math.min(b+2*sz-1,len-1));
            }
        }
    }

    //快速排序
    private static void quick_sort(int[] nums, int l, int h) {
        if (h <= l) {
            return;
        }
        int j = partition2(nums, l, h);
        quick_sort(nums, l, j - 1);
        quick_sort(nums, j + 1, h);
    }
    //自己写的比较麻烦
    private static int partition(int[] nums, int l, int h) {
        int i=l+1, j=h;
        for(;i<=j;){
            if(nums[i]<nums[l]){
                i++;continue;
            }
            if(nums[j]>nums[l]){
                j--;continue;
            }
            int temp = nums[i];
            nums[i]=nums[j];
            nums[j] = temp;
            i++;j--;
        }
        if(nums[j]<nums[l]){
            int temp = nums[l];
            nums[l]=nums[j];
            nums[j] = temp;
        }else{
            j=l;
        }
        return j;
    }
    //比较简洁
    private static int partition2(int[] nums, int l, int h) {
        int i=l, j=h+1;
        while(true){
            while(nums[++i]<nums[l] && i!=h){ }
            while(nums[--j]>nums[l] && j!=l){ }
            if(i>=j){
                break;
            }
            int temp = nums[i];
            nums[i]= nums[j];
            nums[j] = temp;
        }
        int temp = nums[j];
        nums[j]= nums[l];
        nums[l] = temp;
        return j;
    }
}
```