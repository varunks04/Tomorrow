# Core Data Structures: Memory Architecture & Operation Complexities

> **Domain:** Core Computer Science  
> **Sub-Domain:** Data Structures & Algorithms  
> **Interview Importance:** Very High / Foundational CS Technical Competency  

---

## 1. Topic & Definitions

- **Data Structure:** A specialized format and mathematical organization for storing, organizing, processing, and retrieving data in computer memory efficiently.
- **Linear Data Structures:** Elements arranged sequentially in memory (Arrays, Linked Lists, Stacks, Queues).
- **Non-Linear Data Structures:** Elements arranged hierarchically or interconnectedly (Trees, Heaps, Graphs, Tries).

---

## 2. Core Data Structures Deep Dive

```text
+─────────────────────────────────────────────────────────────────────────────+
|                         CORE DATA STRUCTURES TAXONOMY                       |
+─────────────────────────────────────────────────────────────────────────────+
| LINEAR:                                                                     |
|   • Array: Contiguous memory, constant-time index access O(1), fixed size.  |
|   • Linked List: Dynamic nodes connected via pointers, O(1) insert/delete.  |
|   • Stack: LIFO (Last-In, First-Out), push/pop at top.                      |
|   • Queue: FIFO (First-In, First-Out), enqueue at tail, dequeue at head.    |
|─────────────────────────────────────────────────────────────────────────────|
| NON-LINEAR:                                                                 |
|   • Hash Table: Key-value map using hash functions with bucket chaining.   |
|   • Tree / BST: Hierarchical nodes; BST maintains left < root < right.      |
|   • Heap: Complete binary tree satisfying heap invariant (Min-Heap/Max-Heap)|
|   • Graph: Set of vertices connected by directed or undirected edges.       |
|   • Trie: Prefix tree where edges represent characters for fast lookup.     |
+─────────────────────────────────────────────────────────────────────────────+
```

### 1. Arrays vs. Linked Lists (Memory Architecture)
- **Array:** Stored in a single, **contiguous block of physical memory**.
  - *Advantage:* Instantaneous $O(1)$ random access via base address arithmetic: `Address(i) = BaseAddress + (i * ElementSize)`. Maximizes **CPU Cache Locality** (spatial prefetching).
  - *Disadvantage:* Fixed capacity; resizing requires allocating a new memory block and copying all elements ($O(N)$). Insertions/deletions at the beginning require shifting elements ($O(N)$).
- **Linked List:** Nodes scattered non-contiguously throughout the Heap, connected by forward (and backward in Doubly Linked Lists) memory pointers.
  - *Advantage:* Truly dynamic size; $O(1)$ insertion and deletion once pointer is located.
  - *Disadvantage:* $O(N)$ sequential search time; pointer memory overhead (8 bytes per pointer on 64-bit systems); terrible CPU cache locality (cache misses on pointer hops).

---

### 2. Stacks and Queues
- **Stack (LIFO):** Used in call stack management, undo buffers, syntax parsing, and depth-first search (DFS). Operations `push()` and `pop()` are strictly $O(1)$.
- **Queue (FIFO):** Used in CPU scheduling ready queues, printer spoolers, packet buffers, and breadth-first search (BFS). Operations `enqueue()` and `dequeue()` are strictly $O(1)$.
- **Priority Queue (Heap):** Dequeues the element with the highest/lowest priority in $O(\log N)$ time, implemented via a Binary Heap.

---

### 3. Hash Tables & Collision Resolution
- **Hash Table:** Uses a **Hash Function** to map an arbitrary key (string, integer) into an integer array index (bucket).
- **Collision Resolution Strategies:**
  1. **Separate Chaining (Open Hashing):** Each bucket is a linked list (or balanced red-black tree in Java 8+). Colliding keys are appended to the list at that bucket.
  2. **Open Addressing (Closed Hashing):** All elements are stored directly in the array table. If a collision occurs, the algorithm probes alternative buckets:
     - *Linear Probing:* Check $index + 1, index + 2, \dots$ (suffers from Primary Clustering).
     - *Quadratic Probing:* Check $index + 1^2, index + 2^2, \dots$
     - *Double Hashing:* Uses a secondary hash function for probe step size.

---

