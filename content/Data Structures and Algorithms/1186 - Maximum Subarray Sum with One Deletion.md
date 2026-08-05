---
title: "LeetCode 1186 - Maximum Subarray Sum with One Deletion"
tags:
  - dsa
  - leetcode
  - medium
  - array
  - dynamic-programming
  - kadanes-algorithm
---

# [LeetCode 1186 - Maximum Subarray Sum with One Deletion](https://leetcode.com/problems/maximum-subarray-sum-with-one-deletion/)

- **Difficulty:** Medium
- **Topics:** Array, Dynamic Programming, Kadane's Algorithm

---

## Problem Description

Given an array of integers `arr`, return the maximum sum for a non-empty subarray with at most one element deletion. The resulting subarray must not be empty.

---

## Approach 1: Brute Force (Element Deletion Simulation)

### Intuition
Simulate deleting every element at index `delete_idx` one by one to form a new list `remaining`. For each deleted configuration, run a standard brute-force maximum subarray sum check.

### Code Implementation

```python
arr = [1, -2, 0, 3]


def max_subarray_sum(nums):
    """
    Finds the maximum subarray sum using brute force.
    Time: O(n²)
    """
    best = float("-inf")

    for start in range(len(nums)):
        current_sum = 0

        for end in range(start, len(nums)):
            current_sum += nums[end]
            best = max(best, current_sum)

    return best


def maximum_sum_with_one_deletion(arr):
    """
    Try deleting every element once and compute
    the maximum subarray sum.
    """
    answer = float("-inf")

    for delete_idx in range(len(arr)):
        remaining = arr[:delete_idx] + arr[delete_idx + 1:]
        answer = max(answer, max_subarray_sum(remaining))

    return answer


print(maximum_sum_with_one_deletion(arr))
```

### Complexity Analysis
- **Time Complexity:** $\mathcal{O}(N^3)$ — There are $N$ choices for the deleted index. Slicing takes $O(N)$ and computing maximum subarray sum via nested loops takes $O(N^2)$, resulting in $O(N \cdot N^2) = O(N^3)$.
- **Space Complexity:** $\mathcal{O}(N)$ — Additional space required to store the `remaining` array after slicing.

---

## Approach 2: Optimal Dynamic Programming (Extended Kadane's Algorithm)

### Intuition & Key Insight
Instead of physically deleting elements and recomputing sums, we can extend Kadane's Algorithm to maintain state at each element `arr[i]`:

1. `no_deletion`: The maximum subarray sum ending at index `i` with **zero** deletions.
2. `one_deletion`: The maximum subarray sum ending at index `i` with **one** deletion performed.

At each element `x = arr[i]`:
- `one_deletion` can either be:
  - Deleting the current element `x`: Inheriting `prev_no_deletion` (the max subarray sum ending right before `x` with 0 deletions).
  - Keeping current element `x` after having already deleted an element earlier: `prev_one_deletion + x`.
- `no_deletion` is standard Kadane's: `max(x, prev_no_deletion + x)`.

We update our global maximum `max_sum` with both `no_deletion` and `one_deletion` at each step.

### Algorithm Steps
1. Handle base case where array length is 1: return `arr[0]`.
2. Initialize `no_deletion = arr[0]`, `one_deletion = float('-inf')`, and `max_sum = arr[0]`.
3. Iterate through `arr` starting from index 1:
   - Calculate new `one_deletion = max(no_deletion, one_deletion + x)`.
   - Calculate new `no_deletion = max(x, no_deletion + x)`.
   - Update `max_sum = max(max_sum, no_deletion, one_deletion)`.
4. Return `max_sum`.

### Code Implementation

```python
class Solution:
    def maximumSum(self, arr: list[int]) -> int:
        if not arr:
            return 0
        
        no_deletion = arr[0]
        one_deletion = float("-inf")
        max_sum = arr[0]

        for i in range(1, len(arr)):
            x = arr[i]
            
            # Choice 1: Delete current element x -> keep previous no_deletion sum
            # Choice 2: Keep current element x -> extend previous one_deletion sum
            one_deletion = max(no_deletion, one_deletion + x)
            
            # Standard Kadane's Algorithm for no deletion state
            no_deletion = max(x, no_deletion + x)
            
            # Track peak sum across both states
            max_sum = max(max_sum, no_deletion, one_deletion)

        return max_sum
```

### Complexity Analysis
- **Time Complexity:** $\mathcal{O}(N)$ — Single pass iteration over array of length $N$.
- **Space Complexity:** $\mathcal{O}(1)$ — Constant extra space using variables to store current state.

---

## Edge Cases and Pitfalls
- **All Negative Numbers**: e.g., `[-1, -2, -3]`. Deleting the smallest negative leaves a single negative value as max sum.
- **Single Element Array**: e.g., `[-5]`. Subarray cannot be empty after deletion, so result is `-5`.
