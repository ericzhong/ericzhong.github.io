---
layout: post
title: 求无重复字符的最长子串的长度
tag: 算法
category: 计算机
---



LeetCode 上的一道题：[Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/#/description) ，随机给一个字符串（可以为空串），求出无重复字符的最长子串，然后返回它的长度。

我的方案如下，时间复杂度 O(n)：

```python
class Solution(object):
    def lengthOfLongestSubstring(self, s):
        """
        :type s: str
        :rtype: int
        """
        max_len = sub_len = start = 0
        d = {}
        for i,c in enumerate(s):
            if c in d and d[c] >= start:
                start = d[c] + 1
                sub_len = i - start
            sub_len += 1
            d[c] = i
            if sub_len > max_len:
                max_len = sub_len
        return max_len
```



这种数数的题很容易头晕，把各种可能都列出来更容易理解：

* 如果输入是 `ab` ，那么 `not in d`，执行两次 `sub_len++` ，结果为 2。
* 如果输入是空串，不走循环，返回 `max_len = 0` 的初始值。
* 如果是下面的情况：

```
0 1 2 3 4
a b c d c
↑       ↑  
start   i
```

​	也就是 `c in d`，即发现重复了，且老的 `c` 在 `start` 后面，那就先把子串长度记录下来 `sub_len = i - start = 4 - 0 = 4`，然后 `start` 要从字母 `d` 开始，即 `start = d[c] + 1 = 2 + 1 = 3`，这时 `max_len` 还是 0，因此把 `sub_len` 的值给它保存。

​	然后，看起来就是下面的样子：

```
0 1 2 3 4
a b c d c 
      ↑ ↑  
  start i
```

​	这时  `sub_len = i - start = 4 - 3 = 1`，然后再自增一次变 2，这就对了。

* 如果再往下走，变成这样：

```
0 1 2 3 4
a b c d c e f m n b k j
      ↑           ↑  
  start           i
```

这时 `b in d`，但是老的 `b` 在 `start` 的后面，即 `d[b] < start`，所以不用管它，接着往下走。

