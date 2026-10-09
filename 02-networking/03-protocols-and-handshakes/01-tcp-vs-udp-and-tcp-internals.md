# TCP vs. UDP Internals, Handshakes, States & Flag Attacks

> **Domain:** Networking Fundamentals  
> **Sub-Domain:** Transport Layer Protocols & State Machines  
> **Interview Importance:** Critical / High-Frequency Technical Round Question  

---

## 1. Topic & Definitions

- **Transmission Control Protocol (TCP - RFC 793):** A connection-oriented, reliable transport protocol that guarantees ordered, error-checked, flow-controlled, and congestion-controlled delivery of a stream of octets between processes.
- **User Datagram Protocol (UDP - RFC 768):** A minimalist, connectionless transport protocol providing best-effort delivery of discrete datagrams with minimal protocol overhead, zero connection state, and no delivery guarantees.
- **TCP Flags (Control Bits):** 1-bit flags in the 20-byte TCP header that control the state machine:
  - **`SYN` (Synchronize):** Initiates a connection and synchronizes initial sequence numbers.
  - **`ACK` (Acknowledgment):** Confirms receipt of transmitted data or handshake segments.
  - **`FIN` (Finish):** Gracefully terminates a connection in one direction.
  - **`RST` (Reset):** Abruptly aborts and tears down an abnormal connection.
  - **`PSH` (Push):** Instructs receiver to immediately push buffered data to the application layer.
  - **`URG` (Urgent):** Indicates urgent out-of-band data (handled via urgent pointer).

---

## 2. The TCP 3-Way Handshake & 4-Way Teardown

```text
TCP THREE-WAY CONNECTION ESTABLISHMENT:
Client (State)                                           Server (State)
[CLOSED]                                                 [LISTEN]
   │                                                         │
   ├─────── Step 1: SYN (seq = ISN_c) ──────────────────────►│ [SYN_RCVD]
   │        (Allocates client socket buffers)                │ (Allocates server backlog buffer)
   │                                                         │
[SYN_SENT]                                                   │
   │◄────── Step 2: SYN-ACK (seq = ISN_s, ack = ISN_c + 1) ──┤
   │                                                         │
[ESTABLISHED]                                                │
   ├─────── Step 3: ACK (ack = ISN_s + 1) ──────────────────►│ [ESTABLISHED]
   │                                                         │
   ▼                                                         ▼
   ═════════════════ DATA TRANSFER PHASE ═════════════════════
```

```text
TCP FOUR-WAY CONNECTION TERMINATION:
Client                                                   Server
[ESTABLISHED]                                            [ESTABLISHED]
   │                                                         │
   ├─────── Step 1: FIN (seq = u) ──────────────────────────►│ [CLOSE_WAIT]
[FIN_WAIT_1]                                                 │ (App notified: client done sending)
   │◄────── Step 2: ACK (ack = u + 1) ───────────────────────┤
[FIN_WAIT_2]                                                 │ Server finishes flushing pending data...
   │                                                         │
   │◄────── Step 3: FIN (seq = v) ───────────────────────────┤ [LAST_ACK]
   │                                                         │
   ├─────── Step 4: ACK (ack = v + 1) ──────────────────────►│ [CLOSED]
[TIME_WAIT]                                                  │
   │ (Waits 2 * MSL = 60 to 120 seconds before closing)      │
[CLOSED]                                                     │
```

### Why Does `TIME_WAIT` Exist?
1. **Ensures the final ACK is received:** If the client's final ACK is lost, the server retransmits its FIN. The client must stay in `TIME_WAIT` to resend the ACK.
2. **Drains lingering duplicate packets:** Prevents delayed packets from an old closed connection from accidentally corrupting a newly spawned connection reusing the same port tuple.

---

## 3. TCP Reliability Mechanisms: Sequence, ACKs, Flow & Congestion Control

1. **Sequencing and Acknowledgments:** Every transmitted byte has a Sequence Number. The receiver sends an ACK number indicating the **next expected byte** (Cumulative ACK). Lost packets are detected via retransmission timeouts (RTO) or **Triple Duplicate ACKs** (Fast Retransmit).
2. **Flow Control (Sliding Window):** Prevents a fast sender from overwhelming a slow receiver. The receiver advertises a **Receive Window (`rwnd`)** in every TCP header indicating available buffer capacity in bytes.
3. **Congestion Control:** Prevents the network fabric from collapsing under overload:
   - **Slow Start:** Exponentially increases Congestion Window (`cwnd`) every round-trip time ($1 \to 2 \to 4 \to 8$).
   - **Congestion Avoidance:** Linearly increases `cwnd` once reaching slow-start threshold (`ssthresh`).
   - **Reaction to Packet Loss:** Halves or drops `cwnd` to 1 upon detecting packet drops.

