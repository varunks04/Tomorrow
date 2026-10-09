# Stack vs. Heap, Memory Layout & Buffer Overflow Internals

> **Domain:** Core Computer Science  
> **Sub-Domain:** Operating Systems & Memory Safety  
> **Interview Importance:** Very High / Fundamental to Binary Exploitation & Memory Safety  

---

## 1. Topic & Definitions

- **Virtual Address Space Layout:** A standardized memory map assigned by the OS to every executing process (typically spanning 0x0000000000000000 to 0x7FFFFFFFFFFF in 64-bit user space, with the upper addresses reserved for the kernel).
- **The Stack:** A structured, LIFO (Last-In, First-Out) memory region managed automatically by the CPU via the Stack Pointer (`RSP/ESP`) and Base Pointer (`RBP/EBP`). Used for local variable storage, function call parameters, and return addresses.
- **The Heap:** An unstructured, dynamically managed memory pool allocated at runtime (via `malloc()` in C, `new` in C++, or object allocators in Python/Java). It is managed manually or via a Garbage Collector.

```text
High Memory (0x7FFF... / 0xFFFF...)
+-------------------------------------------------------------+
| KERNEL SPACE (Restricted to Ring 0 / Supervisor)            |
+-------------------------------------------------------------+
| STACK (Grows downward: high memory -> low memory)           |
|   • Local variables, stack frames, return addresses (RIP)   |
|                             │                               |
|                             ▼                               |
|                     [ Unallocated Gap ]                     |
|                             ▲                               |
|                             │                               |
| HEAP (Grows upward: low memory -> high memory)              |
|   • Dynamically allocated objects (malloc, new)             |
+-------------------------------------------------------------+
| BSS SEGMENT (Block Started by Symbol: Uninitialized globals)|
+-------------------------------------------------------------+
| DATA SEGMENT (Initialized global and static variables)      |
+-------------------------------------------------------------+
| TEXT SEGMENT (Compiled CPU instructions / Machine code)     |
+-------------------------------------------------------------+
Low Memory (0x0000...)
```

---

## 2. Anatomy of a Stack Frame & Function Call Execution

When a function `foo(x, y)` is invoked, the CPU pushes a **Stack Frame** onto the stack:

```text
Low Memory (Top of Stack: RSP points here)
▲
│  [ Local Variables ]       (e.g., char buffer[64]; int counter;)
│  [ Stack Canary ]          (Random 8-byte value placed by compiler to detect overflow)
│  [ Saved Frame Pointer ]   (Saved RBP: points to previous stack frame base)
│  [ Saved Return Address ]  (Saved RIP: pointer to instruction to execute after ret)
│  [ Function Parameters ]   (Passed arguments if not held in registers)
▼
High Memory (Bottom of Stack)
```

### The Function Call Sequence
1. **Prologue:**
   ```nasm
   push rbp          ; Save the previous base pointer
   mov  rbp, rsp     ; Set the new base pointer for this frame
   sub  rsp, 0x40    ; Allocate 64 bytes for local variables
   ```
2. **Execution:** Local variables are accessed relative to `RBP` (e.g., `[rbp - 0x10]`).
3. **Epilogue:**
   ```nasm
   mov  rsp, rbp     ; Deallocate local variables
   pop  rbp          ; Restore caller's base pointer
   ret               ; Pop saved return address into RIP and jump to caller
   ```

---

## 3. Key Differences: Stack vs. Heap

| Dimension | The Stack | The Heap |
| :--- | :--- | :--- |
| **Allocation Mechanism** | Automatic by CPU instruction pointer movement (`sub rsp, X`). | Manual or runtime allocator (`malloc()`, `brk()`, `sbrk()`, `mmap()`). |
| **Growth Direction** | Grows **downward** (from high address to low address). | Grows **upward** (from low address to high address). |
| **Allocation Speed** | Extremely fast (single CPU cycle arithmetic instruction). | Slower (requires finding free memory blocks, coalescing chunks). |
| **Lifetime** | Strictly scoped to the lifetime of the enclosing function. | Dynamic; persists until explicitly `free()`d or garbage-collected. |
| **Fragmentation** | Zero fragmentation (strictly LIFO structure). | Susceptible to both internal and external heap fragmentation. |
| **Size Limit** | Small and fixed per thread (typically 8 MB on Linux, 1 MB on Windows). | Vast (limited only by available virtual memory and RAM/swap). |
| **Exhaustion Error** | **Stack Overflow** (`SIGSEGV` or segmentation fault). | `Out of Memory` (OOM error / `NULL` returned by malloc). |

