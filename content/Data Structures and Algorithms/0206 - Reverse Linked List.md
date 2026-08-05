---
title: "LeetCode 0206 - Reverse Linked List"
tags:
  - dsa
  - leetcode
  - easy
  - linked-list
---

# [LeetCode 0206 - Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)

- **Difficulty:** Easy
- **Topics:** Linked List, Recursion

---

## Optimal Approach: Iterative Reversal

### Intuition
Maintain three pointers: `prev`, `curr`, and `next_node`. At each step, flip `curr.next` to point to `prev`.

### Code Implementation

```python
# Definition for singly-linked list.
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

class Solution:
    def reverseList(self, head: ListNode) -> ListNode:
        prev = None
        curr = head
        while curr:
            next_node = curr.next
            curr.next = prev
            prev = curr
            curr = next_node
        return prev
```

### Complexity Analysis
- **Time Complexity:** $\mathcal{O}(N)$
- **Space Complexity:** $\mathcal{O}(1)$
