---
title: "{::nomarkdown}266.{:/} Palindrome Permutation"
date: 2020-01-03
categories:
  - cpps
tags:
  - Hash
  - defaultdict
  - Counter
  - set
---
## Problem: [266. Palindrome Permutation](https://leetcode.com/problems/palindrome-permutation/)

## Sol 1a. defaultdict
```python
"""
Time: O(N)
Space: O(1) # max: 27
"""

from collections import defaultdict

class Solution:
    def canPermutePalindrome(self, s: str) -> bool:
        d = defaultdict(int)
        for c in s:
            d[c] += 1
        num_odds = 0
        for cnt in d.values():
            if cnt % 2 != 0:
                num_odds += 1
            if num_odds > 1:
                return False
        return True
```

## Sol 1b. Counter
```python
"""
Time: O(N)
Space: O(1) # max: 27
"""

from collections import Counter

class Solution:
    def canPermutePalindrome(self, s: str) -> bool:
        d = Counter(s)
        num_odds = 0
        for cnt in d.values():
            if cnt % 2 != 0:
                num_odds += 1
            if num_odds > 1:
                return False
        return True
```

## Sol 2. set
```python
"""
Time: O(N)
Space: O(1) # max: 27
"""

class Solution:
    def canPermutePalindrome(self, s: str) -> bool:
        odds = set()
        for c in s:
            if c in odds:
                odds.remove(c)
            else:
                odds.add(c)
        return len(odds) <= 1
```
