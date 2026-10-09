# CPU Scheduling Algorithms & Schedulers

> **Domain:** Core Computer Science  
> **Sub-Domain:** Operating Systems  
> **Interview Importance:** Medium-High / Core Systems Theory  

---

## 1. Topic & Definitions

- **CPU Scheduling:** The OS process manager mechanism that determines which ready process is allocated CPU execution time when multiple processes compete for CPU cycles.
- **Preemptive vs. Non-Preemptive Scheduling:**
  - **Non-Preemptive:** Once the CPU is allocated to a process, the process holds it until it voluntarily terminates or switches to a waiting state (e.g. for I/O). The OS cannot forcibly take the CPU away.
  - **Preemptive:** The OS scheduler can interrupt and pause a currently running process (via hardware timer interrupts) to allocate the CPU to a higher-priority or newly arrived process.
- **Key Scheduling Metrics:**
  - **Burst Time ($BT$):** Time required by a process to execute on the CPU.
  - **Arrival Time ($AT$):** Time at which the process arrives in the Ready Queue.
  - **Completion Time ($CT$):** Exact time at which process finishes execution.
  - **Turnaround Time ($TAT$):** Total elapsed time from arrival to completion ($TAT = CT - AT$).
  - **Waiting Time ($WT$):** Total time spent idling in the ready queue ($WT = TAT - BT$).
  - **Response Time ($RT$):** Time from arrival until the first time CPU executes it.

---

## 2. Core Scheduling Algorithms Walkthrough

### 1. First-Come, First-Served (FCFS)
- **Nature:** Non-preemptive. Executes strictly in order of arrival.
- **Major Problem:** **The Convoy Effect**. If a CPU-heavy process arrives first ($BT = 100$), subsequent short I/O-bound processes ($BT = 2$) are forced to wait, leading to miserable average waiting times and low device utilization.

### 2. Shortest Job First (SJF) & Shortest Remaining Time First (SRTF)
- **SJF (Non-Preemptive):** Schedules the process with the shortest CPU burst time next.
  - **Provably Optimal:** Produces the lowest theoretical minimum average waiting time.
  - **Limitation:** Impossible to implement perfectly in real general-purpose OSs because the future CPU burst length cannot be known in advance (must be estimated using exponential smoothing).
- **SRTF (Preemptive SJF):** If a new process arrives with a remaining burst time smaller than the currently executing process, the CPU is preempted and given to the newcomer.
- **Major Problem:** **Starvation**. Long processes can be postponed indefinitely if a continuous stream of short processes enters the ready queue.

### 3. Round Robin (RR)
- **Nature:** Preemptive. Designed for interactive and time-sharing systems.
- **Mechanism:** Each process is assigned a small slice of CPU time called a **Time Quantum ($q$)** (typically 10-100 ms). If the process is still running after quantum expiration, it is preempted via a timer interrupt and appended to the back of the Ready Queue.
- **The Quantum Tradeoff:**
  - If $q$ is **too large**: Degenerates into FCFS (Convoy effect returns).
  - If $q$ is **too small**: System spends massive overhead on context switches instead of productive application work.

```text
Time Quantum = 2ms

Ready Queue: [P1(BT=5), P2(BT=3), P3(BT=1)]

Gantt Chart:
|  P1 (0-2)  |  P2 (2-4)  |  P3 (4-5)  |  P1 (5-7)  |  P2 (7-8)  |  P1 (8-9)  |
0            2            4            5            7            8            9 (Done)
```

### 4. Priority Scheduling & Multilevel Feedback Queue (MLFQ)
- **Priority Scheduling:** Each process has a priority integer; CPU is allocated to the highest-priority process.
  - **Problem: Priority Inversion:** A low-priority process holds a lock needed by a high-priority process, while a medium-priority process starves both. Solved via **Priority Inheritance**.
- **Multilevel Feedback Queue (MLFQ):** Modern OS scheduling standard (used in Linux Completely Fair Scheduler / CFS principles and Windows). Uses multiple queues with varying priorities and quantums; dynamically demotes CPU-bound processes to lower queues and promotes I/O-bound interactive processes to high-priority queues. Solves starvation via **Aging**.

---

## 3. Algorithm Comparison Matrix

| Algorithm | Preemption | Starvation Possible? | Optimal Avg WT? | Best Used In |
| :--- | :--- | :---: | :---: | :--- |
| **FCFS** | Non-preemptive | No | No (Convoy Effect) | Simple batch systems, printer spoolers |
| **SJF** | Non-preemptive | **Yes** (Long jobs starve) | **Yes** (Theoretical) | Long-term batch processing |
| **SRTF** | Preemptive | **Yes** (Long jobs starve) | **Yes** | Real-time short task systems |
| **Priority** | Both | **Yes** (Low priority starves) | No | Real-time systems, mission-critical tasks |
| **Round Robin** | **Preemptive** | **No** (Strict fairness) | Moderate | Time-sharing systems, modern desktop OS |
| **MLFQ** | Preemptive | Prevented via **Aging** | High | Modern general-purpose OS (Linux, Windows) |

---

## 4. Cybersecurity Relevance & Threat Vectors

### 1. CPU Starvation Denial of Service (Fork Bombs)
- A process rapidly and recursively spawns child processes (e.g. bash `:(){ :|:& };:`). The scheduler's ready queue and process table are flooded with millions of threads, starving all legitimate processes of CPU time and freezing the system.
- **Defense:** Enforce process limits per user via `ulimit -u` in Linux or system cgroups limits (`pids.max`).

### 2. Priority Inversion Exploits
- Attackers exploit scheduling algorithms by forcing low-priority threads to acquire exclusive mutexes on critical shared resources, deliberately starving high-priority security or monitoring processes (such as antivirus scans or firewall packet inspectors).

### 3. Timing Side-Channel Attacks
- Variations in CPU scheduling latencies and quantum interruptions can reveal information about cryptographic keys or process execution paths to an unprivileged co-located process on the same machine (e.g. in shared cloud virtual machines).

---

## 5. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"CPU scheduling determines which process in the ready queue is granted CPU time. Schedulers are either non-preemptive, where processes yield voluntarily, or preemptive, where timer interrupts allow the OS to reclaim CPU control. Key algorithms include FCFS, which suffers from the convoy effect; SJF/SRTF, which is theoretically optimal for waiting time but suffers from starvation; and Round Robin, which ensures fairness in interactive systems using a time quantum. Modern operating systems combine these concepts into Multilevel Feedback Queues with Priority Aging to guarantee fast interactive response times while preventing long background jobs from starving."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Forgetting that Round Robin performance strictly depends on the size of the time quantum. Always mention the trade-off between convoy effect (if too large) and excessive context-switching overhead (if too small).
- **Trap:** Saying SJF is used in everyday Windows/Linux. It cannot be used directly because the OS cannot predict the exact CPU burst time of an arbitrary user application in advance.
