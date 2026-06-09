# test 3
This is a **coding interview question**, not LLD.

They are asking for **top K nearest lockers** to a customer location.

## Problem

Given:

```text
Customer location = [x, y]
Locker locations = [[x1, y1], [x2, y2], ... [xn, yn]]
k = number of lockers to return
```

Return the **k closest lockers** to the customer.

---

## Clarification questions to ask

You can ask these quickly:

1. Should distance be **Euclidean distance** or Manhattan distance?
2. If two lockers have the same distance, can we return in any order?
3. Can `k` be greater than number of lockers?
4. Do we need exact top k or approximate nearest lockers for very large scale?
5. Are coordinates integers or floating point values?
6. Is this one-time query or will we have many customer queries repeatedly?

For coding round, assume:

```text
Use Euclidean distance
Return any order if tie
If k > N, return all lockers
```

---

## Approach 1: Sort all lockers

Calculate distance of every locker from customer and sort by distance.

```text
distance = (x1 - x)^2 + (y1 - y)^2
```

No need to use square root because relative ordering stays the same.

### Strength

Simple and clean.

### Weakness

Sorting all lockers costs more when we only need top k.

### Time Complexity

```text
O(N log N)
```

### Space Complexity

```text
O(N)
```

---

## Approach 2: Max Heap of size k

This is better when `k` is much smaller than `N`.

Idea:

Maintain a heap of only k closest lockers.

Python has min heap by default, so we store negative distance to simulate max heap.

For every locker:

1. Calculate distance
2. Push into heap
3. If heap size becomes greater than k, remove the farthest locker

At the end, heap has k nearest lockers.

### Strength

Better when N is large and k is small.

### Weakness

Slightly more logic than sorting.

### Time Complexity

```text
O(N log K)
```

### Space Complexity

```text
O(K)
```

---

## Python Code

```python
import heapq
from typing import List


def get_top_k_lockers(
    lockers: List[List[int]],
    customer: List[int],
    k: int
) -> List[List[int]]:
    
    if not lockers or k <= 0:
        return []

    k = min(k, len(lockers))

    cx, cy = customer
    max_heap = []

    for locker in lockers:
        lx, ly = locker

        distance = (lx - cx) ** 2 + (ly - cy) ** 2

        # Store negative distance to simulate max heap
        heapq.heappush(max_heap, (-distance, locker))

        # Keep only k closest lockers
        if len(max_heap) > k:
            heapq.heappop(max_heap)

    return [locker for distance, locker in max_heap]
```

---

## Example

```python
lockers = [[1, 2], [3, 4], [2, 1], [10, 10]]
customer = [0, 0]
k = 2

print(get_top_k_lockers(lockers, customer, k))
```

### Output

```python
[[2, 1], [1, 2]]
```

Both are among the closest lockers.

---

## What to say in interview

I would start with a simple solution where I calculate the distance from the customer to every locker and sort the lockers by distance. That gives me the top k lockers easily, but it costs `O(N log N)`.

Since we only need k lockers, a better approach is to keep a max heap of size k. For each locker, I calculate the squared Euclidean distance. I push it into the heap, and if the heap size becomes greater than k, I remove the farthest locker. This way, the heap always keeps only the k nearest lockers seen so far.

The final complexity becomes `O(N log K)`, which is better when we have many lockers and k is small. Space is `O(K)`.

For production scale, if we have millions of lockers and many repeated customer queries, I would not scan all lockers every time. I would use spatial indexing like geohash, KD-tree, or grid-based partitioning to first narrow down nearby lockers, then apply the heap logic on that smaller candidate set.
