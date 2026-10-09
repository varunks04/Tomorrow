# Operating System Fundamentals: Kernel, Modes & System Calls

> **Domain:** Core Computer Science  
> **Sub-Domain:** Operating Systems  
> **Interview Importance:** Foundational / High Frequency  

---

## 1. Topic & Definition

- **Operating System (OS):** The core system software that acts as an intermediary between computer hardware and user applications. Its primary responsibilities are resource management (CPU, memory, storage, I/O devices), process isolation, access control, and providing a stable abstraction layer over hardware.
- **The Kernel:** The central, permanently resident component of the operating system that runs with complete hardware privileges. It directly interacts with the CPU, Memory Management Unit (MMU), and peripheral device controllers.
- **Dual-Mode Operation:** Modern CPUs implement hardware-enforced protection rings (typically Ring 0 and Ring 3 on x86/x64 architectures) to isolate user applications from operating system internals:
  - **User Mode (Ring 3):** Untrusted execution environment where normal software (browsers, editors, user scripts) runs. Direct hardware access and privileged CPU instructions (such as disabling interrupts or modifying page tables) are strictly blocked by the CPU hardware.
  - **Kernel Mode / Supervisor Mode (Ring 0):** Privileged execution environment where the core OS kernel and device drivers execute. Has unrestricted access to physical memory, CPU control registers, and I/O bus lines.
- **System Call (`syscall`):** The controlled programmatic interface and gateway through which an unprivileged user-mode application requests a service from the privileged kernel (e.g., allocating memory, reading a file, spawning a thread, sending a network packet).

### Intuitive Analogy
> Think of an airport: **User Mode** is the passenger terminal where travelers move freely within designated civilian areas. **Kernel Mode** is the air traffic control tower and tarmac with full runway access. A passenger cannot simply walk onto the runway; they must present a valid boarding pass at a secure boarding gate (**System Call**). The security checkpoint inspects credentials and safely delegates the request.

---

## 2. How It Works: The System Call Transition Mechanism

When an application invokes a standard library function (such as `read()` in C, `read()` in Python, or `ReadFile()` on Windows), a transition from User Mode to Kernel Mode occurs through an atomic hardware trap.

```text
+-----------------------------------------------------------------------------------+
| USER SPACE (Ring 3)                                                               |
|                                                                                   |
|  [ User Application ]                                                             |
|           │                                                                       |
|           ▼                                                                       |
|  [ Standard Library Wrapper ] (e.g., libc read())                                  |
|           │                                                                       |
|           ▼ (Loads syscall ID into RAX/EAX, arguments into RDI, RSI, RDX...)      |
|  [ SYSCALL / SYSENTER Instruction ] ─────────────────────────┐                     |
+──────────────────────────────────────────────────────────────┼────────────────────+
                                                               │ Hardware Trap / CPU
                                                               │ Mode Switch (3 -> 0)
+──────────────────────────────────────────────────────────────┼────────────────────+
| KERNEL SPACE (Ring 0)                                        ▼                     |
|                                                                                   |
|  [ Interrupt / Syscall Dispatcher ] (Looks up index in System Service Descriptor) |
|           │                                                                       |
|           ▼                                                                       |
|  [ Parameter Validation & Permission Checks ] (Checks pointer bounds, DAC)        |
|           │                                                                       |
|           ▼                                                                       |
|  [ Kernel Driver / VFS Service Execution ] (Reads block from disk/cache)          |
|           │                                                                       |
|           ▼                                                                       |
|  [ SYSRET / IRET Instruction ] ──────────────────────────────┘                     |
|    (Switches CPU Mode back to Ring 3 and returns control to user app)             |
+-----------------------------------------------------------------------------------+
```

### Step-by-Step Execution Sequence
1. **Application Request:** The user program executes `read(fd, buffer, count)`.
2. **Library Marshaling:** The C runtime library places the unique system call number (e.g., `0` for `sys_read` on Linux x86_64) into the `RAX` register and function arguments into `RDI`, `RSI`, and `RDX`.
3. **Trap Execution:** The CPU executes the `syscall` (or `int 0x80`) machine instruction.
4. **Hardware Privilege Flip:** The CPU hardware atomically:
   - Switches execution privilege from Ring 3 to Ring 0.
   - Saves the return Program Counter (`RIP`) and flags register.
   - Switches the Stack Pointer (`RSP`) from the User Stack to the Kernel Stack.
   - Jumps to the kernel entry point predefined in the Model Specific Register (`MSR_LSTAR`).
5. **Kernel Dispatching:** The kernel looks up the system call table, validates that the user-space buffer pointer is valid (not referencing kernel memory), and performs the requested operation.
6. **Return to User Mode:** Once finished, the kernel places the return code into `RAX` and executes `sysret`, dropping CPU privilege back to Ring 3 and resuming the user program.

