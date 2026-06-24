# Algorithms and Data Structures

A quick-reference guide to the data structures and algorithms an engineer should know cold — not as LeetCode preparation, but as a mental library for choosing the right tool. Each structure or technique has known costs, strengths, and weaknesses; selection depends on matching those to the workload.

---

## Table of Contents

1. Complexity — Big-O, Amortization, Practical Notes
2. Arrays and Slices
3. Linked Lists
4. Stacks and Queues
5. Hash Tables
6. Trees
7. Heaps and Priority Queues
8. Tries
9. Disjoint Sets (Union-Find)
10. Graphs
11. Sorting
12. Searching
13. Recursion, Backtracking, Memoization
14. Dynamic Programming
15. Greedy Algorithms
16. Divide and Conquer
17. Probabilistic and Approximate Structures

---

## 1. Complexity — Big-O, Amortization, Practical Notes

### Big-O Basics

Asymptotic upper bound on growth. Three bounds are distinct: **O** is an upper bound, **Ω** a lower bound, and **Θ** a tight bound (both at once). Common usage writes O where Θ is meant — "quicksort is O(n log n) on average" asserts a tight average bound. This note follows that convention; the worst/average/best columns below supply the precision the loose O omits.

```
O(1)          constant       hash lookup, stack push
O(log n)      logarithmic    binary search, balanced tree op
O(n)          linear         scanning an array
O(n log n)    linearithmic   good sort, balanced tree build
O(n^2)        quadratic      nested loops, bubble sort
O(n^3)        cubic          three nested loops, naive matrix mul
O(2^n)        exponential    brute force subset enumeration
O(n!)         factorial      enumerate permutations
```

The constant factor matters in practice (a O(n^2) algorithm with tiny constants can beat an O(n log n) with huge constants on small inputs), but at scale, asymptotic behavior dominates.

### Best, Average, Worst

| Algorithm | Best | Average | Worst |
|-----------|------|---------|-------|
| Quicksort | n log n | n log n | n^2 (sorted input, bad pivot) |
| Hash insert | 1 | 1 | n (all collisions) |
| Binary search | 1 | log n | log n |
| Heap pop | log n | log n | log n |

Quote average for typical reasoning; quote worst when robustness matters (real-time systems, security-sensitive code, adversarial inputs).

### Amortized Analysis

The average cost per operation over a sequence, even when individual operations are sometimes expensive.

**Example: dynamic array resize.**

```
Push to a vector backed by an array:
  Most pushes: O(1) — just append.
  When full: allocate 2x capacity, copy n elements — O(n).

Over n pushes: ~2n total work (each element copied ~once across resizes).
Amortized: O(1) per push.
```

Used by: Python list, Go slice, Java ArrayList, C++ std::vector. The doubling factor makes the expensive resizes rare.

### Space Complexity

Same notation as time. Often the key constraint:

```
n bytes can fit in:
  RAM        (smallest, fastest)
  SSD        (larger, slower)
  object store (effectively unbounded, but slow access)
```

Algorithms that need O(n) extra space (merge sort) differ in practice from in-place ones (quicksort) on memory-constrained systems.

### Practical Performance Notes

- **Cache effects dominate at small scale.** An O(n^2) algorithm with sequential array access can beat an O(n log n) algorithm with pointer chasing on millions of elements.
- **Constants matter.** A 100x constant factor difference can swamp asymptotic differences over a realistic input range.
- **Allocation is expensive.** Algorithms with low allocation can outperform asymptotically-better ones with high allocation.
- **Branch prediction.** Predictable branches are free; unpredictable ones cost ~10 cycles.

---

## 2. Arrays and Slices

An array is a fixed-length, contiguous block of memory holding equal-size elements, each located by an integer index through `base + index * element_size` arithmetic — hence O(1) random access. The most fundamental data structure.

```
  index:     0     1     2     3     4
           +-----+-----+-----+-----+-----+
           |  10 |  20 |  30 |  40 |  50 |
           +-----+-----+-----+-----+-----+
  addr:    100   104   108   112   116        (4-byte elements)

  array[3] lives at base + 3*4 = 112 -> found by arithmetic, no scan, O(1)
```

### Operations

| Operation | Cost |
|-----------|------|
| Index access | O(1) |
| Update at index | O(1) |
| Append (amortized, dynamic array) | O(1) |
| Insert / delete at end | O(1) |
| Insert / delete at middle | O(n) — shift elements |
| Search (unsorted) | O(n) |
| Binary search (sorted) | O(log n) |

### Why Arrays Are Fast

- **Cache locality.** Contiguous memory; the CPU prefetches subsequent elements.
- **No pointer chasing.** Indexing is arithmetic, not dereference.
- **SIMD-friendly.** Operations on consecutive elements vectorize (SIMD: single instruction, multiple data — one operation applied across many values at once).

A linked list with the same number of elements often runs 10-100x slower in practice for sequential access.

### Dynamic Arrays (Vec, List, Slice)

A dynamic array is an array that grows automatically: it keeps a separate capacity and length, and when full it allocates a larger backing array (typically 2x) and copies the elements over, giving amortized O(1) append (see section 1).

Pre-allocate capacity when the size is known:

```python
result = [None] * n      # preallocated
for i in range(n):
    result[i] = compute(i)

# vs
result = []
for i in range(n):
    result.append(compute(i))   # triggers resizes
```

Both end at the same place; the first avoids reallocations.

### Two-Pointer Pattern

The two-pointer technique solves an array/string problem with two indices that traverse the data under a controlled rule — moving toward each other from the ends, or as a fast/slow pair — collapsing what looks like a nested-loop O(n^2) scan into a single O(n) pass.

```python
# Remove duplicates from a sorted array in place:
def dedup(arr):
    write = 0
    for read in range(len(arr)):
        if read == 0 or arr[read] != arr[read-1]:
            arr[write] = arr[read]
            write += 1
    return arr[:write]
```

O(n) time, O(1) extra space.

### Sliding Window

The sliding-window technique maintains a contiguous range [left, right) over an array or string, expanding the right edge to include elements and contracting the left edge to restore a constraint, so each element enters and leaves the window once — an O(n) scan over all subarrays of interest.

```python
# Longest substring with at most K distinct chars:
def longest_with_k_distinct(s, k):
    counts = {}
    left = best = 0
    for right, ch in enumerate(s):
        counts[ch] = counts.get(ch, 0) + 1
        while len(counts) > k:
            counts[s[left]] -= 1
            if counts[s[left]] == 0:
                del counts[s[left]]
            left += 1
        best = max(best, right - left + 1)
    return best
```

O(n) instead of O(n * k).

### Prefix Sums and Range Queries

A prefix-sum array `P` stores `P[i] = arr[0] + ... + arr[i-1]`, built in O(n). Any range sum is then `P[r+1] - P[l]` in O(1). The technique generalizes to any invertible aggregate (counts, XOR) and to 2D (`O(1)` submatrix sums from a 2D prefix table). It assumes the array is **static**: a single update forces rebuilding the suffix, O(n).

For ranges over a **mutable** array, two array-backed trees give O(log n) per query and per update:

- **Fenwick tree (binary indexed tree)** — supports prefix-sum queries and point updates in O(log n) using O(n) space and a few lines of code. Restricted to invertible aggregates (range sum = prefix(r) − prefix(l)).
- **Segment tree** — stores an aggregate per array segment in a complete tree; handles any associative aggregate (sum, min, max, gcd), supports range queries and point updates in O(log n), and with **lazy propagation** supports range updates in O(log n). More general than Fenwick, at roughly double the space and constant factor.

