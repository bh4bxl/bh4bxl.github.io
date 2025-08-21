---
layout: post
title: Manacher 算法
date: 2025-08-21
tag: [algorithm]
categories: Algorithm
---

Manacher 算法是 Glenn K. Manacher 在 1975 年提出的。这个算法是用来找一个字符串的最长回文（Palindrome）子串的线性方法（\\(O(n)\\)）。

## 步骤

### 1. 扩展字符串

在原字符串\\(s\\)的每一个字符中间添加一个字符 ‘``#``’，然后在字符串的收尾添加 ‘``^#``’ 和 ‘``#$``’，成为新的字符串\\(t\\)。通过变换后，字符串\\(t\\)的字符数就是奇数了。同时处理字符串\\(t\\)的时候，如果遇到了 ‘``^``’ 和 ‘``$``’，就表示到了边界（假设字符串中不包含这两个字符，也可以用其他的字符串中不包含的字符代替）。然后用一个数组\\(p\\)来保存字符串\\(t\\)的每个字符的最大回文数。

```
s = "abcbaade"
t = "^#a#b#c#b#a#a#d#e#$"
p = [0, 0, 1, 0, 3, 0, 5, 0, 3, 0, 1, 2, 1, 0, 1, 0, 1, 0, 0]
```

回文串的起始位置坐标\\((i-p[i])/2\\)，长度就是\\(p[i]\\)。针对上面的\\(p\\)，就是\\((6-5)/2=0\\)，所以最大回文串的起始坐标就是\\(0\\)。长度就是\\(5\\)。

### 2. 计算\\(p[i]\\)

计算\\(p[i]\\)是 Manacher 算法的关键。

如果\\(c\\)是回文串中心的位置，\\(r\\)是回文串最右边的位置，则\\(r=c+p[c]\\)。对于\\(i\\)（\\(c<i<r\\)），\\(p[i]=p[i_m]\\)。其中\\(i_m\\)是位置\\(i\\)关于位置\\(c\\)的镜像位置。

但是有3种情况下，\\(p[i]\neq p[i_m]\\)：

#### a. \\(p[c]+p[i_m]>r\\)

此时不能用对称性了，但是可以扩展到\\(r\\)，至少有\\(p[i]>=r-1\\)。以\\(i\\)为中心，\\(r+1\\)开始，通过中心扩展来找回文半径（不用从0开始了）。

#### b. \\(i_m\\)遇到原字符串\\(s\\)的左边界

此时\\(p[i_m]=1\\)，由于是\\(i_m\\)在扩展的时候遇到边界终止的，而\\(i\\)还没有遇到边界，所以不能简单的\\(p[i]=p[i_m]\\)，而是用中心扩展法找到\\(p[i]\\)。

#### c. \\(i=r\\)

这种情况下，用中心扩展法从0开始找就可以了。

### 3. 更新\\(c\\)和\\(r\\)

当\\(p[i]\\)的右边界大于当前的\\(r\\)时，就需要更新\\(c\\)和\\(r\\)为当前的回文串了。

## 复杂度

时间复杂度：\\(O(n)\\)

空间复杂度：\\(O(n)\\)

## 应用

### leetcode 5

[leetcode 5 Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/)

```rust
pub fn longest_palindrome(s: String) -> String {
    if s.is_empty() {
        return String::new();
    }

    // Work over chars to be Unicode-safe at the code-point level.
    let chars: Vec<char> = s.chars().collect();

    // Transform: "^#a#b#b#a#$" style (sentinels avoid bound checks).
    let mut t = Vec::with_capacity(chars.len() * 2 + 3);
    t.push('^');                 // start sentinel
    t.push('#');
    for &c in &chars {
        t.push(c);
        t.push('#');
    }
    t.push('$');                 // end sentinel

    let n = t.len();
    let mut p = vec![0usize; n]; // p[i] = radius around i in transformed string
    let (mut center, mut right) = (0usize, 0usize);

    for i in 1..(n - 1) {
        // Mirror of i around current center.
        let mirror = 2 * center - i;

        // If we're inside the current right boundary, use the mirror's radius as a lower bound.
        if i < right {
            p[i] = p[mirror].min(right - i);
        }

        // Expand around i while the characters match.
        while t[i + 1 + p[i]] == t[i - 1 - p[i]] {
            p[i] += 1;
        }

        // Update center/right if we've expanded beyond the current right.
        if i + p[i] > right {
            center = i;
            right = i + p[i];
        }
    }

    // Find the maximum radius and its center.
    let (mut max_len, mut center_idx) = (0usize, 0usize);
    for i in 1..(n - 1) {
        if p[i] > max_len {
            max_len = p[i];
            center_idx = i;
        }
    }

    // Map back to original indices in `chars`.
    // In this transform, start = (center_idx - max_len) / 2
    let start = (center_idx - max_len) / 2;
    chars[start..start + max_len].iter().collect()
}
```
## 参考

- [1][Manacher](https://oi-wiki.org/string/manacher/)
- [2][Manacher's Algorithm - Finding all sub-palindromes in O(N)](https://cp-algorithms.com/string/manacher.html)
- [3][一文让你彻底明白马拉车算法](https://zhuanlan.zhihu.com/p/70532099)
