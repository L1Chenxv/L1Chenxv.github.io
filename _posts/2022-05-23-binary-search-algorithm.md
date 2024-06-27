---
layout: post
title: 二分查找中的循环不变量
categories: 数据结构
description: 二分查找中的循环不变量
keywords: 数据结构
topmost: false
---

理解不同写法中二分查找的循环不变量是掌握二分查找的精髓

>  [二分查找 精讲](https://www.bilibili.com/video/BV1AP41137w7/?vd_source=8ba6bd7294c0aa28f29b845df8ff80ed)

### 基本概念
三种写法：

- [L, R] 
- (L, R)
- [L, R)

**循环不变量**：区间外皆符合一定条件

四种基本类型：

- ≥
- ＞
- ≤
- ＜

类型转换，以 **≥X**为base

 **＞X **=> **≥ X + 1**

**<X **=> **(≥X) - 1**

**≤X **=> **(＞X) - 1**



### 模板方法
// lowerBound 返回最小的满足 nums[i] >= target 的 i
// 如果数组为空，或者所有数都 < target，则返回 nums.length
// 要求 nums 是非递减的，即 nums[i] <= nums[i + 1]
target：**理解循环不变量，初始化边界，边界缩小写法**

#### 闭区间写法
```java
// 闭区间写法
private int lowerBound(int[] nums, int target) {
    int left = 0, right = nums.length - 1; // 闭区间 [left, right]
    while (left <= right) { // 区间不为空
        // 循环不变量：
        // nums[left-1] < target
        // nums[right+1] >= target
        int mid = left + (right - left) / 2;
        if (nums[mid] < target)
            left = mid + 1; // 范围缩小到 [mid+1, right]
        else
            right = mid - 1; // 范围缩小到 [left, mid-1]
    }
    return left; // 或者 right+1
}
```
#### 左开右闭区间写法
```java
// 左闭右开区间写法
private int lowerBound2(int[] nums, int target) {
    int left = 0, right = nums.length; // 左闭右开区间 [left, right)
    while (left < right) { // 区间不为空
        // 循环不变量：
        // nums[left-1] < target
        // nums[right] >= target
        int mid = left + (right - left) / 2;
        if (nums[mid] < target)
            left = mid + 1; // 范围缩小到 [mid+1, right)
        else
            right = mid; // 范围缩小到 [left, mid)
    }
    return left; // 或者 right
}
```
**开区间写法**
```java
// 开区间写法
private int lowerBound3(int[] nums, int target) {
    int left = -1, right = nums.length; // 开区间 (left, right)
    while (left + 1 < right) { // 区间不为空
        // 循环不变量：
        // nums[left] < target
        // nums[right] >= target
        int mid = left + (right - left) / 2;
        if (nums[mid] < target)
            left = mid; // 范围缩小到 (mid, right)
        else
            right = mid; // 范围缩小到 (left, mid)
    }
    return right; // 或者 left+1
}
```