| Structure | Query | Point update | Range update | Aggregates |
|-----------|-------|--------------|--------------|------------|
| Prefix sum | O(1) | O(n) | O(n) | invertible only |
| Fenwick | O(log n) | O(log n) | — (without tricks) | invertible only |
| Segment tree | O(log n) | O(log n) | O(log n) lazy | any associative |

---

## 3. Linked Lists

A linked list is a linear sequence of nodes, each holding an element and a reference to the next node (singly linked), or to both neighbors (doubly linked). Elements are not contiguous in memory, so access is by traversal from an endpoint, not by index.

```
  Singly linked (each node = data + pointer to next; last points to null):

    head
     |
     v
    +----+---+    +----+---+    +----+---+
    | 10 | *-+--> | 20 | *-+--> | 30 | / |     (/ = null)
    +----+---+    +----+---+    +----+---+

  Doubly linked (each node also points back to the previous):

    null <-> | 10 | <-> | 20 | <-> | 30 | <-> null
```

### Operations

| Operation | Singly | Doubly |
|-----------|--------|--------|
| Index access | O(n) | O(n) |
| Insert at head | O(1) | O(1) |
| Insert at tail | O(n) or O(1) with tail pointer | O(1) |
| Insert before given node | O(n) | O(1) |
| Delete given node | O(n) | O(1) |
| Search | O(n) | O(n) |

### When Linked Lists Are Right

In practice, almost never for general data. They lose to arrays at almost everything because of cache locality.

Specific cases:

- **LRU (least-recently-used) cache** (doubly linked list of recent items + hash map for O(1) lookup).
- **Free lists** in allocators.
- **Concurrent lock-free queues** (Michael-Scott queue).
- **Functional persistent lists** (cons cell, immutable).

### Common Linked-List Tricks

