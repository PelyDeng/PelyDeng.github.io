title: 数组 - 两个数组的交集 II
author: Peilin Deng
tags:
  - 数组
categories:
  - 算法小抄
date: 2022-01-14 10:58:00
---
# 两个数组的交集 II
给你两个整数数组 nums1 和 nums2 ，请你以数组形式返回两数组的交集。返回结果中每个元素出现的次数，应与元素在两个数组中都出现的次数一致（如果出现次数不一致，则考虑取较小值）。可以不考虑输出结果的顺序。

示例 1：

输入：nums1 = [1,2,2,1], nums2 = [2,2]
输出：[2,2]
示例 2:

输入：nums1 = [4,9,5], nums2 = [9,4,9,8,4]
输出：[4,9]
 

提示：

1 <= nums1.length, nums2.length <= 1000
0 <= nums1[i], nums2[i] <= 1000
 

进阶：

如果给定的数组已经排好序呢？你将如何优化你的算法？
如果 nums1 的大小比 nums2 小，哪种方法更优？
如果 nums2 的元素存储在磁盘上，内存是有限的，并且你不能一次加载所有的元素到内存中，你该怎么办？

```java
class Solution {
    public int[] intersect(int[] arr1, int[] arr2) {
        Arrays.sort(arr1);
        Arrays.sort(arr2);

        int arr1Len = arr1.length, arr2Len = arr2.length;
        int len = Math.min(arr1Len, arr2Len);
        int[] newInt = new int[len];
        int i = 0, j = 0, k = 0;
        while (i < arr1Len && j < arr2Len) {
            if (arr1[i] == arr2[j]) {
                newInt[k++] = arr1[i];
                i++;
                j++;
            } else if (arr1[i] < arr2[j]) {
                i++;
            } else {
                j++;
            }
        }
        return Arrays.copyOfRange(newInt, 0, k);
    }
}
```
