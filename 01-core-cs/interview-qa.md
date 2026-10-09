# Core Computer Science — Technical Interview Q&A

### Q1: What is the fundamental difference between User Mode and Kernel Mode?
> **Model Answer:**  
> CPUs execute in dual modes to protect system integrity:
> - **User Mode (Ring 3):** Applications have restricted access to hardware. They cannot execute privileged CPU instructions or directly access memory outside their allocated virtual address space.
> - **Kernel Mode (Ring 0):** The OS kernel has unrestricted execution rights over CPU, registers, and physical hardware.  
> Whenever an application requires hardware I/O (reading a file, opening a network socket), it executes a **System Call** (`syscall`), which triggers a trap/software interrupt, safely switching execution to Kernel Mode via the OS interrupt vector table.

---

### Q2: How does a context switch work and why is it expensive?
> **Model Answer:**  
> A context switch is the OS mechanism of saving the state of the currently running process/thread and restoring the state of another process to resume execution.
> 1. The OS saves CPU registers, program counter, and stack pointer into the process's **Process Control Block (PCB)**.
> 2. The CPU updates memory mapping registers (Page Table Base Register / CR3 in x86).
> 3. The CPU cache and Translation Lookaside Buffer (TLB) are invalidated or partially flushed.
> It is expensive because it consumes CPU cycles without executing application logic and destroys CPU cache locality.

---

### Q3: What are the 4 necessary conditions for a Deadlock?
> **Model Answer (Coffman Conditions):**
> 1. **Mutual Exclusion:** Resources cannot be shared simultaneously.
> 2. **Hold and Wait:** A process holding at least one resource is waiting to acquire additional resources held by others.
> 3. **No Preemption:** Resources cannot be forcibly taken from a process holding them.
> 4. **Circular Wait:** A closed chain of processes exists such that each process holds a resource needed by the next.  
> *Mitigation:* Breaking any one condition prevents deadlocks (e.g., establishing a strict global resource ordering breaks Circular Wait).

---

### Q4: Explain the difference between Clustered and Non-Clustered Indexes in SQL.
> **Model Answer:**
> - **Clustered Index:** Defines the physical ordering of the rows on disk. A table can have **only one** clustered index (typically the Primary Key). The leaf nodes of the B+ Tree contain the actual data rows.
> - **Non-Clustered Index:** A separate structure containing sorted keys and pointers (row locators / clustered key values) pointing to the physical data rows. A table can have multiple non-clustered indexes.

---

### Q5: How do SOLID principles improve application security?
> **Model Answer:**
> - **Single Responsibility Principle (SRP):** Limits module attack surface; security-critical code (auth/crypto) is isolated from generic logic.
> - **Open/Closed Principle (OCP):** Extensions are built without modifying proven, audited base implementations.
> - **Liskov Substitution Principle (LSP):** Subclasses fulfill parent contracts without introducing unexpected insecure behaviors.
> - **Interface Segregation Principle (ISP):** Clients don't depend on unneeded methods, upholding the Principle of Least Privilege.
> - **Dependency Inversion Principle (DIP):** Enables mocking and pluggable security providers (e.g. swapping crypto engines easily).