**Fast and slow pointers** (Floyd's cycle detection):

```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False
```

Same trick finds the middle of the list (slow points to middle when fast hits end).

**Reverse in place:**

```python
def reverse(head):
    prev = None
    curr = head
    while curr:
        next_node = curr.next
        curr.next = prev
        prev = curr
        curr = next_node
    return prev
```

---

## 4. Stacks and Queues

### Stack

A stack is an abstract data type storing elements under last-in-first-out (LIFO) discipline: insertions (push) and removals (pop) act only on the most recently added element (the top), and peek reads it. All three are O(1).

```
  push --> +----+ <-- pop
           | 30 |        top (most recently pushed)
           +----+
           | 20 |
           +----+
           | 10 |        bottom (first pushed, last to leave)
           +----+
```

Implemented on a dynamic array.

**Used for:**

- Recursion (the call stack is a stack).
- Undo history.
- Expression evaluation.
- DFS (depth-first search).
- Bracket matching / parser symbol stacks.

### Queue

A queue is an abstract data type storing elements under first-in-first-out (FIFO) discipline: insertions (enqueue) happen at the back and removals (dequeue) at the front, so elements leave in arrival order. Both are O(1).

```
  dequeue <-- +----+----+----+----+ <-- enqueue
  (oldest)    | 10 | 20 | 30 | 40 |     (newest)
              +----+----+----+----+
               front           back
```

A naive array implementation is O(n) for dequeue (shift elements). Real queues use:

- **Circular buffer** (ring): fixed-size, O(1) ops.
- **Doubly linked list:** O(1) both ends.
- **Two-stack queue:** amortized O(1), nice for functional implementations.

Most languages have a deque (double-ended queue) supporting O(1) at both ends.

**Used for:**

- BFS (breadth-first search).
- Task queues, work queues.
- Producer-consumer.
- Sliding window via deque.

### Deque

A deque (double-ended queue) is an abstract data type supporting insertion and removal at both the front and the back in O(1). The Swiss army knife — covers stacks, queues, and sliding window structures.

### Monotonic Stack / Deque

A monotonic stack keeps its elements in sorted order by discarding, before each push, every existing element that violates the order. Pushing element x onto a decreasing stack first pops all elements smaller than x; each element is pushed and popped at most once, so a full pass is O(n) total (amortized O(1) per element). It answers **next/previous greater (or smaller) element** queries in a single sweep — the elements popped by x are exactly those for which x is the next greater element.

A monotonic deque applies the same discipline at both ends and yields the **sliding-window maximum/minimum** in O(n): the front always holds the window's extremum, stale indices are dropped from the front, and dominated candidates are dropped from the back.

### Priority Queue

A priority queue is an abstract data type in which each element carries a priority and every removal returns an element of highest priority, regardless of insertion order. Typically backed by a heap (see section 7).

---

## 5. Hash Tables

A hash table is a data structure mapping keys to values by applying a hash function to each key to compute an array index (its bucket), giving average O(1) insert, lookup, and delete. The workhorse of modern programming.

### Operations

| Operation | Average | Worst |
|-----------|---------|-------|
| Insert | O(1) | O(n) |
| Lookup | O(1) | O(n) |
| Delete | O(1) | O(n) |

The worst case is when every key hashes to the same bucket — pathological without good hashing.

### How It Works

```
Array of buckets, each holding (key, value) entries.

Insert (k, v):
  bucket_index = hash(k) % num_buckets
  store (k, v) in that bucket (handle collisions)

Lookup k:
  bucket_index = hash(k) % num_buckets
  scan bucket for matching k
```

```
  key "cat" --hash--> 0x9e3... % 5--> bucket 2

  buckets:   0     1      2       3     4
           +-----+-----+--------+-----+-----+
           |     |     | "cat"  |     |     |
           +-----+-----+--------+-----+-----+
```

### Collision Handling

A collision is when two keys hash to the same bucket. Two ways to resolve it:

**Chaining:** each bucket is a list (or another hash table, or a tree).

```
  0 -> (empty)
  1 -> [ "cat":3 ] -> [ "dog":7 ]      both hashed to bucket 1
  2 -> [ "fish":9 ]
  3 -> (empty)
```

**Open addressing:** find the next free bucket via probing. Variants: linear, quadratic, double hashing.

```
  "cat" -> bucket 1 (free, store)
  "dog" -> bucket 1 (taken) -> probe 2 (free, store)

  index:    0      1        2        3
          +-----+--------+--------+-----+
          |     | "cat"  | "dog"  |     |
          +-----+--------+--------+-----+
```

- **Linear probing** is cache-friendly but suffers from primary clustering.
- **Robin Hood hashing** is open addressing with a fairness trick: insertions push out elements that are closer to their ideal slot. Smooths probe distances.

Deletion under open addressing cannot simply blank a slot: doing so would truncate the probe chain and hide later entries that collided through it. The slot is instead marked with a **tombstone** that lookups probe past and insertions may overwrite. Tombstones accumulate and degrade probe lengths, so a table with heavy churn is rehashed periodically to clear them. Chaining has no such problem — deletion just unlinks the entry.

### Load Factor and Resize

Load factor = elements / buckets. When it exceeds a threshold (typically 0.7-0.9), the table doubles size and rehashes everything. Amortized O(1).

### Hash Function Quality

A bad hash function (e.g., always returning 0) makes everything O(n). Standard library hashes are good for general use. For cryptographic resistance (against an adversary choosing keys to collide), use SipHash or HMAC-keyed (hash-based message authentication code) hashing.

### When NOT to Use a Hash Table

- Ordered iteration is required (use a tree).
- Range queries are required (use a tree).
- A bounded worst-case is required (use a tree).
- Predictable iteration order across runs is required (older Python had a random hash seed; modern dicts preserve insertion order).

### Hash Set

A hash set is a hash table that stores only keys (no associated values), supporting average O(1) membership tests (present / absent).

---

## 6. Trees

A tree is a connected, acyclic graph of nodes in which one node is the root and every other node has exactly one parent; nodes reference their children. With n nodes it has exactly n−1 edges. Hierarchical by construction.

### Binary Tree

A binary tree is a tree in which every node has at most two children, distinguished as the left child and the right child.

```
       1
      / \
     2   3
    / \   \
   4   5   6
```

**Traversals:**

```
Pre-order:   1, 2, 4, 5, 3, 6  (root, left, right)
In-order:    4, 2, 5, 1, 3, 6  (left, root, right) — yields sorted output for BST
Post-order:  4, 5, 2, 6, 3, 1  (left, right, root)
Level-order: 1, 2, 3, 4, 5, 6  (breadth-first search, BFS)
```

### Binary Search Tree (BST)

A binary tree with the invariant: left subtree < node < right subtree.

```
Search:     O(log n) average, O(n) worst (degenerate / unbalanced)
Insert:     O(log n) average
Delete:     O(log n) average
```

**Deletion** has three cases by the number of children of the target node: a leaf is removed outright; a node with one child is replaced by that child; a node with two children is replaced by its **in-order successor** (the minimum of its right subtree), which is then deleted from the right subtree. The successor has at most one child, so the recursion terminates. All cases are O(height).

Plain BSTs degenerate (become linked lists) on sorted input. Use a balanced variant in practice.

### Self-Balancing Trees

A plain BST can degenerate to O(n). Self-balancing trees enforce an O(log n) height bound in the *worst* case by restructuring after each insert/delete. The restructuring primitive is the **rotation** — a local, O(1) pointer rearrangement that changes the tree's height while keeping it a valid BST (the left-to-right order of keys is unchanged).

A rotation swaps a parent and one of its children. Below, `x` and `y` are single nodes; `A`, `B`, `C` are whole subtrees hanging off them. A **right rotation** lifts `x` above `y`; a **left rotation** does the reverse. Only one piece actually moves: the middle subtree `B`, which detaches from `x` and re-attaches to `y`.

```
       y                        x
      / \      right rot       / \
     x   C     -- at y -->    A   y
    / \                          / \
   A   B       <-- at x --      B   C
                left rot

  read left to right (in-order), both trees give:  A  x  B  y  C
  B simply hops from x's right side to y's left side; A and C never move
```

Concrete instance — here x = 4, y = 8, the moving middle subtree is B = {6}, and C = {9}. Node 8 is unbalanced: its left side is two levels deep while its right side is empty.

```
        8
       / \
      4   9
     / \
    2   6
   /
  1
```

A right rotation at 8 lifts node 4 to the top and re-parents node 6 from 4's right child to 8's left child:

```
      4
     / \
    2   8
   /   / \
  1   6   9

  in-order:  1 2 4 6 8 9    (unchanged before and after)
  height:    3  ->  2       (the long path 8-4-2-1 is gone)
```

The keys still read `1 2 4 6 8 9`, so the tree is still a correct BST — but the longest root-to-leaf path shrank from 3 edges to 2, so lookups are faster. That is the whole point: a rotation preserves the order (correctness) while reducing the height (speed).

In the worst case — inserting already-sorted keys builds a one-sided chain of height O(n), no better than a linked list — repeated rotations restore height O(log n). AVL and red-black trees fire them automatically after any insert or delete that unbalances a node.

**Red-Black Tree** — a BST whose nodes are colored red or black under invariants that keep the longest root-leaf path at most twice the shortest:

- Root and the (NIL) leaves are black.
- A red node has no red child (no two reds in a row).
- Every root-to-leaf path crosses the same number of black nodes (its **black-height**).

This bounds height to ≤ 2·log₂(n+1). An insert or delete needs at most 2–3 rotations plus recoloring (O(1) structural changes amortized), so updates are cheaper than AVL. It is the default general-purpose ordered map. Used in: Java `TreeMap`/`TreeSet`, C++ `std::map`/`std::set`, the Linux kernel (virtual-memory areas, epoll).

**AVL Tree** — a BST keeping a **balance factor** (right height − left height) in {−1, 0, +1} at every node, restored by 1–2 rotations after each update. More rigidly balanced than red-black (height ≤ 1.44·log₂n), so **lookups are faster**, at the cost of **more rotations per insert/delete**. Choose it for lookup-heavy, update-light workloads (an in-memory index built once and queried often).

**B-Tree / B+Tree** — a balanced *m-ary* tree: each node holds up to `m−1` keys and `m` children (its **fan-out**), so the tree is wide and shallow — height `log_m n`. Sizing a node to a disk block or cache line minimizes the number of I/Os per lookup.

- **B-Tree** stores keys (and their values) in both internal and leaf nodes.
- **B+Tree** keeps all values in the **leaves**, with internal nodes holding only routing keys, and links the leaves in key order — making **range scans and ordered iteration** efficient.

Used in: virtually every relational-database index, key-value stores, and filesystems (NTFS, ext4, Btrfs). Detailed mechanics in `Database Internals.md` section 6; storage-engine trade-offs (B-tree vs LSM) in `Data Systems.md` section 4.

**Others (for completeness):**

- **Treap** — BST keyed on values, heap-ordered on random priorities; expected O(log n) with very simple code.
- **Splay tree** — moves each accessed node to the root; O(log n) *amortized*, excellent under skewed access (caches).
- **Scapegoat / weight-balanced** — rebuild a subtree when it grows too lopsided; no per-node balance bookkeeping.
- **Heap** — a complete binary tree with the parent ≤/≥ children property (not an ordered-search tree); see section 7.
- **Trie (prefix tree)** — a tree where each root-to-node path spells a string; see section 8.

| Tree | Balance guarantee | Lookups | Updates | Best for |
|---|---|---|---|---|
| Red-Black | height ≤ 2 log n | fast | few rotations | general ordered map, write-heavy |
| AVL | height ≤ 1.44 log n | fastest | more rotations | read-heavy in-memory index |
| B+Tree | height ≈ log_m n | fast (few I/Os) | splits/merges in blocks | on-disk / large indexes, range scans |

The balanced search trees above guarantee **O(log n)** insert, search, and delete in the worst case — versus a plain BST's O(n) on adversarial input.

### Tree Patterns

**Recursive structure:** most tree algorithms are naturally recursive.

```python
def tree_sum(node):
    if not node: return 0
    return node.value + tree_sum(node.left) + tree_sum(node.right)
```

**Path problems:** typically DFS, accumulating path state.

**Subtree problems:** typically post-order (process children first, then root).

---

## 7. Heaps and Priority Queues

A heap is a complete binary tree satisfying the heap property: every node has priority over its children (min-heap: parent ≤ both children; max-heap: parent ≥ both children). The root is therefore always the minimum (or maximum) element.

### Operations

```
peek (top)              O(1)
push                    O(log n)
pop                     O(log n)
build from n items      O(n)
```

### Binary Heap

A binary heap is a heap realized as a complete binary tree stored implicitly in an array, with parent and child links given by index arithmetic rather than pointers:

```
For node at index i:
  parent = (i - 1) / 2
  left child = 2i + 1
  right child = 2i + 2

Array:    [1, 3, 5, 7, 9, 8, 6]
Tree:           1
              /   \
             3     5
            / \   / \
           7   9 8   6
```

### Push and Pop operations

**Push:** add to end, sift up (swap with parent while smaller).

**Pop:** remove root, move last element to root, sift down (swap with smaller child).

### Used For

- **Priority queues** (Dijkstra, A*, scheduler).
- **Top-K:** find k largest in n items in O(n log k).
- **Heap sort:** O(n log n) in-place sort.
- **Event loops:** earliest-deadline-first scheduling.

### Other Heap Variants

- **Fibonacci heap:** better amortized bounds for Dijkstra theoretically, slower in practice.
- **D-ary heap:** more children per node, fewer levels.
- **Pairing heap, leftist heap:** specialized.

For 95% of use cases, binary heap is the answer.

---

## 8. Tries

A trie (prefix tree) is a tree in which each edge is labelled with a character, so the path from the root to any node spells a string prefix; nodes marked terminal denote complete stored keys. Lookup cost depends on the key length, not on the number of keys stored.

```
Insert "cat", "car", "cap":

           root
            |
           'c'
            |
           'a'
          / | \
        't' 'r' 'p'
         *   *   *      (* = terminal)
```

### Operations

| Operation | Cost |
|-----------|------|
| Insert | O(m) where m = string length |
| Search | O(m) |
| Prefix search | O(m + matches) |

Independent of total number of strings!

### Used For

- **Autocomplete** (find all words with prefix).
- **Spell checkers.**
- **IP routing** (binary trie / radix tree on IP bits).
- **String dictionaries** with prefix queries.

### Variants

**Radix tree / patricia trie:** compresses chains of single-child nodes into single nodes. More memory-efficient.

**DAWG (Directed Acyclic Word Graph):** further compresses by merging shared suffixes.

**Suffix tree / suffix array:** both built in O(n) over a text of length n. A suffix *tree* then locates a pattern of length m with all its occurrences in O(m + matches). A suffix *array* (a sorted array of suffix start positions) is more memory-compact but queries in O(m log n) by binary search, improved to O(m + log n) when paired with an LCP (longest-common-prefix) array. Used in bioinformatics, search.

---

## 9. Disjoint Sets (Union-Find)

A disjoint-set (union-find) structure maintains a partition of n elements into disjoint sets, each identified by a representative element, supporting:

- `find(x)`: which set does x belong to?
- `union(x, y)`: merge x's and y's sets.

```
With union by rank + path compression:
  Both operations: O(α(n)) — effectively constant.
  (α is the inverse Ackermann function, < 4 for any conceivable n.)
```

### How It Works

Each set is represented as a tree, and the **root** of that tree is the set's representative. Every element stores only a pointer to its parent; a root points to itself. So the whole forest is just a `parent` array — `find(x)` walks parent pointers up to the root, and two elements share a set exactly when they reach the same root.

Start with every element in its own singleton set (each is its own root):

```
elements:  0   1   2   3   4   5
parent:   [0,  1,  2,  3,  4,  5]     every node points to itself
```

`union(0,1)` then `union(2,3)` — each merge hangs one root under another root:

```
   0      2      4   5         parent: [0, 0, 2, 2, 4, 5]
   |      |
   1      3
```

`union(0,2)` joins the two trees (the shorter one is hung under the taller root):

```
      0                        parent: [0, 0, 0, 2, 4, 5]
     / \
    1   2
        |
        3
```

`find(3)` follows parent pointers upward until it reaches a node that points to itself (the root): `parent[3]=2`, `parent[2]=0`, `parent[0]=0` — so the root is 0.

**Path compression:** on the way back, `find` re-points each node it visited directly at the root. Node 3 used to point at 2; now it points straight at 0. Only `parent[3]` changes (2 -> 0), and the tree gets flatter:

```
   before find(3)            after find(3)

        0                        0
       / \                     / | \
      1   2                   1  2  3      <- 3 now hangs off the root
          |
          3

   parent: [0,0,0,2,4,5]     parent: [0,0,0,0,4,5]
```

The set is unchanged — still {0,1,2,3} with root 0 — but a later `find(3)` now reaches the root in one hop instead of two. Repeated finds keep flattening the tree, so it stays nearly flat and lookups stay near O(1).

Two tricks keep the trees flat, which is what makes the operations near-constant:

- **Union by rank** — attach the shorter tree under the taller one's root (`rank` tracks an upper bound on height), so repeated unions never build a tall chain.
- **Path compression** — on every `find`, point each node on the path directly to the root, flattening it for future queries.

Together they give the O(α(n)) amortized bound noted above.

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # path compression
        return self.parent[x]

    def union(self, x, y):
        px, py = self.find(x), self.find(y)
        if px == py: return False
        # union by rank: attach shorter tree under taller
        if self.rank[px] < self.rank[py]:
            px, py = py, px
        self.parent[py] = px
        if self.rank[px] == self.rank[py]:
            self.rank[px] += 1
        return True
```

### Used For

- **Kruskal's MST algorithm.**
- **Connected components** in dynamic graphs.
- **Cycle detection** in undirected graphs.
- **Network connectivity, percolation.**
- **Image segmentation** (group connected pixels).

---

## 10. Graphs

A graph G = (V, E) is a set of vertices V connected by a set of edges E. It is the most general data structure: trees, lists, and grids are all special cases. Edges may be **directed** (an ordered pair u -> v, modelling one-way relations) or **undirected** (an unordered pair, modelling symmetric relations), and may carry a **weight** (a cost, distance, or capacity).

**Notation used throughout this section:**

- **V** — the number of vertices (also written |V|); **E** — the number of edges (|E|).
- **u, v** — individual vertices; an edge is written (u, v), directed from u (tail) to v (head).
- **neighbors of u** — the vertices reachable from u by one edge; its **out-neighbors** if the graph is directed.
- **deg(u)** — the degree of u: the number of edges incident to it. For a directed graph this splits into **in-degree** (edges into u) and **out-degree** (edges out of u). The sum of all degrees is 2E (undirected) or E (counting out-degrees, directed).
- A graph is **sparse** when E is far below its maximum V^2 (most real graphs), and **dense** when E approaches V^2.

### Representations

All three below describe the same directed graph, used as the running example:

```
      A           edges:  A -> B
     / \                  A -> C
    B   C                 B -> D
     \ /                  C -> D
      D
```

**Adjacency list:** for each vertex, a list (or set) of its neighbors.

```python
graph = {
    'A': ['B', 'C'],
    'B': ['D'],
    'C': ['D'],
    'D': []
}
```

Space: O(V + E). Iterating a vertex's neighbors is O(deg). Testing whether a specific edge (u, v) exists is O(deg(u)) unless the per-vertex container is a hash set. The default choice for most algorithms because real graphs are sparse (E far below V^2) and traversals (BFS/DFS) only ever ask for neighbors.

**Adjacency matrix:** V × V matrix; entry [u][v] holds 1 (or the edge weight) if the
edge exists, else a "no edge" sentinel. For an *unweighted* graph that sentinel is 0,
but for a *weighted* graph 0 is a legal weight, so use +inf (or None) to mark absence.
For an undirected graph the matrix is symmetric.

```python
#         A  B  C  D
graph = [[0, 1, 1, 0],   # A
         [0, 0, 0, 1],   # B
         [0, 0, 0, 1],   # C
         [0, 0, 0, 0]]   # D
```

Space: O(V^2). Edge existence and weight lookup are O(1), but listing a vertex's neighbors is O(V) (scan a whole row), and the matrix wastes space on the absent edges. Best for dense graphs, or when the algorithm repeatedly probes "is (u, v) an edge?" (e.g. Floyd-Warshall, transitive closure).

**Edge list:** a flat list of (u, v, weight) tuples.

```python
edges = [('A', 'B', 1), ('A', 'C', 1), ('B', 'D', 1), ('C', 'D', 1)]
```

Space: O(E). No O(1) neighbor or edge-existence query, but the format is ideal when an algorithm processes every edge as a unit, especially after sorting them: Kruskal's MST (sort by weight, union-find), Bellman-Ford (relax all edges V-1 times).

**Trade-offs:**

| operation              | adjacency list | adjacency matrix | edge list |
|------------------------|----------------|------------------|-----------|
| space                  | O(V + E)       | O(V^2)           | O(E)      |
| edge (u, v) exists?    | O(deg u)*      | O(1)             | O(E)      |
| iterate neighbors of u | O(deg u)       | O(V)             | O(E)      |
| add edge               | O(1)           | O(1)             | O(1)      |
| best for               | sparse, traversal | dense, edge probes | edge-batch algos |

*O(1) if each vertex stores neighbors in a hash set instead of a list.

### Sparse Matrices

An adjacency matrix for a sparse graph is almost all zeros: V^2 cells holding only E nonzeros. Storing it densely is wasteful, but O(1) edge lookup is still attractive. **Sparse matrix formats** keep only the nonzero entries plus enough indexing to find them, trading some access speed for O(E) space. The same formats appear throughout scientific computing (finite-element meshes, PageRank, sparse linear solvers), where the "graph" is the nonzero pattern of a huge matrix.

Take a short, wide matrix — 3 rows, 8 columns, only 5 nonzeros. Stored densely it needs 3 x 8 = 24 cells; the formats below keep just the 5 nonzeros:

```
         col:  0  1  2  3  4  5  6  7
  row 0:     [ 0  5  0  0  8  0  0  0 ]
  row 1:     [ 0  0  0  0  0  0  0  0 ]
  row 2:     [ 7  0  0  3  0  0  0  9 ]
```

**COO (coordinate list / triplet):** three parallel arrays — row, column, value — one triple per nonzero, in row order. Trivial to build and append to, but lookups and row slicing are unordered. (For a graph's adjacency matrix, these triples are exactly its weighted edge list.)

```
row = [0, 0, 2, 2, 2]
col = [1, 4, 0, 3, 7]
val = [5, 8, 7, 3, 9]
```

**CSR (compressed sparse row):** the workhorse format. The COO `row` array is wasteful — it only marks where each row's entries start. CSR drops it and keeps just `col` and `val` (the nonzeros read left-to-right, top-to-bottom) plus a small `row_ptr`.

`row_ptr[i]` says where row i's entries begin inside `col`/`val`. Build it by counting the nonzeros in each row and keeping a running total (a prefix sum — a cumulative running sum), starting at 0:

```
  row     nonzeros   running total -> row_ptr
  row 0      2       row_ptr[0] = 0
  row 1      0       row_ptr[1] = 0 + 2 = 2
  row 2      3       row_ptr[2] = 2 + 0 = 2    <- empty row 1 repeats the value
                     row_ptr[3] = 2 + 3 = 5    (= total nonzeros)

  row_ptr = [0, 2, 2, 5]      length = rows + 1 = 4   (independent of the 8 columns)
  col     = [1, 4, 0, 3, 7]   column of each nonzero, read row by row
  val     = [5, 8, 7, 3, 9]
```

To get the entries of row i, slice `col`/`val` from `row_ptr[i]` up to `row_ptr[i+1]`:

```
  row 0:  col[0:2] = [1, 4]      values [5, 8]
  row 1:  col[2:2] = []          empty (start == end)
  row 2:  col[2:5] = [0, 3, 7]   values [7, 3, 9]
```

That is the win: CSR stores `col` + `val` (one slot per nonzero) plus a `row_ptr` of only `rows + 1` slots — and `row_ptr`'s size depends on the row count, **not** the column count. Here that is 14 numbers instead of 24 dense cells; widen the matrix to 3 x 1000 with the same 5 nonzeros and dense storage explodes to 3000 cells while CSR stays at 14. Neighbor iteration and matrix-vector products are fast and cache-friendly because everything sits in contiguous arrays; the structure is fixed once built.

**CSC (compressed sparse column):** the transpose of CSR, compressing columns instead of rows. Efficient for column slices — in graph terms, listing the *in*-neighbors (predecessors) of a vertex.

**DOK / LIL (dictionary-of-keys, list-of-lists):** mutable build formats. DOK maps (row, col) -> value in a hash table (O(1) random insert and lookup); LIL keeps a sorted per-row list. Both are convenient while incrementally constructing a matrix, then converted to CSR/CSC for fast read-only computation.

| format | space | random lookup | iterate native axis | mutable | role          |
|--------|-------|---------------|---------------------|---------|---------------|
| COO    | O(E)  | O(E)          | O(E)                | append  | build, exchange |
| CSR    | O(E)  | O(log deg)    | O(deg) per row      | no      | compute (row-wise) |
| CSC    | O(E)  | O(log deg)    | O(deg) per column   | no      | compute (col-wise) |
| DOK/LIL| O(E)  | O(1) / O(deg) | O(E)                | yes     | incremental build |

The adjacency list is, in effect, CSR with per-vertex pointer chasing instead of flat arrays; CSR is the cache-friendly, fixed-size counterpart used when the graph never changes after construction.

### BFS (Breadth-First Search)

Breadth-first search explores a graph from a source vertex in order of increasing distance: it visits every vertex at distance 1, then every vertex at distance 2, and so on, using a FIFO queue to hold the frontier. On unweighted graphs this yields shortest-path distances.

```python
def bfs(graph, start):
    visited = {start}
    queue = deque([start])
    while queue:
        node = queue.popleft()
        process(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
```

**Properties:**
- Finds shortest path in unweighted graphs.
- O(V + E) time.

### DFS (Depth-First Search)

Depth-first search explores a graph by following each branch as far as possible before backtracking to the most recent vertex with unexplored neighbors, using recursion or an explicit stack.

```python
def dfs(graph, start, visited=None):
    if visited is None: visited = set()
    visited.add(start)
    process(start)
    for neighbor in graph[start]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
```

**Used for:**
- Cycle detection.
- Topological sort.
- Connected components.
- Maze solving.
- Tarjan's strongly-connected-components (SCC) algorithm.

### Topological Sort

For a **DAG** (directed acyclic graph — a directed graph with no cycles), a linear ordering of the vertices in which every edge u -> v places u before v. It exists if and only if the graph is acyclic.

**Kahn's algorithm (BFS-based):** repeatedly remove vertices with in-degree 0 (no remaining prerequisites).

```python
def topo_sort(graph):
    in_degree = {node: 0 for node in graph}
    for node in graph:
        for neighbor in graph[node]:
            in_degree[neighbor] += 1

    queue = deque([node for node, deg in in_degree.items() if deg == 0])
    result = []
    while queue:
        node = queue.popleft()
        result.append(node)
        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    if len(result) != len(graph):
        raise ValueError("Graph has a cycle")
    return result
```

Used in: build systems, dependency resolution, course scheduling.

### Shortest Path

Given a graph with edge weights, a shortest path between two vertices is a path of minimum total weight; these algorithms compute such paths (or their lengths) from a source, or between all pairs.

**BFS:** computes shortest paths on unweighted graphs (every edge counts as 1). O(V + E).

**Dijkstra:** single-source shortest paths on graphs with non-negative weights; it repeatedly settles the nearest unsettled vertex, relaxing its outgoing edges. O((V + E) log V) with a binary heap. Negative edges break it: settling a vertex assumes no cheaper path can later appear, but a negative edge can reduce a distance after the vertex is already finalized — hence Bellman-Ford for negative weights.

```python
def dijkstra(graph, start):
    dist = {node: float('inf') for node in graph}
    dist[start] = 0
    pq = [(0, start)]
    while pq:
        d, node = heappop(pq)
        if d > dist[node]: continue
        for neighbor, weight in graph[node]:
            nd = d + weight
            if nd < dist[neighbor]:
                dist[neighbor] = nd
                heappush(pq, (nd, neighbor))
    return dist
```

**Bellman-Ford:** single-source shortest paths that also handles negative edge weights by relaxing all E edges V−1 times; a further relaxation that still improves a distance proves a reachable negative cycle. O(V * E).

**Floyd-Warshall:** all-pairs shortest paths via dynamic programming, progressively allowing each vertex as an intermediate. O(V^3).

**A\*:** single-source-to-target search that orders Dijkstra's frontier by `distance-so-far + heuristic(estimate to target)`; an **admissible** heuristic (never overestimates) guarantees an optimal path, and a **consistent** one (obeys the triangle inequality) additionally guarantees no vertex is expanded twice. Explores far fewer vertices than Dijkstra.

### Minimum Spanning Tree (MST)

A minimum spanning tree of a connected, weighted, undirected graph is a subset of edges that connects all V vertices (forming a tree of V−1 edges) with minimum total weight.

**Kruskal's:** sort edges, add if they don't form a cycle (use Union-Find). O(E log E).

**Prim's:** grow tree from a starting vertex, picking the cheapest connecting edge. O((V + E) log V) with heap.

### Connected Components

Find all maximal connected subgraphs. DFS or BFS over the whole graph, marking which component each vertex belongs to. O(V + E).

### Strongly Connected Components (Directed)

Tarjan's or Kosaraju's algorithm. O(V + E). A strongly connected component is a maximal subgraph where every vertex can reach every other.

---

## 11. Sorting

### Comparison Sorts

| Algorithm | Avg | Worst | Space | Stable | Notes |
|-----------|-----|-------|-------|--------|-------|
| Bubble | n^2 | n^2 | 1 | yes | educational only |
| Insertion | n^2 | n^2 | 1 | yes | best for tiny / nearly-sorted |
| Selection | n^2 | n^2 | 1 | no | educational only |
| Merge | n log n | n log n | n | yes | predictable, parallelizable |
| Quick | n log n | n^2 | log n* | no | fast in practice, in-place |
| Heap | n log n | n log n | 1 | no | guaranteed n log n, in-place |
| Tim | n log n | n log n | n | yes | Python and Java default |

*Quicksort's O(log n) space assumes recursing into the smaller partition first (bounding stack depth); a naive recursion on already-sorted input degrades to O(n) stack.

**Stable** means the sort preserves the relative order of elements that compare equal: if two items have the same key, the one that came first in the input stays first in the output. This matters for multi-key sorting — to order by age, then by name within an age, sort by name first and then run a *stable* sort by age, and the name order survives among equal ages.

**Comparison-sort lower bound:** Ω(n log n). No comparison sort can do better.

### Quicksort

Quicksort sorts an array in place by choosing a pivot element, partitioning the array so that elements ≤ pivot come before it and elements > pivot after it, then recursively sorting each partition. Average O(n log n); worst O(n^2) on bad pivots.

```python
def quicksort(arr, lo=0, hi=None):
    if hi is None: hi = len(arr) - 1
    if lo < hi:
        p = partition(arr, lo, hi)
        quicksort(arr, lo, p - 1)
        quicksort(arr, p + 1, hi)

def partition(arr, lo, hi):
    pivot = arr[hi]
    i = lo - 1
    for j in range(lo, hi):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    arr[i+1], arr[hi] = arr[hi], arr[i+1]
    return i + 1
```

Pivot choice matters: median-of-three, randomized, or introsort (switch to heap sort if recursion gets deep).

**Selection (k-th smallest).** The same partition step finds the k-th order statistic without fully sorting: **Quickselect** recurses into only the side containing rank k, giving O(n) average (but O(n^2) worst-case) time; **median-of-medians** chooses a provably good pivot to guarantee O(n) worst-case. Both are the partition-based analogue of sorting then indexing.

### Merge Sort

Merge sort sorts by recursively splitting the array into two halves until single elements remain, then merging adjacent sorted runs into larger sorted runs. Stable, O(n log n) in all cases, O(n) extra space.

```
Sort:   [3, 7, 1, 4, 6, 2, 8, 5]
Split:  [3, 7, 1, 4]      [6, 2, 8, 5]
        [3, 7] [1, 4]      [6, 2] [8, 5]
        [3] [7] [1] [4]    [6] [2] [8] [5]
Merge:  [3, 7] [1, 4]      [2, 6] [5, 8]
        [1, 3, 4, 7]       [2, 5, 6, 8]
        [1, 2, 3, 4, 5, 6, 7, 8]
```

Stable, predictable, parallelizable. Uses O(n) extra space.

### Tim Sort

Adaptive merge sort. Finds existing runs, merges them. Excellent on partially-sorted data (very common in practice). Python's `sorted()` and Java's `Arrays.sort` for objects.

### Non-Comparison Sorts (Linear Time)

When data fits a special structure:

**Counting sort:** for integers in known range R. O(n + R) time and space.

**Radix sort:** sort by digit/byte at a time. O(d(n + R)) where d = number of digits.

**Bucket sort:** distribute into buckets, sort each.

Useful when sorting fixed-width keys or integers in a bounded range.

---

## 12. Searching

### Linear Search

Linear search scans the elements one by one, returning the first index whose element equals the target, or reporting absence after the full pass. O(n); works on unsorted data.

It is the right tool more often than its reputation suggests: it needs no preprocessing, touches memory sequentially (cache-friendly), and beats fancier methods on small or one-shot inputs. Reach for something better only when the same collection is searched repeatedly (then sort once and binary-search, or build a hash table / tree).

### Binary Search

Binary search locates a target in a sorted array by repeatedly halving the search interval: it compares the middle element and continues in the half that may still contain the target. Each comparison discards half the remaining elements, so it finishes in O(log n) comparisons.

```
  index:   0   1   2   3   4   5   6
  arr:     1   3   5   7   9  11  13     target = 9

  step 1: [1   3   5   7   9  11  13]    mid=3, arr[3]=7  < 9  -> drop left half
  step 2:                 [9  11  13]    mid=5, arr[5]=11 > 9  -> drop right half
  step 3:                 [9]            mid=4, arr[4]=9  = 9  -> found at index 4
```

```python
def binary_search(arr, target):
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = (lo + hi) // 2          # In some langs: lo + (hi - lo) // 2 to avoid overflow
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1
```

**Preconditions and pitfalls:**

- The array must be **sorted** on the same key you compare against; otherwise the result is meaningless.
- Fix a loop invariant and hold it: "if the target exists, it lies within the current window." Decide once whether the window is closed `[lo, hi]` or half-open `[lo, hi)` and stay consistent — mixing the two is the classic source of off-by-one bugs and infinite loops.
- Use `mid = lo + (hi - lo) // 2` in fixed-width integer languages; `(lo + hi)` can overflow.
- Cost is O(log n) *comparisons*, but each comparison is on a whole key — for strings that is O(L log n) character comparisons.

**Variants** (all O(log n)):

- **First / last occurrence** of a target that repeats.
- **lower_bound** — smallest index with `arr[i] >= target`.
- **upper_bound** — smallest index with `arr[i] > target`.

These are the building blocks for counting duplicates (`upper_bound - lower_bound`) and for range queries. `lower_bound` is just binary search that, instead of stopping on equality, keeps moving left to find the boundary:

```python
def lower_bound(arr, target):     # smallest i with arr[i] >= target
    lo, hi = 0, len(arr)          # half-open [lo, hi); hi can be len(arr)
    while lo < hi:
        mid = lo + (hi - lo) // 2
        if arr[mid] < target:
            lo = mid + 1          # mid too small, boundary is to the right
        else:
            hi = mid              # mid may be the answer, keep it in range
    return lo                     # in [0, len]; equals len if every element < target
```

The standard library usually has these (`bisect` in Python, `lower_bound`/`upper_bound` in C++). Don't reimplement unless necessary.

### Binary Search on Answer

Binary search needs no array — the sorted array is just its most common use, not a requirement. All it fundamentally needs is an ordered range to search and a **monotonic predicate** over that range of candidate answers. The "array" can be implicit: the integers from `lo` to `hi`, with a feasibility test playing the role that `arr[mid] == target` plays in the classic version. "Monotonic" means: once some value X is feasible, every larger value is feasible too (or every smaller one). The feasibility results then form a sorted `F...FT...T` pattern, and you binary-search for the boundary — the smallest feasible X — over the *value range*, not over stored data.

```
  capacity:   9   10   11   12   13   14   15
  can_ship:   F   F    F    T    T    T    T
                            ^ smallest feasible capacity = the answer
```

Cost is O(log(range) * C), where `range = hi - lo` is the span of candidate answers and `C` is the cost of one feasibility check. The win is that checking "is X feasible?" is often far easier than directly computing the optimum.

```python
# Find the smallest capacity that can ship all packages in D days.
def min_capacity(weights, D):
    def can_ship(cap):
        days = 1
        load = 0
        for w in weights:
            if load + w > cap:
                days += 1
                load = 0
            load += w
        return days <= D

    lo = max(weights)
    hi = sum(weights)
    while lo < hi:
        mid = (lo + hi) // 2
        if can_ship(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo
```

A few insights worth holding onto:

- **Monotonicity is what makes it legal.** A larger capacity can never break a plan that already worked, so feasibility flips exactly once — that is the `F...FT...T` the search depends on.
- **The bounds are the extremes of the answer.** `lo = max(weights)` is the smallest capacity that can hold any single package; `hi = sum(weights)` ships everything in one day. The answer must lie between them, so they safely bracket the search.
- **It is just `lower_bound` over a value range.** The `hi = mid` / `lo = mid + 1` shape finds the first `T`; only the "array" (an implicit range of values) and the comparison (a feasibility call) differ.

The tell in a problem statement is *"minimize the maximum,"* *"maximize the minimum,"* or *"smallest X such that..."* — the optimum is hard to compute directly, but checking a fixed candidate is easy.

### Other Search Strategies

- **Interpolation search** — for sorted, roughly uniform numeric keys, guess the probe position proportionally to the target's value (how you open a phone book near "S", not in the middle) instead of always probing the midpoint. O(log log n) average on uniform data, but degrades to O(n) on skewed distributions.
- **Exponential (galloping) search** — for sorted input of unknown or unbounded length: probe indices 1, 2, 4, 8, ... until you overshoot the target, then binary-search the last bracket. O(log i) where i is the target's position — excellent when the target is near the front.
- **Ternary search** — for a **unimodal** function (rises then falls, or vice versa): probe two interior points and discard the outer third that cannot hold the extremum, converging on the peak/valley in O(log n) evaluations. This finds an optimum, not array membership.

### Substring Search

Locating a pattern P of length m inside a text T of length n. Naive scanning is O(nm). Two standard linear-time methods:

- **Knuth-Morris-Pratt (KMP)** — precomputes, for each prefix of P, the length of its longest proper prefix that is also a suffix (the "failure function"), so on a mismatch the pattern shifts without re-examining text already matched. O(n + m), with no backtracking over T.
- **Rabin-Karp** — compares a rolling hash of each length-m text window against the hash of P, verifying character-by-character only on a hash match. O(n + m) expected; the rolling hash updates in O(1) per shift. Extends naturally to multi-pattern and 2D search.

Boyer-Moore (and its Boyer-Moore-Horspool variant) skips ahead using bad-character and good-suffix rules, achieving sublinear average time on large alphabets; it is the basis of most `grep`/`memmem` implementations.

**Search is also a property of the data structure**, covered in their own sections: hash table O(1) average (section 5), balanced BST / tree O(log n) ordered lookup (section 6), trie prefix search in O(key length) (section 8), and BFS/DFS over graphs (section 10). Choosing the structure up front is usually a bigger win than optimizing the search algorithm.

---

## 13. Recursion, Backtracking, Memoization

### Recursion

A function calling itself with a smaller subproblem.

```python
def factorial(n):
    if n <= 1: return 1
    return n * factorial(n - 1)
```

**Tail recursion:** the recursive call is the last operation. Some compilers optimize to iteration. (Python doesn't.)

**Stack overflow:** each call costs stack frame; deep recursion crashes. Convert to iteration for very deep recursion.

### Backtracking

Backtracking is a systematic exhaustive search that builds a candidate solution incrementally and abandons a partial candidate (backtracks) as soon as it cannot possibly be extended to a valid solution — pruning whole branches of the search space.

```python
# N-Queens: place N queens on N×N board with no attacks.
def solve_nqueens(n):
    result = []
    def backtrack(row, cols, diag1, diag2, placement):
        if row == n:
            result.append(placement[:])
            return
        for col in range(n):
            if col in cols or row+col in diag1 or row-col in diag2:
                continue
            placement.append(col)
            cols.add(col); diag1.add(row+col); diag2.add(row-col)
            backtrack(row+1, cols, diag1, diag2, placement)
            placement.pop()
            cols.remove(col); diag1.remove(row+col); diag2.remove(row-col)
    backtrack(0, set(), set(), set(), [])
    return result
```

Standard backtracking template:

```
def backtrack(state):
    if is_solution(state):
        record(state)
        return
    for choice in choices(state):
        if is_valid(choice, state):
            apply(choice, state)
            backtrack(state)
            undo(choice, state)
```

### Memoization

Cache function results to avoid recomputation. Turns exponential recursive solutions into polynomial.

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    if n < 2: return n
    return fib(n-1) + fib(n-2)
```

Without memoization: O(2^n). With: O(n). Same code.

Memoization is top-down DP.

---

## 14. Dynamic Programming

Solve a problem by combining solutions to smaller, overlapping subproblems.

### When DP Applies

- **Optimal substructure:** the optimal solution decomposes into optimal subproblem solutions.
- **Overlapping subproblems:** the same subproblems are solved multiple times.

### Top-Down vs Bottom-Up

**Top-down (memoization):** recurse + cache.

```python
@lru_cache
def lcs(a, b, i, j):
    if i == 0 or j == 0: return 0
    if a[i-1] == b[j-1]:
        return lcs(a, b, i-1, j-1) + 1
    return max(lcs(a, b, i-1, j), lcs(a, b, i, j-1))
```

**Bottom-up (tabulation):** fill a table iteratively.

```python
def lcs(a, b):
    n, m = len(a), len(b)
    dp = [[0] * (m+1) for _ in range(n+1)]
    for i in range(1, n+1):
        for j in range(1, m+1):
            if a[i-1] == b[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    return dp[n][m]
```

Bottom-up tends to be faster (no recursion overhead), top-down is easier to write.

### Classic DP Problems

```
Fibonacci                       O(n)
Longest common subsequence      O(nm)
Edit distance                   O(nm)
Knapsack (0/1)                  O(nW) where W = capacity
Coin change                     O(nW)
Longest increasing subsequence  O(n log n) with patience sorting
Matrix chain multiplication     O(n^3)
Optimal BST                     O(n^3)   (O(n^2) with Knuth's optimization)
Traveling salesman (Held-Karp)  O(2^n * n^2)
```

The knapsack and coin-change bounds are **pseudo-polynomial**: O(nW) is polynomial in the numeric *value* of the capacity W, but exponential in its *input size*, since W occupies only log W bits. Doubling the bit-length of W doubles the table dimension rather than adding a constant factor. This is why 0/1 knapsack is NP-hard despite a table-shaped solution — the table is polynomial only when W is bounded by a polynomial in n. Treat any complexity that scales with a numeric magnitude rather than an element count as pseudo-polynomial.

### Space Optimization

Often DP tables can be reduced. LCS needs only the previous row:

```python
def lcs(a, b):
    prev = [0] * (len(b)+1)
    for i in range(1, len(a)+1):
        curr = [0] * (len(b)+1)
        for j in range(1, len(b)+1):
            curr[j] = prev[j-1] + 1 if a[i-1] == b[j-1] else max(prev[j], curr[j-1])
        prev = curr
    return prev[-1]
```

O(min(n, m)) space.

---

## 15. Greedy Algorithms

A greedy algorithm builds a solution one step at a time, at each step taking the choice that looks best right now and never reconsidering it. It commits irrevocably; unlike DP or backtracking, it never revisits a decision.

### When Greedy Works

Two properties must both hold:

- **Greedy-choice property:** some locally optimal choice is part of *a* globally optimal solution — so committing to it never rules out the best answer.
- **Optimal substructure:** after making that choice, what remains is a smaller instance of the same problem, and an optimal solution to the whole contains optimal solutions to its subproblems.

The standard way to prove a greedy is correct is the **exchange argument**: take any optimal solution, show that swapping its first differing choice for the greedy choice leaves it no worse, and conclude by induction that the all-greedy solution is optimal. If you cannot run this argument, the greedy is probably wrong.

### Examples

**Interval scheduling** (most non-overlapping intervals): sort by *end* time, take each interval that starts after the last one taken finishes. Earliest-finishing leaves the most room for the rest — the exchange argument made concrete.

**Huffman coding:** repeatedly combine the two least-frequent items. The two rarest symbols can always sit deepest in the optimal prefix tree, so merging them first is safe.

**MST (Kruskal's):** add the cheapest edge that forms no cycle. The cut property guarantees the lightest edge crossing any cut belongs to some MST.

**Activity selection** — the scheduling problem above, in its classic framing.

### When Greedy Fails

When a locally optimal choice forecloses the global optimum. Coin change with arbitrary denominations is the canonical case: with coins {1, 3, 4} making 6, greedy takes the largest first — 4 + 1 + 1 = 3 coins — but the optimum is 3 + 3 = 2 coins. The early grab of 4 was locally best yet globally wasteful. (Greedy *does* work for "canonical" coin systems like standard currency, which is why it feels right until it isn't.)

The dividing line: **DP solves general optimization by considering all choices and combining subproblem results; greedy solves the special cases where one choice is provably always safe, trading generality for an O(n log n)-or-better runtime with no memo table.** When unsure, reach for DP — a greedy that fails is silently wrong, not slow.

---

## 16. Divide and Conquer

Split into subproblems, solve recursively, combine.

```
1. Divide the problem into subproblems.
2. Conquer (solve recursively).
3. Combine the solutions.
```

### Examples

- **Merge sort:** divide array, sort halves, merge.
- **Quick sort:** partition, recurse on halves.
- **Binary search:** divide range in half.
- **Strassen matrix multiplication:** O(n^2.81).
- **Karatsuba multiplication:** O(n^1.58) for large integer multiplication.
- **FFT (Fast Fourier Transform):** O(n log n) — foundational for signal processing, polynomial multiplication.

### Master Theorem

For recurrences of the form `T(n) = aT(n/b) + f(n)`:

```
T(n) = a * T(n/b) + O(n^d)

Three cases:
  d > log_b(a):   T(n) = O(n^d)
  d = log_b(a):   T(n) = O(n^d log n)
  d < log_b(a):   T(n) = O(n^(log_b a))
```

Examples:

- Merge sort: `T(n) = 2T(n/2) + O(n)` → log_2(2) = 1 = d → O(n log n).
- Binary search: `T(n) = T(n/2) + O(1)` → log_2(1) = 0 = d → O(log n).
- Strassen: `T(n) = 7T(n/2) + O(n^2)` → log_2(7) ≈ 2.81 > 2 = d → O(n^2.81).

This `O(n^d)` form covers only polynomial f(n). It says nothing when f(n) falls between the cases — for example `f(n) = n log n` against `n^(log_b a) = n` — a gap the full Master Theorem handles with a regularity condition, and Akra-Bazzi handles for uneven splits.

---

## 17. Probabilistic and Approximate Structures

For when exactness costs too much space or time.

### Bloom Filter

A set with no false negatives and tunable false positives. Space-efficient for testing membership.

```
Bit array of m bits.
k hash functions.

Insert x:
  for each hash function h_i:
    set bit h_i(x) % m

Test x:
  if all bits h_i(x) are set:
    "probably present"
  else:
    "definitely not present"
```

Used in: Cassandra (skip SSTable — sorted string table — reads), web caches (skip backend lookups), Chrome (URL blocklist).

**Cost:** O(k) per op, O(m) space. False positive rate ~ (1 - e^(-kn/m))^k.

### Count-Min Sketch

Estimate frequency of items in a stream. Used for top-K queries on infinite streams.

```
2D array of counters indexed by k hash functions.
On insert: increment counters[i][h_i(x)] for each i.
On query: min over all counters[i][h_i(x)].

Always overestimates (never undercounts).
```

### HyperLogLog

Estimate the number of distinct elements in a stream using small, fixed memory — on the order of a kilobyte — with a standard error of roughly 1.04/√m for m registers, so a few-percent error at ~1.5 KB and under 1% at ~12 KB. Works on cardinalities into the billions.

Used in: analytics ("unique visitors today"), database cardinality estimation, Redis PFCOUNT.

### Reservoir Sampling

Sample k items uniformly from a stream of unknown length.

```python
def reservoir_sample(stream, k):
    reservoir = []
    for i, item in enumerate(stream):
        if i < k:
            reservoir.append(item)
        else:
            j = random.randint(0, i)
            if j < k:
                reservoir[j] = item
    return reservoir
```

Used when the whole stream does not fit in memory but a uniform sample is required.

### Skip List

A skip list is a probabilistic ordered structure: a base sorted linked list overlaid with several "express" levels, each a randomly sampled sublist of the one below, so a search drops down through sparse-to-dense levels and skips over most elements. Expected O(log n) search, insert, and delete.

```
Level 3:  ----------------> 30 ----------------> 90
Level 2:  ----> 10 -------> 30 -----> 50 ------> 90
Level 1:  ----> 10 -> 20 -> 30 -> 40 -> 50 -> 60 -> 70 -> 80 -> 90
```

Expected O(log n) operations. Used in Redis sorted sets (ZSET), LevelDB, MemSQL.

### Consistent Hashing

A way to map keys to N servers such that adding/removing a server reshuffles only ~1/N of keys.

```
Hash space as a ring [0, 2^32).
Each server gets multiple positions on the ring.
Each key hashes to a position, belongs to the next server clockwise.
```

Used in: Memcached, Cassandra, Dynamo, every distributed cache or DB that shards.

See `Distributed Systems.md` section 12 for consistent hashing with virtual nodes, and `Data Systems.md` section 8 for its use in sharding.

---

### Choosing the Right Tool

A quick lookup by problem shape:

| Need | Reach for |
|------|----------|
| Fast lookup by key | Hash table |
| Lookup + sorted iteration | Tree (TreeMap, std::map) |
| LRU cache | Hash map + doubly linked list |
| Top-K from stream | Heap of size K |
| Range queries (sum/min) | Prefix sum / segment tree / fenwick tree |
| String prefix search | Trie |
| Connected components | Union-Find |
| Shortest path (non-neg weights) | Dijkstra |
| Shortest path (unweighted) | BFS |
| Topological order | Kahn's algorithm |
| Set membership at scale | Bloom filter |
| Unique count at scale | HyperLogLog |
| Uniform stream sample | Reservoir sampling |
| Sort | Use the language's standard sort |
| Search in sorted | Binary search |

The language's standard library usually has the right data structure. Use it. Reimplementing a hash table or sort in production code is almost always wrong.
