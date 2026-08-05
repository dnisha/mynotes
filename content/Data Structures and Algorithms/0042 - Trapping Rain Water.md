---
title: "LeetCode 0042 - Trapping Rain Water"
tags:
  - dsa
  - leetcode
  - hard
  - two-pointers
  - array
---

# [LeetCode 0042 - Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/)

- **Difficulty:** Hard
- **Topics:** Array, Two Pointers, Dynamic Programming, Monotonic Stack

---

## Optimal Approach: Two Pointers

### Intuition
Water trapped at any index $i$ is determined by $\min(\text{max\_left}, \text{max\_right}) - \text{height}[i]$.
By using two pointers moving inward (`left` and `right`), we can maintain `left_max` and `right_max` dynamically.

### Code Implementation

```python
class Solution:
    def trap(self, height: list[int]) -> int:
        if not height:
            return 0
            
        left, right = 0, len(height) - 1
        left_max, right_max = height[left], height[right]
        water = 0

        while left < right:
            if left_max < right_max:
                left += 1
                left_max = max(left_max, height[left])
                water += left_max - height[left]
            else:
                right -= 1
                right_max = max(right_max, height[right])
                water += right_max - height[right]

        return water
```

### Complexity Analysis
- **Time Complexity:** $\mathcal{O}(N)$ — Single pass using two pointers.
- **Space Complexity:** $\mathcal{O}(1)$ — Constant extra space.
