---
title: Java 排序方法总结
tags: []
categories: []
toc: false
date: 2019-07-25 16:42:11
---

![](/images/sort-algorithms.png)

> 经常项目中会遇到排序问题面试也会问，收集下各种排序思想和代码实现以便加深记忆



### 插入排序 （Insertion Sort）
将数组中所有的元素依次跟之前的元素相比较，如遇到比自身小的元素则进行交换直到比较所有元素

![图解](/images/insert-sort.gif)

#### 代码实现
``` java
public static void main(String[] args) {
        int[] insert_sort = new int[]{55, 2, 23, 98, 35};
        System.out.println("未排序" + Arrays.toString(insert_sort));
        /**
         * 外循环取出移动元素
         */
        for (int i = 1; i < insert_sort.length; i++) {
            int comparativeValue = insert_sort[i];
            /**
             * 内循环比较值前所有元素进行对比
             * 小于则进行位置后移直到所有元素比较完毕或遇到大于等于的元素进去插入
             */
            int j = i;
            while (j > 0 && insert_sort[j - 1] > comparativeValue) {
                insert_sort[j] = insert_sort[j - 1];
                j--;
            }
            insert_sort[j] = comparativeValue;
            System.out.println("第" + i + "次移动元素" + comparativeValue + "后" + Arrays.toString(insert_sort));
        }
    }
```

|平均时间复杂度|最好情况|最坏情况|空间复杂度|
|-|-|-|-|
|$O(n^2)$|$O(n)$|$O(n^2)$|O(1)|

### 希尔排序（Shell Sort）

希尔排序又称__递减增量排序算法__属于插入排序改进版，算法思想为讲整个待排序记录分割为若干个子序列进行直接插入排序，然后再对所有子序列进行直接插入排序



### 冒泡排序

### 快速排序