# Nmap Scanning Techniques, Port States & Packet Mechanics

> **Domain:** Networking Fundamentals & Network Security  
> **Sub-Domain:** Network Reconnaissance & Port Scanning  
> **Interview Importance:** Very High / Standard Technical Round Interview Topic  

---

## 1. Topic & Definitions

- **Network Scanning:** The automated reconnaissance process of probing IP addresses and transport-layer ports to identify active hosts, open services, software versions, and operating system implementations.
- **Nmap (Network Mapper):** The open-source industry-standard network exploration and security auditing tool developed by Gordon Lyon (Fyodor).
- **The Core Objective:** Determining which network ports are reachable and listening on a target, and identifying the exact software running behind them.

---

## 2. The 6 Official Nmap Port States

When Nmap scans a target port, it categorizes it into one of six distinct states based on the returned response (or lack thereof):

| Port State | Technical Definition & Response Received | Underlying Meaning |
| :--- | :--- | :--- |
| **`Open`** | Target accepts connection: replies with **`SYN-ACK`** (TCP) or application response (UDP). | An application/service is actively listening and ready to accept connections. |
| **`Closed`** | Target host replies with a **`RST` (Reset)** packet (TCP) or **ICMP Port Unreachable Type 3 Code 3** (UDP). | The host is reachable and active, but **no service is listening** on that port. |
| **`Filtered`** | **No response received (Timeout)** or host replies with ICMP Destination Unreachable (Type 3 Code 1, 2, 9, 10, or 13). | A **firewall, network filter, or router ACL** is blocking the probe and dropping packets. |
| **`Unfiltered`** | Target responds to probe (e.g. replies with `RST` to an ACK probe), but Nmap cannot determine if open or closed. | Port is accessible through the firewall, but open/closed status requires further scanning. |
| **`Open\|Filtered`** | Target returns zero response to a UDP, FIN, NULL, or Xmas probe. | Nmap cannot determine whether the port is open (which drops probe) or filtered by a firewall. |
| **`Closed\|Filtered`**| Returned during IP ID idle scans when inconclusive. | State cannot be resolved. |

---

## 3. TCP Scan Types & Packet Handshake Mechanics

```text
TCP CONNECT SCAN (-sT): Complete 3-Way Handshake
Attacker                                 Target (Open Port)
   │ ────────── 1. SYN ────────────────────────► │
   │ ◄───────── 2. SYN-ACK ──────────────────── │
   │ ────────── 3. ACK (Connection Established!)► │ (Logged by Web/App server!)
   │ ────────── 4. RST (Teardown) ─────────────► │

TCP SYN STEALTH SCAN (-sS): Half-Open Scan (Default for Root)
Attacker                                 Target (Open Port)
   │ ────────── 1. SYN ────────────────────────► │
   │ ◄───────── 2. SYN-ACK ──────────────────── │
   │ ────────── 3. RST (Immediately Aborts!) ──► │ (Never reaches app layer! Zero app logs!)
```

### 1. TCP Connect Scan (`-sT`)
- **How It Works:** Calls the operating system's standard `connect()` system call to establish a complete, formal TCP 3-way handshake.
- **Pros:** Can be run by **unprivileged standard users** (does not require root/administrator).
- **Cons:** Slower, noisy, and heavily logged: because the connection reaches `ESTABLISHED` state, the application layer registers a connection event in server logs.

### 2. TCP SYN Stealth Scan (`-sS`)
- **How It Works:** Operates via raw sockets (requires `root` on Linux or Administrator on Windows). Sends raw SYN packets. If the target replies with `SYN-ACK`, Nmap knows the port is open, immediately sends a `RST` to abort, and never completes the handshake.
- **Pros:** **Stealthy:** Most application daemons (web servers, FTP, SSH) only log connections that successfully complete the handshake. Does not trigger application-level log generation.

