---
layout: single
author_profile: true
comments: true
title: 1个快速排序等于5+个LeetCode题目
tagline: 
category: essay
tags : [LeetCode]
---

> 快速排序最坏情况运行时间为O(n^2)，虽然这个最坏情况运行时间比较差，但快速排序通常是用于排序的最佳的实用选择，这是因为其平均性能相当好：期望运行时间为O(nlgn)，而且O(nlgn)记号中隐含的常数因子很小。 ——《算法导论》

快速排序的主要思想是：分治。从待排序数组A中随机选择一个元素x，将数组A分为两个数组Aleft和Aright，Aleft中的元素全部<=x，Aright中的元素全部>x。然后对Aleft和Aright数组重复上述过程（递归）。

根据算法导论第7章写出如下程序

    void qsort(int *a, int lower, int upper){
      if(lower < upper){
        int mid = partition(a, lower, upper);
        qsort(a, lower, mid-1);
        qsort(a, mid+1, upper);
      }
    }
    
    int partition(int *a, int lower, int upper){
      int x = *(a+upper);// or call random()
      int i = lower - 1;
      for(int j = lower; j<upper; j++){
        if(a[j] <= x){
          swap(a[++i], a[j]);
        }
      }
      swap(a[++i], a[upper]);
      return i;
    }

上述算法描述中partition的实现方式值得学习，通过两个指针i,j，将数组元素分为3部分，

- a[lower, i]是 <=x 的值
- a[i,j)是 >x 的值
- a[j, upper)是未划分的值

在partition函数中，如果每次选取的 x 不能有效的划分数组，那么时间复杂度就会接近O(n^2)，
极端情况下，x 是a[lower, upper]中的最大值或者最小值，此时划分的结果是1：upper-lower，最不平衡的划分。  

如果面试官问你：快排什么情形下出现最坏时间复杂度？  
回答：数组有序。这是不准确的，因为partition可能是通过random随机取的一个元素，而非上述代码中的最后一个。  
更好的回答：划分函数极度不平衡的时候，qsort时间复杂度是O(n^2)。  


###LeetCode上应用partition的题目有

- [Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/)

查找第k大的元素，因为划分，分成这样的两段：前一段小于等于一个元素，而后一段大于该元素，当然可以确定这两段的个数，这样就知道第k大的数值分布在前一段还是后一段。

- [Sort Colors]( https://leetcode.com/problems/sort-colors/)

对0,1,2进行排序，可通过partition方式完成O(n)解法，先用2划分成left, right两个数组，再用1对left数组进行划分。

- [Remove Element](https://leetcode.com/problems/remove-element/)

- [Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array/)

- [Remove Duplicates from Sorted Array II](https://leetcode.com/problems/remove-duplicates-from-sorted-array-ii/)


