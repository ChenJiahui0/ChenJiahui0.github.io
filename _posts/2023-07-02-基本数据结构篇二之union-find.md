---
title: 基本数据结构篇二之并查集
published: true
layout: post
author: 陈家辉
tags:
- 数据结构
- 算法
- 基础
- 古城算法
---

# 引用

[古城算法ppt](https://docs.google.com/presentation/d/1GCtsrnfljBVwm_ng0izVP0M8CGs0ZcOMVoPRIPrahfk/edit#slide=id.g967dfc86ae_0_59)

# 是什么

并查集 是一种树形的数据结构，用于处理不交集的合并(union)及查询(find)问题。

* Find：确定元素属于哪一个子集。它可以被用来确定两个元素是否属于同一子集。
* Union：将两个子集合并成同一个集合。

![image-20230702151642519](https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202307021516591.png)

## 模板

![img](https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202307021518980.png)

### 基础模板

```JAVA
public class DSU {
    int[] parent;
    public DSU(int n){
        parent = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }
    public int find(int x) {
        if(parent[x]!=x) parent[x] = find(parent[x]);
        return parent[x];
    }
    public void union(int x,int y){
        parent[find(x)] = find(y);
    }
}
```

### improved with size

![img](https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202307021543173.png)

```java
public class DSU {
    int[] parent;
    int[] size;

    public DSU2(int n) {
        parent = new int[n];
        size = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
        Arrays.fill(size, 1);
    }

    public int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }

    public void union(int x, int y) {
        int rootX = find(x), rootY = find(y);
        if (rootX == rootY) return;
        if (size[rootX] <= size[rootY]) {
            parent[rootX] = rootY;
            size[rootY] += size[rootX];
        }else{
            parent[rootY] = rootX;
            size[rootX] += size[rootY];
        }
    }
}
```

### improved with rank

让并查集树高度最矮

![img](https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202307021544547.png)

```java
public class DSU {
    int[] parent;
    int[] rank;

    public DSU3(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
        Arrays.fill(rank, 1);
    }

    public int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }

    public void union(int x, int y) {
        int rootX = find(x), rootY = find(y);
        if (rootX == rootY) return;
        if (rank[rootX] < rank[rootY]) {
            parent[rootX] = rootY;
            rank[rootY] += rank[rootX];
        }else if (rank[rootX] > rank[rootY]){
            parent[rootY] = rootX;
            rank[rootX] += rank[rootY];
        }else{
            parent[rootX] = rootY;
            rank[rootY]++;
        }
    }
}
```

# 习题

[547. 省份数量](https://leetcode.cn/problems/number-of-provinces/)

<img src="https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202307021548007.png" alt="image-20230702154821968" style="zoom:67%;" />

```JAVA
class Solution {
    public int findCircleNum(int[][] isConnected) {
        DSU dsu = new DSU(isConnected.length);
        for(int i=0;i<isConnected.length;i++){
            for(int j=0;j<isConnected[0].length;j++){
                if(isConnected[i][j]==1) dsu.union(i,j);
            }
        }
        int res=0;
        for(int i=0;i<isConnected.length;i++){
            if(dsu.find(i)==i) res++;
        }
        return res;
    }
}
public class DSU {
    int[] parent;
    public DSU(int n){
        parent = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }
    public int find(int x) {
        if(parent[x]!=x) parent[x] = find(parent[x]);
        return parent[x];
    }
    public void union(int x,int y){
        parent[find(x)] = find(y);
    }
}
```



![img](https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202307021603726.png)

```java
public class IslandNumber {
    public static void main(String[] args) {
        IslandNumber islandNumber = new IslandNumber();
        System.out.println(islandNumber.numsIsland(3, 3, new int[][]{{0, 0}, {0, 1}, {1, 2}, {2, 1}}));
    }
    int[][] dirs = {{1, 0}, {0, 1}, {-1, 0}, {0, -1}};
    public List<Integer> numsIsland(int m, int n, int[][] position){
        DSU dsu = new DSU(m * n);
        boolean[][] isIsland = new boolean[m][n];
        int count=0;
        List<Integer> res = new ArrayList<>();
        for (int[] cur : position) {
            if (isIsland[cur[0]][cur[1]]){
                res.add(count);
            }
            isIsland[cur[0]][cur[1]] = true;
            count++;
            for (int[] dir : dirs) {
                int x = cur[0]+dir[0];
                int y = cur[1]+dir[1];
                if(x<0||x>=m||y<0||y>=n||!isIsland[x][y]) continue;
                int component1 = dsu.find(x * m + y);
                int component2 = dsu.find(cur[0] * m + cur[1]);
                if (component1!= component2) {
                    dsu.union(component1,component2);
                    count--;
                }
            }
            res.add(count);
        }
        return res;
    }
    class DSU{
        int[] parent;
        public DSU(int size){
            parent = new int[size];
            for (int i = 0; i < parent.length; i++) {
                parent[i] = i;
            }
        }
        public void union(int x,int y){
            parent[find(x)] = find(y);
        }
        public int find(int x){
            if(parent[x]!=x) parent[x] = find(parent[x]);
            return parent[x];
        }
    }
}
```

[128. 最长连续序列](https://leetcode.cn/problems/longest-consecutive-sequence/)

<img src="https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230703223546767.png" alt="image-20230703223546767" style="zoom:67%;" />

```java
class Solution {
    public int longestConsecutive(int[] nums) {
        Map<Integer,Integer> map = new HashMap<>();
        DSU dsu =  new DSU(nums.length);
        for(int i=0;i<nums.length;i++){
            if(map.containsKey(nums[i])) continue;
            map.put(nums[i],i);
            if(map.containsKey(nums[i]+1)) dsu.union(map.get(nums[i]+1),i);
            if(map.containsKey(nums[i]-1)) dsu.union(map.get(nums[i]-1),i);
        }
        return dsu.findMax();
    }
}
class DSU{
    int[] size;
    int[] parent;
    public DSU(int n){
        size = new int[n];
        Arrays.fill(size,1);
        parent = new int[n];
        for(int i=0;i<n;i++){
            parent[i] = i;
        }
    }

    public int find(int x){
        if(parent[x]!=x) parent[x] = find(parent[x]);
        return parent[x];
    }

    public void union(int x,int y){
        int rootX = find(x);
        int rootY = find(y);
        if(rootX==rootY) return;
        if(size[rootX]<=size[rootY]){
            parent[rootX] = parent[rootY];
            size[rootY] += size[rootX];
        }else{
            parent[rootY] = parent[rootX];
            size[rootX] += size[rootY];
        }
    }

    public int findMax(){
        int res = 0;
        for(int s:size){
            res = Math.max(res,s);
        }
        return res;
    }
}
```

# 总结

![img](https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202307032323720.png)

并查集是一种树形的数据结构，用于处理不交集的合并及查询问题。

有三种优化形式：

1. Path compression
2. Union by size
3. Union by rank