### 4. Trees & Binary Search Trees (BST)
- **Binary Search Tree (BST) Invariant:** For every node $N$:
  - Every key in the **left subtree** is strictly **less** than $N$'s key.
  - Every key in the **right subtree** is strictly **greater** than $N$'s key.
- **Balanced Trees (AVL & Red-Black Trees):** Prevent BST degradation into a skewed $O(N)$ linked list by performing tree rotations during insertion/deletion, guaranteeing height $h \approx \log_2 N$. (Linux kernel uses Red-Black trees for virtual memory VMA tracking and the CFS scheduler).

---

### 5. Graphs & Tries
- **Graph Representation:**
  - **Adjacency Matrix:** 2D array of size $V \times V$. $O(1)$ edge lookup; requires $O(V^2)$ memory. Ideal for dense graphs.
  - **Adjacency List:** Array of lists of size $V + E$. Space efficient $O(V + E)$. Ideal for sparse networks.
- **Trie (Prefix Tree):** A tree where every node represents a character. Lookups depend strictly on the **length of the search key ($L$)**, completely independent of the total number of items ($N$) in the database ($O(L)$ search).

---

## 3. Comprehensive Time & Space Complexity Master Matrix

| Data Structure | Access (Avg / Worst) | Search (Avg / Worst) | Insertion (Avg / Worst) | Deletion (Avg / Worst) | Space Complexity |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Array** | $O(1) / O(1)$ | $O(N) / O(N)$ | $O(N) / O(N)$ | $O(N) / O(N)$ | $O(N)$ |
| **Singly Linked List** | $O(N) / O(N)$ | $O(N) / O(N)$ | $O(1) / O(1)^*$ | $O(1) / O(1)^*$ | $O(N)$ |
| **Stack / Queue** | $O(N) / O(N)$ | $O(N) / O(N)$ | $O(1) / O(1)$ | $O(1) / O(1)$ | $O(N)$ |
| **Hash Table** | N/A | $O(1) / O(N)$ | $O(1) / O(N)$ | $O(1) / O(N)$ | $O(N)$ |
| **Binary Search Tree (BST)**| $O(\log N) / O(N)$ | $O(\log N) / O(N)$ | $O(\log N) / O(N)$ | $O(\log N) / O(N)$ | $O(N)$ |
| **Red-Black / AVL Tree** | $O(\log N) / O(\log N)$| $O(\log N) / O(\log N)$| $O(\log N) / O(\log N)$| $O(\log N) / O(\log N)$| $O(N)$ |
| **Binary Heap** | $O(1)$ (find-min/max) | $O(N) / O(N)$ | $O(\log N) / O(\log N)$| $O(\log N) / O(\log N)$| $O(N)$ |
| **Trie** | $O(L) / O(L)$ | $O(L) / O(L)$ | $O(L) / O(L)$ | $O(L) / O(L)$ | $O(\Sigma \times L \times N)$ |

*\* Insertion/Deletion in Linked List is $O(1)$ given direct pointer to node; finding node is $O(N)$.*

---

## 4. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Data structures organize memory for algorithmic efficiency. Arrays offer $O(1)$ random access and superior CPU cache locality via contiguous memory, but suffer from fixed sizing and $O(N)$ middle insertions. Linked Lists offer dynamic heap allocation and $O(1)$ pointer insertions at the cost of $O(N)$ traversal and pointer overhead. Hash Tables provide average $O(1)$ lookups via hash functions with chaining or open addressing to resolve collisions, but degrade to $O(N)$ under hash flooding attacks. Self-balancing trees like Red-Black trees guarantee strict $O(\log N)$ lookups, and Tries enable $O(L)$ prefix searches independent of dataset size. Choosing the right structure is critical for performance and security against algorithmic Denial of Service."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Claiming Hash Table lookups are *always* $O(1)$.  
  *Correction:* Hash tables have an **average** lookup of $O(1)$. In the **worst case** (where multiple keys collide into the same bucket), performance degrades to $O(N)$ linked list traversals.
- **Trap:** Forgetting hardware cache locality when comparing Arrays and Linked Lists.  
  *Correction:* In practice, an array iteration is significantly faster than a linked list iteration due to CPU hardware cache line prefetching, even if theoretical Big-O operations appear similar.
