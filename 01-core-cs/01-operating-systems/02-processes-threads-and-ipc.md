# Processes, Threads, Context Switching & Inter-Process Communication (IPC)

> **Domain:** Core Computer Science  
> **Sub-Domain:** Operating Systems  
> **Interview Importance:** Very High / Foundational Core Concept  

---

## 1. Topic & Definitions

- **Process:** An independent program in execution. It represents the active state of an executable, comprising its own allocated virtual address space, memory segments (Code, Data, Heap, Stack), file descriptors, security identifiers, and environment variables.
- **Thread:** The smallest sequence of programmed instructions that can be managed independently by an OS scheduler. Often called a **lightweight process (LWP)**, a thread executes within the address space of its parent process and shares heap memory, global variables, and open files with other threads in that process, but maintains its own program counter, CPU registers, and private execution stack.
- **Process Control Block (PCB):** A kernel data structure containing complete information about an active process: PID, PPID, process state, program counter, CPU registers, scheduling priority, memory management pointers (page table base), and I/O status.
- **Context Switch:** The state transition mechanism where the OS stops the currently executing process or thread, preserves its CPU execution state in its PCB/TCB, and restores the execution state of another process/thread to resume on the CPU.
- **Inter-Process Communication (IPC):** Mechanisms provided by the operating system allowing separate, memory-isolated processes to exchange data, synchronize actions, and communicate.

---

## 2. How It Works: Process States & Context Switching

### The 5-State Process Lifecycle Model

```text
       [Admitted]               [Scheduler Dispatch]
  NEW ────────────► READY ─────────────────────────────────► RUNNING
                      ▲                                         │
                      │           [I/O or Event Wait]           ▼
                      └──────────────────────────────────── WAITING (BLOCKED)
                                [I/O Complete / Event]          │
                                                                │ [Exit / Terminate]
                                                                ▼
                                                            TERMINATED
```

1. **New:** Process is created by `fork()`/`exec()` or `CreateProcess()` but not yet loaded into active memory.
2. **Ready:** Program is loaded into memory, PCB initialized, waiting in the ready queue for CPU allocation.
3. **Running:** Instructions are actively being executed by a CPU core.
4. **Waiting / Blocked:** Process cannot proceed because it is waiting for an external event (disk read, network socket data, mutex acquisition).
5. **Terminated:** Process has finished execution; its resources are freed, but its exit status remains in the process table until collected by the parent (preventing "Zombie" processes).

### Detailed Context Switch Internal Flow

```text
Process A (User Mode)          Operating System (Kernel Mode)          Process B (User Mode)
        │                                     │                                  │
        │ [Timer Interrupt / Syscall]         │                                  │
        ├────────────────────────────────────►│                                  │
        │                                     │ 1. Save Process A CPU registers  │
        │                                     │    (RIP, RSP, GPRs) to PCB_A     │
        │                                     │ 2. Update PCB_A state (READY)    │
        │                                     │ 3. Run CPU Scheduler algorithm   │
        │                                     │ 4. Select Process B              │
        │                                     │ 5. Switch Memory Map (CR3/TLB)   │
        │                                     │ 6. Restore Process B CPU state   │
        │                                     │    from PCB_B                    │
        │                                     ├─────────────────────────────────►│
        │                                     │                                  │ Process B executes...
```

---

## 3. Inter-Process Communication (IPC) Mechanisms

Because processes have strict memory isolation, they cannot directly read or write each other's memory. The OS provides standard IPC mechanisms:

| IPC Mechanism | Architecture & How It Works | Speed & Overhead | Common Use Case |
| :--- | :--- | :--- | :--- |
| **Anonymous Pipe** | Unidirectional byte stream between parent-child processes (via `pipe()`). Data resides in a kernel buffer. | High throughput, local only. | Shell piping (`cat file \| grep err`). |
| **Named Pipe (FIFO)**| Unidirectional/bidirectional pipe accessible via a filesystem path; works between unrelated processes. | Moderate; requires open/close file handles. | Local client-server daemons. |
| **Shared Memory** | Two processes map the exact same physical RAM pages into their virtual address spaces (`shmget`, `mmap`). | **Fastest IPC:** Zero kernel copying once mapped. Requires mutexes. | Real-time audio/video streaming, high-frequency trading. |
| **Message Queues** | Kernel-managed linked list of messages tagged with types; processes read/write discrete messages. | Moderate; involves kernel-space data copying. | Asynchronous job dispatching. |
| **Unix Domain Sockets**| Bidirectional socket communication using the standard socket API, but bypassing network protocol stack. | Fast; avoids IP/TCP overhead. | Docker daemon (`/var/run/docker.sock`), local database IPC. |
| **Network Sockets** | Sockets operating over TCP/UDP network stack; works across machines. | Slowest; protocol stack overhead, encapsulation. | Web apps, microservices, remote APIs. |
| **Signals** | Asynchronous software interrupts sent to a process (`SIGINT`, `SIGTERM`, `SIGKILL`). | Minimal payload (signal integer only). | Process lifecycle management, aborts. |

