# Algorithmic Complexity, Big-O Notation & Sorting/Searching Algorithms

> **Domain:** Core Computer Science  
> **Sub-Domain:** Data Structures & Algorithms  
> **Interview Importance:** Very High / Foundational CS Technical Competency  

---

## 1. Topic & Definitions

- **Asymptotic Analysis:** The mathematical framework used to evaluate and compare the resource consumption (execution time and memory space) of an algorithm as the input size $N$ grows toward infinity ($N \to \infty$).
- **The Three Asymptotic Notations:**
  - **Big-O ($O$):** Describes the **Upper Bound** (worst-case scenario). Guarantees an algorithm will never perform worse than this rate.
  - **Big-$\Omega$ ($\Omega$):** Describes the **Lower Bound** (best-case scenario).
  - **Big-$\Theta$ ($\Theta$):** Describes the **Tight Bound** (both upper and lower bounds grow at the same asymptotic rate).

```text
Complexity Hierarchy (Fastest to Slowest):
O(1) < O(log N) < O(N) < O(N log N) < O(N^2) < O(2^N) < O(N!)
```

---

## 2. Complexity Classes Visualized

| Complexity Class | Common Name | Execution Scale ($N=1,000$) | Typical Example |
| :--- | :--- | :--- | :--- |
| **$O(1)$** | Constant | 1 operation | Array indexing by index; Hash Table lookup (average). |
| **$O(\log N)$** | Logarithmic | $\approx 10$ operations | Binary search on sorted array; B+ Tree seek. |
| **$O(N)$** | Linear | 1,000 operations | Linear search through unsorted list; finding Max/Min. |
| **$O(N \log N)$** | Linearithmic | $\approx 10,000$ operations | Merge Sort, Quick Sort (average), Heap Sort. |
| **$O(N^2)$** | Quadratic | $1,000,000$ operations | Bubble sort, nested loops comparing all pairs. |
| **$O(2^N)$** | Exponential | $1.07 \times 10^{301}$ (Heat death of universe) | Recursive Fibonacci; brute-forcing an $N$-bit key. |
| **$O(N!)$** | Factorial | Incalculable | Traveling Salesperson brute force; generating all permutations. |

---

## 3. Search Algorithms: Linear vs. Binary Search

### 1. Linear Search
- **Mechanism:** Sequentially inspects every element from index $0$ to $N-1$ until target is found.
- **Preconditions:** Works on completely unsorted data.
- **Complexity:** Best: $O(1)$, Worst: $O(N)$, Average: $O(N)$.

### 2. Binary Search
- **Mechanism:** Divide-and-conquer algorithm. Inspects the midpoint:
  - If $Target == Mid$: Return index.
  - If $Target < Mid$: Discard right half; recurse on left half.
  - If $Target > Mid$: Discard left half; recurse on right half.
- **Strict Precondition:** Data **MUST be sorted** beforehand!
- **Complexity:** Best: $O(1)$, Worst: $O(\log N)$, Space: $O(1)$ (iterative).

```text
Target = 23 | Sorted Array: [ 2, 5, 8, 12, 16, 23, 38, 56, 72, 91 ]
Step 1: Low = 0, High = 9, Mid = 4 (Value 16). 23 > 16 -> Search right half.
Step 2: Low = 5, High = 9, Mid = 7 (Value 56). 23 < 56 -> Search left half.
Step 3: Low = 5, High = 6, Mid = 5 (Value 23). Match found at Index 5! (Took 3 steps instead of 6)
```

---

## 4. Sorting Algorithms Comparison Master Matrix

| Sorting Algorithm | Best Time | Average Time | Worst Time | Space Overhead | Stable? | In-Place? | Key Mechanism & Vulnerabilities |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **Bubble Sort** | $O(N)$ | $O(N^2)$ | $O(N^2)$ | $O(1)$ | Yes | Yes | Swaps adjacent inverted elements. Educational only. |
| **Insertion Sort** | $O(N)$ | $O(N^2)$ | $O(N^2)$ | $O(1)$ | Yes | Yes | Inserts each item into sorted prefix. Fast on small/near-sorted arrays ($N < 16$). |
| **Merge Sort** | $O(N \log N)$ | $O(N \log N)$ | $O(N \log N)$ | $O(N)$ | **Yes** | No | Divide-and-conquer. Guarantees $O(N \log N)$ worst-case; requires extra RAM. |
| **Quick Sort** | $O(N \log N)$ | $O(N \log N)$ | $O(N^2)^*$ | $O(\log N)$ | **No** | **Yes** | Partition around pivot. Fastest in practice; worst-case if pivot poorly chosen. |
| **Heap Sort** | $O(N \log N)$ | $O(N \log N)$ | $O(N \log N)$ | $O(1)$ | **No** | **Yes** | Builds max-heap, repeatedly extracts root. Deterministic time, zero extra RAM. |

*\* Note: QuickSort degrades to $O(N^2)$ if the pivot chosen is consistently the extreme min/max (e.g. sorting an already sorted array with first/last element pivot). Modern implementations use Randomized Pivot or Median-of-Three to mitigate this.*

---

## 5. Recursion & Stack Space Limits

- **Call Stack:** Every recursive call allocates a new stack frame containing local variables and return addresses.
- **Base Case:** The halting condition in a recursive function. Missing a base case causes unbounded recursion.
- **Stack Overflow Exception:** The call stack exceeds the thread's memory limit (typically 1-8 MB), causing an uncatchable segmentation fault / stack exhaustion.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Asymptotic analysis evaluates algorithmic growth as input scales to infinity. Big-O represents the upper bound worst-case scenario. Linear search operates in $O(N)$ on unsorted data, while Binary Search executes in $O(\log N)$ by halving sorted search spaces. Among sorting algorithms, Merge Sort guarantees strict $O(N \log N)$ worst-case performance and stability at the cost of $O(N)$ auxiliary memory space, whereas Quick Sort is typically faster in practice due to cache locality and in-place sorting, but has an $O(N^2)$ worst-case risk if adversarial inputs force unbalanced pivot partitioning. In systems programming, hybrid algorithms like Timsort (Merge + Insertion) or Introsort (Quick + Heap + Insertion) are chosen to guarantee both peak average performance and deterministic worst-case bounds."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Forgetting the prerequisite for Binary Search.  
  *Correction:* Never say "Run binary search on this list" without verifying the data is **sorted**. Searching an unsorted list with binary search produces garbage results.
- **Trap:** Saying QuickSort is always $O(N \log N)$.  
  *Correction:* QuickSort is $O(N \log N)$ on **average**. Its worst-case is strictly $O(N^2)$ unless paired with randomized pivots or fallback algorithms like Introsort.
