---
title: "LeetCode 0001 - Two Sum"
tags:
  - dsa
  - leetcode
  - easy
  - array
  - hash-table
---

# [LeetCode 0001 - Two Sum](https://leetcode.com/problems/two-sum/)

- **Difficulty:** Easy
- **Topics:** Array, Hash Table
- **Companies:** Google, Amazon, Meta, Apple, Microsoft

---

## Problem Description

Given an array of integers `nums` and an integer `target`, return indices of the two numbers such that they add up to `target`.

You may assume that each input would have **exactly one solution**, and you may not use the same element twice.

---

## Approach 1: Brute Force

### Intuition
Check every pair $(i, j)$ where $i \neq j$ and see if $nums[i] + nums[j] == target$.

### Complexity Analysis
- **Time Complexity:** $\mathcal{O}(N^2)$
- **Space Complexity:** $\mathcal{O}(1)$

---

## Approach 2: One-Pass Hash Map (Optimal)

### Intuition & Key Insight
As we iterate through the array, for each element `num`, we need to find if its complement `complement = target - num` exists in our previously visited numbers. Using a Hash Map (Hash Table) allows $O(1)$ lookups.

### Algorithm Steps
1. Initialize an empty hash map `seen` storing `{number: index}`.
2. Loop through `nums` with index `i` and element `num`:
   - Calculate `complement = target - num`.
   - If `complement` is in `seen`, return `[seen[complement], i]`.
   - Otherwise, store `seen[num] = i`.

### Code Implementation

```python
class Solution:
    def twoSum(self, nums: list[int], target: int) -> list[int]:
        seen = {}
        for i, num in enumerate(nums):
            complement = target - num
            if complement in seen:
                return [seen[complement], i]
            seen[num] = i
        return []
```

### Complexity Analysis
- **Time Complexity:** $\mathcal{O}(N)$ — We traverse the list containing $N$ elements only once.
- **Space Complexity:** $\mathcal{O}(N)$ — Hash map stores up to $N$ elements.

---

## Edge Cases and Pitfalls
- Duplicate numbers (e.g. `[3, 3]`, target `6` -> index 0 and 1).
- Negative values in array.