---

## 4. Key Differences: Process vs. Thread

| Evaluation Feature | Process | Thread |
| :--- | :--- | :--- |
| **Memory Isolation** | Completely isolated address space. Cannot corrupt peer processes. | Shares Code, Data, and Heap with all threads in process. Only Stack is private. |
| **Creation Cost** | Expensive: requires allocating new page tables, file descriptor tables, and PCB. | Cheap: reuses parent process memory space and page tables. |
| **Context Switch Cost** | High: requires flushing TLB and switching CR3 page table base register. | Low: page tables remain untouched; only CPU registers and stack pointer flip. |
| **Communication** | Must use IPC mechanisms (Pipes, Sockets, Shared Memory). | Direct read/write to shared heap variables and pointers. |
| **Crash Blast Radius** | Isolated: if one process crashes, other processes continue unaffected. | Fatal: an unhandled exception (SegFault) in one thread terminates the entire process. |

---

## 5. Cybersecurity Relevance & Threat Vectors

### 1. Process Injection Attacks
Malware commonly injects malicious code into legitimate, trusted running processes (e.g., `svchost.exe`, `explorer.exe`) to evade endpoint detection (EDR):
- **DLL Injection:** Malware forces a remote process to execute `LoadLibrary()` pointing to a malicious DLL via `CreateRemoteThread`.
- **Process Hollowing:** An attacker spawns a benign process in a suspended state, unmaps its legitimate code using `NtUnmapViewOfSection`, writes malicious shellcode into that space, and resumes the thread.
- **Reflective DLL Injection:** Loading a DLL directly from memory without saving it to disk or using Windows `LoadLibrary`.

### 2. Zombie and Orphan Processes (Resource Starvation)
- **Zombie Process:** A child process that has completed execution (`exit()`), but its entry remains in the process table because the parent has not yet read its exit status via `wait()`. While zombies consume no memory, a massive accumulation can exhaust the OS PID table, preventing new processes from spawning (Denial of Service).
- **Orphan Process:** A child whose parent terminated before it did. In Linux, orphan processes are automatically adopted by `init` / `systemd` (PID 1), which periodically reaps them.

### 3. Insecure Shared Memory & Race Conditions
- If shared memory is created with overly permissive permissions (e.g., world-readable/writable `0666`), an unprivileged attacker can read sensitive secrets or corrupt data structures.
- Without atomic synchronization (mutexes/semaphores), concurrent threads accessing shared memory suffer from **Race Conditions** and **TOCTOU (Time-of-Check to Time-of-Use)** vulnerabilities.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"A process is an isolated executing program with its own dedicated virtual address space and resources, providing strong security boundaries. A thread is a lightweight execution path within a process that shares the heap, code, and global data with sibling threads while maintaining its own private stack and registers. Context switching between processes is computationally expensive because the OS must reload page tables and invalidate the TLB, whereas switching between threads in the same process avoids memory re-mapping. Because processes are isolated, they must communicate via IPC mechanisms such as pipes, sockets, or shared memory."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Believing threads share their stack.  
  *Correction:* Threads NEVER share execution stacks. Each thread has its own private stack to track local variables, parameters, and function call frames. Only the heap and global data are shared.
- **Trap:** Thinking `fork()` immediately copies the entire physical RAM of the parent.  
  *Correction:* Modern OSs implement **Copy-On-Write (COW)**. The child shares the parent's physical pages in read-only mode. Only when one process writes to a page does the OS allocate a new physical frame and copy that specific page.

### ❓ Expected Follow-Up Questions
1. **Q:** What is the difference between a mutex and a binary semaphore in thread synchronization?  
   **A:** Ownership. A mutex can only be unlocked by the exact thread that locked it (ownership concept). A semaphore can be signaled (unlocked) by any thread or process.
2. **Q:** How do EDRs monitor process injection?  
   **A:** By hooking user-space API calls (e.g., `VirtualAllocEx`, `WriteProcessMemory`, `CreateRemoteThread`) and monitoring kernel callbacks via Microsoft Windows Threat Intelligence (ETW-TI) drivers.
