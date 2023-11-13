---
layout: post
title: Kruskal 算法
date: 2023-10-27
tag: [algorithm]
categories: Algorithm
---

Kruskal 算法是用来寻找最小树的算法，是由美国数学家 [Joseph Bernard Kruskal, Jr.](https://en.wikipedia.org/wiki/Joseph_Kruskal) 在 1956 年发表的。

## 步骤

1. 新建图\\(G\\)，\\(G\\)中拥有原图中的相同的顶点，但是没有边
2. 将原图中的所有边按权值从小到大排序
3. 从权值最小的边开始，如果这条边连接的两个顶点在图\\(G\\)中不在同一个连通分量中，则添加这条边到图\\(G\\)中
4. 重复步骤 3 ，直到图\\(G\\)中所有的顶点都在同一个连通分量中

## 证明

1. 第 3 步的只有不包含同一个连通分量的顶点的边才会被加入到新的图中，保证了新的图最终是一个树
2. 用反证法证明，如下：

- 存在一条边\\(e\\)不在新图\\(G\\)中，如果把这条边加入到图\\(G\\)，则会生成一个回路\\(P\\)。
- 假设在\\(P\\)中存在一个边\\(e_1\\)（这个边应该在新图\\(P\\)中），使得\\(w(e_1)>w(e)\\)（不是最小生成树）。
- 那么在第 3 步的时候\\(e\\)应该先于\\(e_1\\)进行判断。在之后判断\\(e_1\\)的时候，由于生成回路，\\(e_1\\)就不会家到新的图\\(G\\)中。这与\\(e_1\\)在新图\\(G\\)中矛盾。
- 所以存在\\(w(e_1)>w(e)\\)不成立。

## 复杂度

平均时间复杂度是：\\(O(\lvert{E}\rvert log\lvert{V}\rvert)\\)，或者：\\(O(\lvert{E}\rvert\alpha(\lvert{V}\rvert))\\)。

空间复杂度是：\\(O(\lvert{E}\rvert+\lvert{V}\rvert)\\)

## 应用

### leetcode 1584

[leetcode 1584 Min Cost to Connect All Points](https://leetcode.com/problems/min-cost-to-connect-all-points/) 可以使用寻找最小树的方法来解

```c++
class Solution {
public:
    int minCostConnectPoints(vector<vector<int>>& points) {
        if (points.size() <= 1) return 0;
        int totalMinCost = 0;

        // calcuate all edges and sort them by weight
        vector<Edge> edges;
        for (int i = 0; i < points.size(); i++) {
            for (int j = i + 1; j < points.size(); j++) {
                Edge edge(calcWeight(points[i], points[j]), i, j);
                edges.push_back(edge);
            }
        }
        sort(edges.begin(), edges.end());

        fa.resize(points.size());
        iota(fa.begin(), fa.end(), 0);
        sz.insert(sz.begin(), points.size(), 1);

        for (auto &[w, u, v] : edges) {
            if (merge(u, v)) totalMinCost += w;
        }

        return totalMinCost;
    }

private:
    // Edge: <weight, u, v>
    typedef tuple<int, int, int> Edge;
    vector<int> fa, sz;

    int calcWeight(vector<int> start, vector<int> end) {
        return abs(start[0] - end[0]) + abs(start[1] - end[1]);
    }

    int find(int x) {
        while (x != fa[x])
            x = fa[x] = fa[fa[x]];
        return x;
    }

    bool merge(int x, int y) {
        x = find(x), y = find(y);
        if (x == y) return false;
        if (sz[x] > sz[y]) std::swap(x, y);
        fa[x] = y;
        sz[y] += sz[x];
        return true;
    }
};
```

## 参考

- [1] [Wikipedia: 克鲁斯克尔演算法](https://zh.wikipedia.org/wiki/%E5%85%8B%E9%B2%81%E6%96%AF%E5%85%8B%E5%B0%94%E6%BC%94%E7%AE%97%E6%B3%95#)
