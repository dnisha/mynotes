---
title: "LeetCode 0152 - Maximum Product Subarray"
tags:
  - dsa
  - leetcode
  - medium
  - array
  - dynamic-programming
  - prefix-suffix
---

# [LeetCode 0152 - Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray/)

- **Difficulty:** Medium
- **Topics:** Array, Dynamic Programming, Prefix / Suffix Product

---

## Problem Description

Given an integer array `nums`, find a contiguous non-empty subarray within the array that has the largest product, and return *the product*.

The test cases are generated so that the answer will fit in a **32-bit** integer.

---

## Approach 1: Brute Force

### Intuition
Check every possible contiguous subarray $(i, j)$ where $0 \le i \le j < N$. For each starting index `i`, continuously multiply elements as `j` expands from `i` to `N - 1`, tracking the maximum product observed so far.

### Code Implementation

```python
class Solution:
    def maxProduct(self, nums: list[int]) -> int:
        n = len(nums)
        max_product = float('-inf')

        for i in range(n):
            current_product = 1
            for j in range(i, n):
                current_product *= nums[j]
                max_product = max(max_product, current_product)

        return max_product
```

### Dry Run Example (`nums = [2, 3, -2, 4]`)

| i | j | `nums[j]` | `current_product` | `max_product` |
| :---: | :---: | :---: | :---: | :---: |
| 0 | 0 | 2 | 2 | 2 |
| 0 | 1 | 3 | 6 | **6** |
| 0 | 2 | -2 | -12 | 6 |
| 0 | 3 | 4 | -48 | 6 |
| 1 | 1 | 3 | 3 | 6 |
| 1 | 2 | -2 | -6 | 6 |
| 1 | 3 | 4 | -24 | 6 |
| 2 | 2 | -2 | -2 | 6 |
| 2 | 3 | 4 | -8 | 6 |
| 3 | 3 | 4 | 4 | 6 |

**Output:** `6`

### Complexity Analysis
- **Time Complexity:** $\mathcal{O}(N^2)$ — Two nested loops iterating over all possible pairs of indices $(i, j)$.
- **Space Complexity:** $\mathcal{O}(1)$ — Uses constant extra space for `max_product` and `current_product`.

---

## Approach 2: Optimal Prefix and Suffix Products

### Intuition & Key Insight
1. **No Negative / Even Negatives:** If the array contains positive numbers and an even count of negative numbers, multiplying all elements yields the maximum product.
2. **Odd Negatives:** If the array contains an odd number of negative numbers, dropping either the prefix up to the first negative number or the suffix after the last negative number leaves an even count of negative numbers. Hence, the maximum product subarray must be either a **prefix product** or a **suffix product**.
3. **Zeros:** If the array contains `0`, multiplying by `0` resets the product to `0`. A zero acts as a divider splitting the array into separate subarrays. Whenever `prefix_product` or `suffix_product` becomes `0`, we reset it back to `1` on the next step.

Thus, traversing simultaneously from left-to-right (`prefix_product`) and right-to-left (`suffix_product`) guarantees capturing the maximum product subarray in a single pass.

### Code Implementation

```python
class Solution:
    def maxProduct(self, nums: list[int]) -> int:
        n = len(nums)
        prefix_product = 1
        suffix_product = 1
        max_product = float('-inf')

        for i in range(n):
            if prefix_product == 0:
                prefix_product = 1
            if suffix_product == 0:
                suffix_product = 1

            prefix_product *= nums[i]
            suffix_product *= nums[n - 1 - i]

            max_product = max(max_product, prefix_product, suffix_product)

        return max_product
```

### Dry Run Example (`nums = [2, 3, -2, 4]`)

| i | `nums[i]` (Prefix) | `nums[n-1-i]` (Suffix) | `prefix_product` | `suffix_product` | `max_product` |
| :---: | :---: | :---: | :---: | :---: | :---: |
| 0 | 2 | 4 | 2 | 4 | 4 |
| 1 | 3 | -2 | 6 | -8 | **6** |
| 2 | -2 | 3 | -12 | -24 | 6 |
| 3 | 4 | 2 | -48 | -48 | 6 |

**Output:** `6`

### Complexity Analysis
- **Time Complexity:** $\mathcal{O}(N)$ — Single pass through the array of length $N$.
- **Space Complexity:** $\mathcal{O}(1)$ — Constant space used for tracking prefix, suffix, and max product.

---

## Approach 3: Dynamic Programming / Modified Kadane's Algorithm

### Intuition
Unlike addition, multiplying by a negative number flips maximums into minimums and minimums into maximums. Therefore, at each step we track both:
- `cur_max`: Max product ending at index `i`.
- `cur_min`: Min (most negative) product ending at index `i`.

When we encounter a negative number, multiplying swaps the roles of `cur_max` and `cur_min`.

### Code Implementation

```python
class Solution:
    def maxProduct(self, nums: list[int]) -> int:
        res = max(nums)
        cur_min, cur_max = 1, 1

        for n in nums:
            if n == 0:
                cur_min, cur_max = 1, 1
                continue
            
            tmp = cur_max * n
            cur_max = max(n * cur_max, n * cur_min, n)
            cur_min = min(tmp, n * cur_min, n)
            res = max(res, cur_max)

        return res
```

### Complexity Analysis
- **Time Complexity:** $\mathcal{O}(N)$ — One pass through the array.
- **Space Complexity:** $\mathcal{O}(1)$ — Scalar state tracking variables.

---

## Edge Cases and Pitfalls

1. **Single Negative Element:** e.g., `nums = [-2]`. `max_product` correctly initialized to `float('-inf')` and updated to `-2`.
2. **Zeros in Array:** e.g., `nums = [-2, 0, -1]`. Zeros reset products to `1`, isolating zero-segmented subarrays.
3. **Negative Numbers Count:** Handled via prefix/suffix or dynamic min/max tracking.