---

## 3. Practical Example: System Call Tracing

### Observing System Calls in Action (Linux `strace`)
In Linux, the `strace` diagnostic tool intercepts and records all system calls made by a process:

```bash
# Trace all system calls made by a simple 'cat' command
strace cat /etc/passwd
```

**Key System Calls Observed in Output:**
```text
openat(AT_FDCWD, "/etc/passwd", O_RDONLY) = 3        <-- Kernel opens file, returns File Descriptor 3
read(3, "root:x:0:0:root:/root:/bin/bash\n"..., 131072) = 2843 <-- Kernel copies bytes into user buffer
write(1, "root:x:0:0:root:/root:/bin/bash\n"..., 2843)   <-- Kernel writes to stdout (FD 1)
close(3)                                            <-- Closes File Descriptor
exit_group(0)                                       <-- Process termination
```

---

## 4. Key Differences: User Mode vs. Kernel Mode

| Evaluation Criteria | User Mode (Ring 3) | Kernel Mode (Ring 0) |
| :--- | :--- | :--- |
| **CPU Instruction Access** | Restrained to non-privileged instruction set. | Full access to privileged instructions (`cli`, `sti`, `lidt`, `wrmsr`). |
| **Direct Hardware Access** | Completely blocked. Cannot talk to disk, NIC, or screen directly. | Direct memory-mapped I/O and port I/O access to hardware devices. |
| **Memory Address Space** | Restricted to the virtual memory allocated to the specific process. | Access to entire physical RAM and kernel virtual memory space. |
| **Crash Impact (Blast Radius)** | Process crashes with SegFault (`SIGSEGV`); OS and other processes continue normally. | Kernel Panic (Linux) or Blue Screen of Death / BSOD (Windows); complete OS crash. |
| **Context Switch Overhead** | Low (within thread level). | High (requires hardware mode switch, register push, TLB impact). |

---

## 5. Cybersecurity Relevance & Threat Vectors

### 1. Privilege Escalation (PrivEsc)
- Attackers who obtain initial execution in user mode (e.g., via a remote web shell) seek to elevate privileges to Kernel Mode (Ring 0) or root/SYSTEM.
- **Kernel Exploits:** Exploiting vulnerabilities in kernel system call handlers (e.g., `Dirty COW` / CVE-2016-5195, `Dirty Pipe` / CVE-2022-0847). A flaw in kernel code allows user space to overwrite kernel memory tables and grant `root` privileges.

### 2. Meltdown and Spectre (Microarchitectural Side-Channel Attacks)
- **Meltdown (CVE-2017-5754):** Exploited out-of-order execution in CPUs. User space code speculatively accessed kernel memory before the CPU privilege check was committed, leaking kernel secrets via CPU cache timing attacks. Mitigated by **KPTI (Kernel Page Table Isolation)**.

### 3. Rootkits
- **Kernel-level Rootkits:** Operate in Ring 0 (e.g., as malicious kernel modules/drivers). Because they execute with kernel privileges, they can hook the system call table, hiding malicious processes, open ports, and files from user-space tools like `ps`, `netstat`, or antivirus agents.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Operating systems enforce a hardware-backed separation between unprivileged User Mode (Ring 3) and privileged Kernel Mode (Ring 0). User applications cannot interact with hardware or access physical memory directly; instead, they trigger a hardware interrupt known as a System Call (`syscall`). The CPU saves the process state, switches to Ring 0, executes the kernel service after validating parameters, and transitions back to Ring 3. This dual-mode design provides memory protection, crash isolation, and the foundational security boundary between processes and the operating system."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing a standard library function with a system call.  
  *Correction:* `printf()` is a user-space C library function that formats a string in a user buffer. It internally calls `write()`, which is the actual system call that traps into the kernel.
- **Trap:** Claiming that root/administrator runs in Kernel Mode.  
  *Correction:* Root/administrator is a **User Mode** identity with high filesystem and process permissions. The root user still runs in Ring 3 and must make system calls to enter Ring 0. Only the OS kernel and device drivers run in Ring 0.

### ❓ Expected Follow-Up Questions
1. **Q:** What is Kernel Page Table Isolation (KPTI)?  
   **A:** A defensive mitigation introduced to counter the Meltdown vulnerability by completely unmapping kernel memory from user-space page tables while running in User Mode.
2. **Q:** What happens if a user process tries to execute a privileged instruction like `wrmsr` directly?  
   **A:** The CPU detects the Ring 3 state, immediately blocks execution, raises a General Protection Fault (GPF), and the kernel terminates the offending process with a segmentation fault.
