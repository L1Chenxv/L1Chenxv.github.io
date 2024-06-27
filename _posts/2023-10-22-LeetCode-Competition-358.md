---
layout: post
title: 第358场LeetCode周赛分析
categories: [LeetCode]
description: some word here
keywords: keyword1, keyword2
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---



# 正文

第358场LeetCode周赛总结

本次周赛题目由于自己对时间复杂度分析出错，导致可以ac的t3没能写出，因此把题目进行记录：

### 时间复杂度分析

[100097. 合法分组的最少组数](https://leetcode.cn/problems/minimum-number-of-groups-to-create-a-valid-assignment/)

```markdown
给你一个长度为 n 下标从 0 开始的整数数组 nums 。

我们想将下标进行分组，使得 [0, n - 1] 内所有下标 i 都 恰好 被分到其中一组。

如果以下条件成立，我们说这个分组方案是合法的：

对于每个组 g ，同一组内所有下标在 nums 中对应的数值都相等。
对于任意两个组 g1 和 g2 ，两个组中 下标数量 的 差值不超过 1 。
请你返回一个整数，表示得到一个合法分组方案的 最少 组数。
```

```
示例：
输入：nums = [10,10,10,3,1,1]
输出：4
解释：一个得到 2 个分组的方案如下，中括号内的数字都是下标：
组 1 -> [0]
组 2 -> [1,2]
组 3 -> [3]
组 4 -> [4,5]
分组方案满足题目要求的两个条件。
无法得到一个小于 4 组的答案。
所以答案为 4 。
```

题解：

[妙蛙种子大佬的时间负责度分析][https://leetcode.cn/problems/minimum-number-of-groups-to-create-a-valid-assignment/solutions/2493127/mei-ju-fen-lei-tao-lun-by-tsreaper-chb5/]

