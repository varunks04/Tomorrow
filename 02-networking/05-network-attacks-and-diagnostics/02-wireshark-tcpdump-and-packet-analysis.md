# Packet Analysis Mastery: Wireshark, tcpdump & Anomaly Detection

> **Domain:** Networking Fundamentals & Security Operations  
> **Sub-Domain:** Packet Capture, Protocol Analysis & Forensics  
> **Interview Importance:** Very High / Hands-on Technical Round & SOC Analysis  

---

## 1. Topic & Definitions

- **Packet Capture (PCAP):** The process of intercepting and logging raw data frames traversing a network interface. Files are saved in standard formats (`.pcap` or modern `.pcapng`).
- **Promiscuous Mode:** A configuration setting on a Network Interface Card (NIC) causing the controller to pass all frames received on the physical media to the CPU/OS, rather than passing only frames addressed to its own MAC address or broadcast.
- **Berkeley Packet Filter (BPF):** A raw bytecode technology in Unix/Linux kernels allowing user-space tools (`tcpdump`, Wireshark) to filter packets directly inside the kernel before they are copied to user space, maximizing capture performance.
- **Capture Filter vs. Display Filter:**
  - **Capture Filter (BPF):** Evaluated by the kernel **before** saving packets to disk. Cannot be changed during capture. Packets dropped here are permanently discarded.
  - **Display Filter:** Evaluated by Wireshark in memory **after** packets are captured. Allows interactive slicing, searching, and protocol decoding.

---

## 2. Capture Filters (BPF) vs. Display Filters (Wireshark)

| Feature | Capture Filter (BPF) | Display Filter (Wireshark GUI) |
| :--- | :--- | :--- |
| **When Applied** | At capture time (Kernel level). | Post-capture (User-space memory). |
| **Syntax Style** | Primitive BPF syntax: `host 10.0.0.5 and port 80`. | Rich protocol syntax: `ip.addr == 10.0.0.5 && tcp.port == 80`.|
| **Resource Impact** | Minimizes CPU and disk usage by dropping irrelevant packets early. | Heavy RAM consumption if full captures are loaded. |
| **Flexibility** | Rigid: cannot filter by deep application attributes (like HTTP URI or cookie). | Deeply flexible: can inspect any field in the dissected protocol tree. |

---

## 3. Essential tcpdump Commands & BPF Filters

```bash
# Core tcpdump Flags
tcpdump -i eth0                     # Listen on interface eth0
tcpdump -nn                         # Don't resolve hostnames (-n) or port numbers (-nn) -> Faster!
tcpdump -X                          # Print packet data in both Hex and ASCII format
tcpdump -s 0                        # Capture full packet length (snaplen = 0 means entire packet)
tcpdump -w capture.pcap             # Write raw packet capture to file
tcpdump -r capture.pcap             # Read and analyze an existing PCAP file
tcpdump -c 100                      # Stop automatically after capturing 100 packets

# Practical BPF Filter Expressions
tcpdump -i eth0 'tcp port 80 or tcp port 443'       # Capture web traffic only
tcpdump -i eth0 'src net 192.168.1.0/24'             # Capture traffic from specific subnet
tcpdump -i eth0 'not port 22'                        # Exclude SSH traffic (prevents capture feedback loop!)

# Isolating Specific TCP Flags via Bitmasking
# 1. Capture TCP SYN Packets Only (Detecting incoming connections / SYN scans):
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0 and tcp[tcpflags] & (tcp-ack) == 0'

# 2. Capture TCP RST Packets Only (Detecting aborted connections):
tcpdump -i eth0 'tcp[tcpflags] & (tcp-rst) != 0'
```

---

## 4. Essential Wireshark Display Filters for Incident Triage

| Diagnostic Goal | Wireshark Display Filter Expression |
| :--- | :--- |
| **Filter by IP Address** | `ip.addr == 192.168.1.50` or `ip.src == 10.0.0.1 && ip.dst == 10.0.0.2` |
| **Filter by Port** | `tcp.port == 443 || udp.port == 53` |
| **Show HTTP POST Requests** | `http.request.method == "POST"` |
| **Inspect HTTP Cleartext Passwords**| `http contains "password" || http contains "admin"` |
| **Filter Failed DNS Queries** | `dns.flags.rcode != 0` (Non-zero response code indicates NXDOMAIN/failure) |
| **Isolate TCP 3-Way Handshake**| `tcp.flags.syn == 1` |
| **Identify Retransmissions** | `tcp.analysis.retransmission` (Indicates packet loss or network congestion) |
| **Isolate Out-of-Order Packets** | `tcp.analysis.out_of_order` |
| **Detect ARP Anomalies** | `arp.duplicate-address-frame` (Signals possible ARP cache poisoning!) |

---

## 5. Identifying Cyber Attacks in Wireshark

### 1. Detecting Port Scans (Nmap Probing)
- **SYN Stealth Scan (`-sS`):** Look for thousands of SYN packets sent to consecutive destination ports (`21, 22, 23, 25, 80...`) within milliseconds from the same source IP, followed by an immediate RST upon receiving a SYN-ACK.
- **Xmas Scan (`-sX`):** Packets with `URG`, `PSH`, and `FIN` flags all illuminated simultaneously ("lit up like a Christmas tree"):
  `tcp.flags == 0x029` (00101001 binary). Legitimate operating systems never generate this packet combination!
- **NULL Scan (`-sN`):** Packets with zero flags set: `tcp.flags == 0x000`.

### 2. Spotting DNS Tunneling
- Filter: `dns`. Look for hundreds of outbound DNS queries targeting subdomains of the same parent domain with long, randomized, high-entropy character strings:
  `query: 7a8b9c0d1e2f3a4b.c2.attacker.com`
  `query: 9f8e7d6c5b4a3a2b.c2.attacker.com`

### 3. Reconstructing Conversations ("Follow TCP Stream")
- Right-click any TCP packet in Wireshark $\to$ **Follow $\to$ TCP Stream**.
- Wireshark reassembles all fragmented segments, eliminates duplicate sequence numbers, and displays the complete bidirectional conversation in a single ASCII/Hex window:
  - Client data displayed in **Red**.
  - Server response displayed in **Blue**.
- Used by analysts to extract cleartext passwords, dumped SQL queries, and downloaded malware binaries.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Packet analysis provides ground truth visibility into network operations. In production and incident triage, we use tcpdump on remote headless Linux servers to record traffic using Berkeley Packet Filters (BPF) before saving to PCAP files, and Wireshark for deep interactive protocol dissection. The key architectural distinction is that Capture Filters operate in the kernel to prevent resource exhaustion, while Display Filters operate in user space to slice analyzed sessions. Analysts identify attacks by isolating protocol flags—such as detecting SYN floods via anomalous SYN-without-ACK spikes, uncovering stealth scans through illegal flag combinations like Xmas scans (`tcp.flags == 0x029`), and reconstructing compromised application sessions using Wireshark's Follow TCP Stream feature."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Forgetting to exclude SSH when running tcpdump over an SSH terminal session.  
  *Correction:* If you run `tcpdump -i eth0` while connected via SSH, every packet displayed on your terminal generates new SSH packets, creating an infinite recursive feedback loop that freezes the terminal! Always include `not port 22`.
- **Trap:** Confusing BPF syntax with Wireshark syntax.  
  *Correction:* BPF capture syntax uses `port 80` or `host 10.0.0.1`. Wireshark display syntax uses `tcp.port == 80` or `ip.addr == 10.0.0.1`. Using Wireshark display syntax inside tcpdump will cause an immediate syntax error.