---

## 4. Key Differences: TCP vs. UDP

| Evaluation Feature | TCP | UDP |
| :--- | :--- | :--- |
| **Connection State** | Connection-oriented (Handshake required). | Connectionless (No handshake; zero state). |
| **Reliability** | Guaranteed delivery (Retransmissions, sequencing).| Best-effort (Packets can be lost, duplicated, out of order). |
| **Header Size** | 20 to 60 bytes. | Fixed strictly at **8 bytes**. |
| **Speed & Overhead**| Slower due to handshakes, state tracking, and ACKs. | Blisteringly fast; minimal latency. |
| **Streaming Style** | Continuous byte stream (No record boundaries). | Discrete Datagrams (Message boundaries preserved). |
| **Broadcast / Multicast**| **Unicast only** (Cannot broadcast/multicast). | Supports Unicast, Broadcast, and Multicast. |
| **Common Protocols** | HTTP, HTTPS, SSH, FTP, SMTP, MySQL. | DNS queries, DHCP, SNMP, NTP, VoIP, Live Video. |

---

## 5. Cybersecurity Relevance & Threat Vectors

### 1. SYN Flood Attack (DoS) & The SYN Cookie Defense
- **The Attack:** An attacker sends millions of TCP SYN packets with forged, random source IP addresses. For each SYN, the server allocates TCB (Transmission Control Block) socket memory in its **SYN Backlog Queue** and waits for the final ACK. Because the source IPs are fake, the ACK never arrives. The backlog queue fills up, causing the server to reject all legitimate user connections.
- **The Golden Defense: SYN Cookies:**
  - The server **does NOT allocate any memory or state** upon receiving a SYN!
  - Instead, it encodes the connection details into the Initial Sequence Number ($ISN_s$) using a cryptographic hash:
    $$ISN_s = \text{HMAC}(SrcIP, DstIP, SrcPort, DstPort, SecretKey) + Timestamp$$
  - When the client returns the final ACK with $ack = ISN_s + 1$, the server recomputes the hash to verify authenticity. Only then does it allocate memory!

### 2. TCP Reset (RST) Attacks
- If an attacker can sniff network traffic and determine the current sequence numbers, they inject a spoofed packet with the `RST` flag set. Both endpoints immediately tear down the active connection. Used by censorship engines (e.g. Great Firewall of China) and malicious actors to terminate active SSH or VPN tunnels.

### 3. UDP Amplification DDoS Attacks
- Because UDP has **zero handshake and no source verification**, attackers spoof the victim's IP address as the UDP source IP and send small requests to open public reflectors:
  - **DNS Amplification:** Send 60-byte `ANY` query $\to$ Reflects 3,000-byte response to victim (50x amplification).
  - **NTP `monlist`:** Send 234-byte request $\to$ Reflects 48 KB response (200x amplification).

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"TCP is a connection-oriented, reliable transport protocol that uses a 3-way handshake (SYN, SYN-ACK, ACK) to synchronize sequence numbers and establish state before transmitting data. It guarantees ordered delivery, retransmits lost segments, and manages traffic flow via sliding windows and congestion control algorithms. In contrast, UDP is a lightweight, connectionless datagram protocol with an 8-byte header, prioritizing speed and low latency over reliability. In cybersecurity, TCP's connection state makes it vulnerable to SYN flood resource exhaustion—which we mitigate using stateless SYN Cookies—while UDP's lack of source validation makes it the primary vector for reflection and amplification DDoS attacks."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing Flow Control with Congestion Control.  
  *Correction:* **Flow Control** protects the *receiving endpoint's buffer* (`rwnd`). **Congestion Control** protects the *intermediate network routers and links* from collapsing (`cwnd`).
- **Trap:** Forgetting why the 3-way handshake needs 3 steps instead of 2.  
  *Correction:* Two steps (SYN, SYN-ACK) allow the client to confirm the server is ready and knows the client's ISN. But the server still does not know if the client received the server's ISN until the **third step (ACK)** completes the loop.
