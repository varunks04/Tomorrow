# Containers vs. VMs and Linux Kernel Internals: Namespaces, Cgroups, and OverlayFS

## 1. Topic & Definition
Understanding container security requires demystifying the fundamental abstraction: **A container is not a lightweight Virtual Machine; a container is simply a standard Linux process running directly on the host kernel, constrained by Linux kernel isolation primitives**.

While a **Virtual Machine (VM)** virtualizes hardware through a hypervisor and runs a complete, independent guest operating system (with its own guest kernel, memory management, and device drivers), a **Container** shares the underlying host Linux kernel, using kernel namespaces for visibility isolation and control groups (cgroups) for resource constraints.

---

## 2. How It Works: Architectural Comparison & Linux Kernel Foundations

```
+---------------------------------------------------------------------------------------------------+
|                                 Virtual Machines vs. Containers                                   |
+---------------------------------------------------------------------------------------------------+

   VIRTUAL MACHINE ARCHITECTURE:                      CONTAINER ARCHITECTURE:
   +---------------------------------------+          +---------------------------------------+
   | App 1 (Bin/Libs)  | App 2 (Bin/Libs)  |          | App 1 (Bin/Libs)  | App 2 (Bin/Libs)  |
   +-------------------+-------------------+          +-------------------+-------------------+
   | Guest OS 1 (Linux)| Guest OS 2 (Win)  |          | Container Engine (Docker / containerd)|
   +-------------------+-------------------+          +---------------------------------------+
   | Hypervisor (Type 1: KVM/ESXi / Type 2)|          | Host Linux Kernel                     |
   +---------------------------------------+          | (Namespaces + Cgroups + Seccomp)      |
   | Physical Server Hardware              |          +---------------------------------------+
   +---------------------------------------+          | Physical Server Hardware              |
                                                      +---------------------------------------+
```

### The Three Pillars of Linux Container Internals

#### 1. Linux Kernel Namespaces (Visibility & Boundary Isolation)
Namespaces restrict **what a process can see**. A process inside a container believes it is running on a dedicated operating system because the kernel isolates global system resources:

| Namespace | Linux Flag | What It Isolates | Security Significance |
| :--- | :--- | :--- | :--- |
| **PID** | `CLONE_NEWPID` | Process IDs | Container process sees itself as PID 1; cannot see host processes. |
| **NET** | `CLONE_NEWNET` | Network devices, routing, ports | Dedicated network stack, virtual eth interface (`veth`), localhost. |
| **MNT** | `CLONE_NEWNS` | Filesystem mount points | Container has its own isolated root filesystem view (`/`). |
| **IPC** | `CLONE_NEWIPC` | Inter-Process Communication | Isolates POSIX message queues and shared memory segments. |
| **UTS** | `CLONE_NEWUTS` | Hostname and NIS domain name | Allows container to have its own unique hostname. |
| **USER** | `CLONE_NEWUSER`| User and Group ID mappings | **Crucial:** Maps container `root` (UID 0) to unprivileged host user (UID 10001). |

#### 2. Control Groups (Cgroups — Resource Governance)
While namespaces control *what you see*, **Cgroups control how much you can use**. Cgroups prevent Denial of Service (DoS) attacks by enforcing strict resource ceilings:
- **`cpu` / `cpu.max`:** Caps CPU bandwidth (prevents crypto-mining exhaustion).
- **`memory` / `memory.max`:** Enforces hard RAM limits; triggers the Linux Out-Of-Memory (OOM) killer to terminate runaway container processes.
- **`pids` / `pids.max`:** Limits the maximum number of child processes; **completely prevents Fork Bombs** (`:(){ :|:& };:`) from crashing the host node.

#### 3. OverlayFS (Union Filesystem)
Containers construct filesystems by stacking read-only layers on top of each other:
- **LowerDir:** Immutable, read-only base image layers (e.g., Ubuntu OS packages, Python runtime).
- **UpperDir:** Ephemeral, read-write layer where changes made by the running container are stored.
- **MergedDir:** The unified union mount view presented to the container filesystem.
- **Copy-on-Write (CoW):** When a container modifies an existing file from a lower layer, the kernel copies the file up into the `UpperDir` before editing, keeping base image layers untouched and shared across containers.

---

## 3. Container Breakouts & Escape Mechanics

Because containers share the host kernel, any compromise of the kernel boundary allows a container process to escape directly onto the host operating system.