### 3. UDP Scan (`-sU`)
- **How It Works:** Sends empty UDP packets (or service-specific payloads like DNS/SNMP) to target ports.
  - If target replies with **ICMP Port Unreachable (Type 3 Code 3)** $\implies$ Port is **Closed**.
  - If target replies with a UDP packet $\implies$ Port is **Open**.
  - If target returns **no response** $\implies$ Port is marked **`Open|Filtered`** (because packet could have reached a silent open service or dropped by a firewall).
- **Challenge:** Very slow because operating systems (Linux kernel) rate-limit ICMP error responses (e.g. 1 per second).

### 4. Advanced Evasion Scans: FIN, NULL & Xmas Scans
Exploit a subtle loophole in the official **RFC 793 TCP Specification**:
> *"If the destination port state is CLOSED, an incoming segment not containing a RST causes a RST to be sent in response. If the port is OPEN, the incoming segment should be dropped and ignored."*

- **NULL Scan (`-sN`):** Sends a packet with **zero flags set** (`000000`).
- **FIN Scan (`-sF`):** Sends a packet with **only the FIN flag set**.
- **Xmas Scan (`-sX`):** Sends a packet with **FIN, PSH, and URG flags all set** simultaneously.
- **Rule Evaluation:**
  - If target port is **Closed** $\implies$ Target replies with **RST**.
  - If target port is **Open** $\implies$ Target **drops packet with no response**.
- **Limitation:** Does not work against Microsoft Windows systems! Windows non-compliantly sends `RST` for all ports, regardless of state. Works primarily against Unix/Linux systems.

---

## 4. Essential Nmap Command Combinations

```bash
# 1. Comprehensive Service & Vulnerability Audit (The Pentester Standard):
nmap -sS -sV -sC -O -p- -T4 10.0.0.5
# Flags breakdown:
# -sS : SYN Stealth Scan
# -sV : Version Detection (determines daemon name and exact version string)
# -sC : Run default Nmap Scripting Engine (NSE) scripts
# -O  : Operating System fingerprinting
# -p- : Scan ALL 65,535 ports (instead of default top 1,000)
# -T4 : Timing Template 4 (Aggressive speed for fast networks)

# 2. Fast Network Host Discovery (Ping Sweep without port scanning):
nmap -sn 192.168.1.0/24

# 3. Scanning for Known Vulnerabilities via NSE Scripts:
nmap -p 80,443,445 --script vuln 10.0.0.5

# 4. Evading Firewalls via Fragmentation & Decoys:
nmap -sS -f -D RND:5 10.0.0.5
# -f : Fragment probe packets into tiny 8-byte fragments to evade simple ACLs
# -D : Decoy scan (spoofs 5 random IP addresses alongside real IP to hide attacker)
```

---

## 5. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Nmap is the industry-standard network reconnaissance tool used to discover active hosts and open ports across six states: Open (service listening), Closed (replies with RST), Filtered (packet dropped by firewall with timeout), Unfiltered, and composite states. The primary scanning techniques are the TCP Connect scan (`-sT`), which establishes a complete 3-way handshake via standard OS system calls and gets recorded in application logs, and the TCP SYN Stealth scan (`-sS`), which sends raw SYN packets and tears down connections with an immediate RST upon receiving a SYN-ACK, remaining invisible to application loggers. Advanced techniques include UDP scanning via ICMP unreachable messages, and RFC 793 scans like NULL, FIN, and Xmas scans that exploit closed-port RST responses to slip past stateless packet filters."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing "Closed" and "Filtered" port states.  
  *Correction:* **Closed** means the packet reached the target OS, but no application was listening (target actively sent back a `RST`). **Filtered** means the packet never reached the target OS because a firewall dropped it without responding.
- **Trap:** Claiming SYN scan is completely undetectable.  
  *Correction:* While a SYN scan does not trigger *application-layer* logs (e.g. Apache access logs), it is instantly detected by **Network Intrusion Detection Systems (NIDS)**, stateful firewalls, and EDRs watching for anomalous half-open TCP connection spikes.
