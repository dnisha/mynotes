---
title: "LeetCode 0003 - Longest Substring Without Repeating Characters"
tags:
  - dsa
  - leetcode
  - medium
  - string
  - sliding-window
  - hash-table
---

# [LeetCode 0003 - Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)

- **Difficulty:** Medium
- **Topics:** Hash Table, String, Sliding Window
- **Companies:** Meta, Amazon, Google, Microsoft

---

## Problem Description

Given a string `s`, find the length of the **longest substring** without repeating characters.

---

## Optimal Approach: Sliding Window + Hash Map / Character Index Map

### Intuition & Key Insight
Maintain a sliding window `[left, right]`. As `right` advances, if we encounter a repeating character `c` that was last seen at index `last_seen[c]`, we can jump `left` directly to `max(left, last_seen[c] + 1)`.

### Algorithm Steps
1. Create a dictionary `char_map` to map characters to their most recent index.
2. Maintain `left = 0` and `max_len = 0`.
3. Iterate `right` from `0` to `len(s) - 1`:
   - If `s[right]` is in `char_map` and its recorded index $\ge$ `left`, move `left` to `char_map[s[right]] + 1`.
   - Update `char_map[s[right]] = right`.
   - Update `max_len = max(max_len, right - left + 1)`.
4. Return `max_len`.

### Code Implementation

```python
class Solution:
    def lengthOfLongestSubstring(self, s: str) -> int:
        char_map = {}
        left = 0
        max_len = 0

        for right, char in enumerate(s):
            if char in char_map and char_map[char] >= left:
                left = char_map[char] + 1
            char_map[char] = right
            max_len = max(max_len, right - left + 1)

        return max_len
```

### Complexity Analysis
- **Time Complexity:** $\mathcal{O}(N)$ — Single pass over string of length $N$.
- **Space Complexity:** $\mathcal{O}(\min(N, M))$ — Where $M$ is the alphabet/charset size (e.g. 26 for lowercase English or 128 for ASCII).