### Attack Vector 1: The Privileged Container Escape (`--privileged`)
- **Mechanism:** Launching a container with the `--privileged` flag disables all AppArmor/Seccomp profiles and exposes all host device nodes (`/dev`) directly into the container.
- **Exploitation:**
  ```bash
  # Inside the privileged container:
  # 1. Identify the host root disk partition
  fdisk -l
  # 2. Mount host root filesystem into container
  mkdir -p /mnt/host
  mount /dev/sda1 /mnt/host
  # 3. Chroot into the host filesystem -> INSTANT ROOT ON HOST!
  chroot /mnt/host
  ```

### Attack Vector 2: Exposed Docker Daemon Socket (`/var/run/docker.sock`)
- As examined in CI/CD security, mounting the host Docker socket gives root-equivalent control over the Docker daemon API, allowing an attacker to spin up a sibling container with host root mounts.

### Attack Vector 3: Exploiting Linux Capabilities (`CAP_SYS_ADMIN`)
- By default, Docker drops dangerous capabilities, but retains capabilities like `CAP_CHOWN`, `CAP_NET_BIND_SERVICE`. If administrators grant `CAP_SYS_ADMIN` or `CAP_SYS_PTRACE`, attackers can mount filesystems or inject code into host processes via `ptrace`.

---

## 4. Key Differences Matrix: Containers vs Virtual Machines

| Feature | Virtual Machines (VMs) | Containers (Docker / containerd) |
| :--- | :--- | :--- |
| **Isolation Boundary** | **Hardware-Level (Hypervisor / Intel VT-x)** | **Kernel-Level (Namespaces + Cgroups)** |
| **Kernel Instance** | Dedicated guest kernel per VM | **Shared single host Linux kernel** |
| **Startup Time** | Minutes (boots full operating system) | Milliseconds (executes single process) |
| **Memory Footprint** | Gigabytes per VM | Megabytes per container |
| **Kernel Exploit Blast Radius**| Contained to Guest VM (Hypervisor escape rare)| **Host-Wide Compromise** (Kernel exploit breaks all) |
| **Storage Architecture**| Fixed virtual disk images (`.qcow2`, `.vmdk`)| Layered Union Filesystem (OverlayFS) |

---

## 5. Defensive Hardening: Seccomp and AppArmor

### 1. Seccomp-BPF (Secure Computing Mode)
- **Mechanism:** Filters Linux kernel system calls available to container processes.
- The Linux kernel exposes over 300 system calls. The default Docker seccomp profile blocks approximately 44 dangerous syscalls (e.g., `reboot`, `kexec_load`, `ptrace`, `bpf`), preventing containers from triggering known kernel vulnerabilities.

### 2. User Namespaces (`userns-remap`)
- Solves the classic dilemma: inside the container, the process runs as `root` (UID 0) to bind ports or read configuration.
- With User Namespaces enabled, the kernel maps container UID 0 to an unprivileged high UID on the host (e.g., UID 100000). If an attacker breaks out of the container filesystem, they land on the host with **zero root privileges**!

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Containers and Virtual Machines represent two fundamentally different isolation models. VMs virtualize hardware via hypervisors, running dedicated guest kernels that provide strong hardware-enforced security boundaries at the expense of memory and startup latency. In contrast, containers are standard Linux processes sharing the host kernel, isolated by Namespaces for visibility boundaries (PID, NET, MNT, USER), Cgroups for resource limits (CPU, memory, process counts), and OverlayFS for layered copy-on-write storage. Because containers share the kernel, security depends on defense-in-depth: never running containers with `--privileged`, dropping Linux capabilities, enforcing User Namespaces to map root to unprivileged host UIDs, and filtering kernel syscalls via Seccomp-BPF."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Saying "containers have their own lightweight operating system". *Correction:* Containers have their own root filesystem packages (binaries/libs), but they have **no operating system kernel of their own**; they share the host kernel.
- **Trap 2:** Assuming root inside a container cannot harm the host. *Correction:* Without User Namespaces enabled, root (UID 0) inside the container is the exact same cryptographic root (UID 0) on the host kernel!
- **Trap 3:** Running containers with `--privileged` in production. *Correction:* `--privileged` disables all security filters and grants full `/dev` hardware access, making container escape trivial.

### Expected Follow-Up Questions
1. *What happens when a container hits its cgroup memory limit?*
   - The Linux kernel OOM (Out-of-Memory) killer invokes `oom_kill_process`, immediately terminating the container process (often returning Exit Code 137: 128 + SIGKILL 9).
2. *Why are User Namespaces (`userns`) not enabled by default in standard Docker?*
   - Because user mapping can break file permissions when mounting host volumes that expect specific host UIDs/GIDs, requiring deliberate permission planning.
