---
title: Java 排序方法总结
tags: []
categories: []
toc: false
date: 2019-07-25 16:42:11
---

![](/images/sort-algorithms.png)

> 经常项目中会遇到排序问题面试也会问，收集下各种排序思想和代码实现以便加深记忆


![](/images/sortTable.png)


### 冒泡排序（Bubble Sort）
访问数组的所有元素每次比较两个相邻元素如果符合条件则置换位置一直到所有相邻元素满足条件
![图解](/images/bubble-sort.gif)

###### 代码实现
``` java
public static void bubble_sort(int[] sortArray) {
        for (int i = 0; i < sortArray.length; i++) {
     	 /**
         	    * 内循环控制需要比较(length-i-1)次 
         	    * 每次循环会把最大值置换到最右边 
          	    */
            for (int j = 0; j < sortArray.length - i - 1; j++) {
                if (sortArray[j + 1] < sortArray[j]) {
                    int temp = sortArray[j + 1];
                    sortArray[j + 1] = sortArray[j];
                    sortArray[j] = temp;
                }
            }
        }
    }
```

### 选择排序（Selection Sort）
从无序序列中寻找最小值置换到无序序列起始位置即有序序列最末段，循环`n-1`次使其所有元素排序完成
![图解](/images/selection-sort.gif)

##### 代码实现
``` java
public static void selection_sort(int[] sortArray){
    /**
         * 外循环控制无序列区的范围 从0  到 length
         */
        for(int i=0;i<sortArray.length;i++){
            int minIndex=i;
      /**
             * 内循环查找无序列区最小值下标 无序列区首位置为i
             */
            for(int j=i;j<sortArray.length;j++){
                if(sortArray[j]<sortArray[minIndex]){
                    minIndex=j;
                }
            }
      /**
             * 将最小值置换到无序列区首位置
             */
            int temp=sortArray[minIndex];
            sortArray[minIndex]=sortArray[i];
            sortArray[i]=temp;
        }
    }
```

### 插入排序 （Insertion Sort）
将数组中所有的元素依次跟之前的元素相比较，如遇到比自身小的元素则进行交换直到比较所有元素

![图解](/images/insert-sort.gif)

###### 代码实现
``` java
public static void insert_sort(int[] sortArray){
    	/**
         	* 外循环取出移动元素
         	*/
        for (int i = 1; i < sortArray.length-1; i++) {
            int comparativeValue = sortArray[i+1];
      	  /**
             	    * 内循环比较值前所有元素进行对比
             	   * 小于则进行位置后移直到所有元素比较完毕或遇到大于等于的元素进去插入
             	  */
            int preIndex  = i;
            while (preIndex  >= 0 && sortArray[preIndex] > comparativeValue) {
                sortArray[preIndex+1] = sortArray[preIndex];
                preIndex --;
            }
            sortArray[preIndex+1] = comparativeValue;
        }
    }
```

### 希尔排序（Shell Sort）

希尔排序又称__递减增量排序算法__属于插入排序改进版，算法思想为讲整个待排序记录分割为若干个子序列进行直接插入排序，然后再对所有子序列进行直接插入排序



### 快速排序