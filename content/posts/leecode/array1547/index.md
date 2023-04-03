---
title: "1547. 切棍子的最小成本"
date: 2023-04-03T14:22:45+08:00
draft: false
categories: [日常一扣]
tags: [数组篇]
card: false
weight: 0
---

## 题目

https://leetcode.cn/problems/minimum-cost-to-cut-a-stick/

## 分析

1. 根据题目给出棍子的**长度**和**切割点**
2. 得出最小成本，即当前木棒的长度+切点左边的木棒切割完毕的最小成本+切点右边的木棒切割完毕的最小成本

如何得出最小成本是这道题的关键点。这里可以通过<u>分治递归的方法</u>将整体的最小成本细化为一个个子集的最小成本，然后通过<u>记忆化</u>将最小成本子集记录下来，再得出整体的最小成本。

## 实现

方法一：

```java
//记忆化

Map<Long, Integer> memo = new HashMap<>();

public int minCost(int n, int[] cuts) {
    //递归方法
    return  dfs(0, n, cuts);
}

private int dfs(int l, int r, int[] cuts) {
    //防止重复
    Long x = l * 1000000000L + r;
    //获取map中是否已经获得子集最小成本
    if (memo.containsKey(x)) {
        return memo.get(x);
    }
    //结果值
    int res = Integer.MAX_VALUE;
    //切割一个点的成本
    int cost = r - l;
    // 这里太过于暴力，但是也可以过
    for (int cut : cuts) {
        // cut是分割点
        if (cut <= l || cut >= r) {
            continue;
        }
        int a = dfs(l, cut, cuts);
        int b = dfs(cut, r, cuts);
        //获取最小成本结果子集
        res = Math.min(a + b, res);
    }
    //若当前子集 无法进一步切割，则为0，否则为结果子集＋当前结果集成本
    int ans = res == Integer.MAX_VALUE ? 0 : res + cost;
    //记忆当前子集的最小成本
    memo.put(x, ans);
    return ans;
}
```

方法二：

。。。待续