---

## 4. Cybersecurity Relevance: Memory Corruption Vulnerabilities

### 1. Classic Stack Buffer Overflow (Stack Smashing)
- Occurs when an application writes more data to a stack-allocated buffer than it can hold (e.g., using unbounded C functions like `strcpy()`, `gets()`, `scanf("%s")`).
- **Exploitation Mechanism:**
  ```text
  [ buffer (64 bytes) ] ──► [ Stack Canary ] ──► [ Saved RBP ] ──► [ Saved RIP (Return Address) ]
  ══════════════════════════════════════════════════════════════════════════════════════════════
  Attacker input: 'A' * 64 + [Forged Canary] + [Overwritten RBP] + [Malicious Shellcode Address]
  ```
- When the function executes `ret`, the CPU pops the attacker's forged return address into the instruction pointer (`RIP`), transferring execution control to attacker-controlled code.

### 2. Modern Memory Exploit Mitigations
To prevent stack smashing, modern compilers and OSs deploy four essential defenses:

| Defense | Full Name | How It Neutralizes the Attack |
| :--- | :--- | :--- |
| **Stack Canaries** | Stack Smashing Protector | Compiler places a random secret integer before the return address. Prior to returning, the function verifies if the canary was modified; if altered, the program terminates immediately with `*** stack smashing detected ***`. |
| **DEP / NX Bit** | Data Execution Prevention | Hardware MMU marks the Stack and Heap as non-executable. Injected shellcode cannot execute. |
| **ASLR** | Address Space Layout Randomization | Randomizes base addresses of Stack, Heap, and libraries, making hardcoded shellcode pointers invalid. |
| **PIE** | Position Independent Executable | Randomizes the base address of the `.text` executable code section itself. |

### 3. Return-Oriented Programming (ROP)
- When DEP/NX prevents executing shellcode on the stack, attackers utilize **ROP**.
- Instead of injecting new code, the attacker stitches together tiny snippets of existing, legitimate executable machine instructions in loaded libraries (`libc`), ending in `ret` instructions (called **Gadgets**).
- By chaining gadgets on the stack, the attacker can invoke system calls (e.g., `execve("/bin/sh")`) without executing a single byte of untrusted memory.

### 4. Heap Corruptions: Use-After-Free (UAF) & Double Free
- **Use-After-Free (CWE-416):** An application frees a heap pointer but continues to read or write to it. If the attacker allocates a malicious object that reclaims that exact heap chunk, the stale pointer invokes attacker-controlled function pointers.
- **Double Free (CWE-415):** Freeing the same heap memory block twice corrupts the heap manager's internal doubly linked free-list, allowing arbitrary memory writes.

---

## 5. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"The virtual address space of a process is partitioned into segments: Text for code, Data/BSS for globals, the Heap growing upward for dynamic runtime allocations, and the Stack growing downward for local variables and execution frames. The stack is fast, automatically managed, and LIFO, while the heap is dynamic and manually managed. A classic stack buffer overflow occurs when unbounded input overwrites the stack frame, replacing the saved return address (`saved RIP`) to hijack CPU execution flow. In modern systems, this is mitigated by Stack Canaries, the NX/DEP bit, and ASLR, which forced exploit development to evolve toward Return-Oriented Programming (ROP) and heap-based vulnerabilities like Use-After-Free."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Believing stack canaries stop all memory corruption.  
  *Correction:* Canaries only protect against contiguous stack overwrites. They do not protect against Heap overflows, Use-After-Free, or arbitrary memory read/write primitives via format string bugs.
- **Trap:** Forgetting which way the stack grows.  
  *Correction:* On x86/x64, the stack grows **downward** toward lower memory addresses, while heap buffers write **upward** toward higher addresses—which is why an overflowing buffer naturally advances toward the return address!
