---
title: 数据结构扩展篇一之Binary Lifting
published: true
layout: post
author: 陈家辉
tags:
- 数据结构
- 算法
- 基础
- 古城算法
---

# 参考

[古城算法ppt](https://docs.google.com/presentation/d/1F0gjgdt4f5IQAOpMY_YDm0amnxdbGAk6fUUO9rUwCww/edit#slide=id.p)
[倍增](https://oi-wiki.org/basic/binary-lifting/)

# 是什么

倍增法（英语：binary lifting），顾名思义就是翻倍。它能够使线性的处理转化为对数级的处理，大大地优化时间复杂度。

倍增法可以以$O(logN)$时间复杂度查找树节点的第K个祖先。也可以用于解决两节点最近公共祖先问题(LCA)。也能以对数时间复杂度计算树两节点的最大值，最小值以及和。该技术需要使用动态规划以 $O(N log N)$ 的方式预处理树

```c
int halfJumpParent = dp[i][j-1]
dp[i][j] = dp[halfJumpParent][j-1]
// i is the current node
// j is the (2^j)th parent
  
```



![img](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/Gp1SFnMjVoULmwvEW-mtMuCrpDtYzAx_xiNngNR_L9qOwii3naUMowEh8pxSi7GAWkbZMVD-7PIOb0KvZAQNJdhqj5rOzIoAFZ9aoPhxV68w13VNU9uVthFvRod7ReSS_nSws80JPm-D7jIH4OztokYe=s2048.png)

例子：

![img](https://lh4.googleusercontent.com/J_yFVeBHH_h0xhB0Rcc0Hh99eSGPv7AWx_nrkKOY6ljWknvP6R44PM_7GTLBT5BXvO5aNit4fAHYgRiEL0tZ2IIEdOdxoPLTJwQToqcRECVxP4iZZa_6Ub4D49SemlvjL33x9NhiGZNB4_afrNLfa13g=s2048)

![img](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/F61yoqjvrSmWa7TuJHTK79hA3rAyJWrGXjbDxFtr-DNvYTpn01hfGYJBMWSGhg_34cqmOoWkPjlbgvvg40OpCLCbOa25Yu1O9PmjeYE8CaSeuMJKqAnxI4IUYZ4lTepvZm6nuFMTQQPRnB5K5vDKQGzZ=s2048.png)

(a,b)  $2^a th$ parent node is b

```python
	Precalculate():
    dp[i][j] = -1 for each pair i and j
for i = 1 to n
dp[i][0] = parent[i]

for h = 1 to logn
for i =1 to n
If dp[i][h-1] not equal to -1
dp[i][h] = dp[dp[i][h-1]][h-1]

Findkthancestor():
  for i = logn to 0
  	If dp[currentNode][i] != -1 and 2^i <=k:
      currentNode = dp[currentNode][2^i]
      k = k - 2^i
  return currentNode
```

### [树节点的第 K 个祖先](https://leetcode.cn/problems/kth-ancestor-of-a-tree-node/)

![image-20230718113051980](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230718113051980.png)

```java
class TreeAncestor {
    Integer[][] dp;
    public TreeAncestor(int n, int[] parent) {
        dp = new Integer[n][20];
        for(int i=0;i<parent.length;i++){
            dp[i][0] = parent[i];
        }
        for(int j=1;;j++){
            boolean flag = true;
            for(int i=0;i<n;i++){
                dp[i][j] = dp[i][j-1] == -1? -1 : dp[dp[i][j-1]][j-1];
                if(dp[i][j]!=-1)  flag = false;
            }
            if(flag) break;
        }
    }
    
    public int getKthAncestor(int node, int k) {
        if(node==-1 || k==0) return node;
        for(int i=0;k>0;i++){
            if(k%2==1){
                if(dp[node][i]==null || dp[node][i]==-1) return -1;
                node = dp[node][i];
            }
            k /= 2;
        }
        return node;
    }
}
```

**初始化**

| 节点 | (a,b) |       |
| ---- | ----- | ----- |
| 0    | -1    | null  |
| 1    | (0,0) | null  |
| 2    | (0,0) | Null  |
| 3    | (0,1) | (1,0) |
| 4    | (0,1) | (1,0) |
| 5    | (0,2) | (1,0) |
| 6    | (0,2) | (1,0) |

**查询**


$$
查询 node=6，k=2的节点 \\

dp[6][1] = 0
$$


# 总结

预处理时间复杂度 $O(nlogn)$，查询时间复杂度$O(logn)$，空间复杂度$O(nlogn)$

也可以用来解决LCA问题，但是相对来说不常见，掌握基础dfs也能做。
