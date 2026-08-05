---
title: "LeetCode 0146 - LRU Cache"
tags:
  - dsa
  - leetcode
  - medium
  - hash-table
  - linked-list
  - design
---

# [LeetCode 0146 - LRU Cache](https://leetcode.com/problems/lru-cache/)

- **Difficulty:** Medium
- **Topics:** Hash Table, Doubly Linked List, Design
- **Companies:** Amazon, Google, Meta, Apple, Microsoft

---

## Optimal Approach: Hash Map + Doubly Linked List

### Intuition
To achieve $O(1)$ time for both `get` and `put` operations:
- A **Hash Map** provides $O(1)$ key lookup to node pointers.
- A **Doubly Linked List** allows $O(1)$ removal and insertion at head/tail.

### Code Implementation

```python
class Node:
    def __init__(self, key=0, val=0):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {} # key -> Node
        self.head = Node() # Dummy head
        self.tail = Node() # Dummy tail
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node: Node):
        prev, nxt = node.prev, node.next
        prev.next = nxt
        nxt.prev = prev

    def _add_to_head(self, node: Node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key: int) -> int:
        if key in self.cache:
            node = self.cache[key]
            self._remove(node)
            self._add_to_head(node)
            return node.val
        return -1

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self._remove(self.cache[key])
        
        node = Node(key, value)
        self.cache[key] = node
        self._add_to_head(node)

        if len(self.cache) > self.capacity:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```

### Complexity Analysis
- **Time Complexity:** $\mathcal{O}(1)$ for both `get()` and `put()`.
- **Space Complexity:** $\mathcal{O}(\text{capacity})$ for hash map and doubly linked list.
