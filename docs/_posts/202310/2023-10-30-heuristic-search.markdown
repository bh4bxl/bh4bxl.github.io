---
layout: post
title: 启发式搜索
date: 2023-10-30
tag: [algorithm]
categories: Algorithm
---

启发式搜索是在普通搜索算法的基础上引入启发函数的搜索算法。

启发式函数的作用是基于已有的信息对搜索的每一个分支选择都做估价，进而选择分支。简单来说，启发式搜索就是对取和不取都做分析，从中选取更优解或删去无效解。

## 应用

### leetcode 1631

[leetcode 1631: Path With Minimum Effort](https://leetcode.com/problems/path-with-minimum-effort)

这个题目要求找出路径上的最小的Effort，和找到最短路径有些类似，但是又不能直接区找最短路径。如果想用最短路径的算法话，就可以定义一个启发函数：

$$
h(x)=max(e_0, e_1, ..., e_n)
$$

其中\\(e_0, e_1, ..., e_n\\)是路径上格子高度的差的绝对值，也就是相邻格子的Effort。

定义好启发函数后就可以用Dijkstra算法去搜索了。

代码如下（官方代码稍作修改）：

```c++
class Solution {
public:
    int minimumEffortPath(vector<vector<int>>& heights) {
        int m = heights.size();
        int n = heights[0].size();

        priority_queue<Cell, vector<Cell>> q;
        q.emplace(0, 0, 0);

        vector<int> dist(m * n, INT32_MAX);
        dist[0] = 0;
        vector<bool> seen(m * n, false);

        while (!q.empty()) {
            auto [d, x, y] = q.top();
            q.pop();
            int id = x * n + y;
            if (seen[id]) continue;
            if (x == m - 1 && y == n - 1) break;
            seen[id] = true;

            for (int i = 0; i < 4; ++i) {
                int nx = x + dirs[i][0];
                int ny = y + dirs[i][1];
                if (nx >= 0 && nx < m && ny >= 0 && ny < n) {
                    // heuristic function
                    int effort = max(-d, abs(heights[x][y] - heights[nx][ny]));
                    if (effort < dist[nx * n + ny]) {
                        dist[nx * n + ny] = effort;
                        q.emplace(-effort, nx, ny);
                    }
                }

            }
        }
        return dist[m * n - 1];
    }

private:
    static constexpr int dirs[4][2] = {
        {-1, 0},
        {1, 0},
        {0, -1},
        {0, 1}
    };

    // Cell: height_diff, x, y
    typedef tuple<int, int, int> Cell;
};
```

## 参考

- [1] [启发式搜索](https://oi-wiki.org/search/heuristic/)

