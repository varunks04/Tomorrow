# Client-Server TCP Connection and HTTPS Handshake Lifecycle

## 1. Topic & Definition
The foundation of secure network communications is the dual-handshake architecture: establishing a reliable Layer-4 transport session via the **Transmission Control Protocol (TCP — RFC 793)**, followed immediately by establishing an encrypted, authenticated presentation session via **Transport Layer Security (TLS 1.3 — RFC 8446)**.

Mastery of this lifecycle requires tracking state transitions, sequence number arithmetic, congestion control algorithms, cryptographic key exchanges, and connection teardown states (`TIME_WAIT`).

---

## 2. Protocol Handshakes: Mathematical and Packet Mechanics

```
+---------------------------------------------------------------------------------------------------+
|                         Complete TCP and TLS 1.3 Handshake Protocol Flow                          |
+---------------------------------------------------------------------------------------------------+

   CLIENT (Source Port: 54128)                                SERVER (Listening Port: 443)
       |                                                                  |
       |  === PHASE 1: TCP THREE-WAY HANDSHAKE (Reliable Transport) ===   |
       |                                                                  |
       |--- 1. [SYN] Seq=1000, Win=65535, MSS=1460 ---------------------->| (State: SYN_SENT -> SYN_RCVD)
       |<-- 2. [SYN, ACK] Seq=5000, Ack=1001, Win=65535, MSS=1460 -------|
       |--- 3. [ACK] Seq=1001, Ack=5001 --------------------------------->| (State: ESTABLISHED)
       |                                                                  |
       |  === PHASE 2: TLS 1.3 CRYPTOGRAPHIC HANDSHAKE (1-RTT) ===        |
       |                                                                  |
       |--- 4. ClientHello (ClientRandom, SupportedCiphers, KeyShare_C) ->|
       |<-- 5. ServerHello (ServerRandom, SelectedCipher, KeyShare_S) ---|
       |       {EncryptedExtensions}                                      | (All subsequent data
       |       {Certificate} + {CertificateVerify}                        |  is encrypted using
       |       {Finished}                                                 |  Handshake Keys!)
       |--- 6. {Finished} ----------------------------------------------->|
       |                                                                  |
       |  === PHASE 3: ENCRYPTED APPLICATION DATA EXCHANGE ===            |
       |                                                                  |
       |--- 7. Application Data (HTTP GET / Encrypted AES-256-GCM) ------>|
       |<-- 8. Application Data (HTTP 200 OK / Encrypted) ----------------|
```

### A. TCP Sequence & Acknowledgement Number Arithmetic
- TCP treats data as an un-delimited stream of bytes.
- The **Initial Sequence Number (ISN)** is generated randomly by the OS kernel to prevent sequence prediction attacks.
- **The Rule:** The `ACK` number always equals **the next byte expected from the sender**:
  - Client sends `SYN` with $Seq = 1000$ (consumes 1 phantom byte).
  - Server replies with `SYN-ACK`: sets $Ack = 1001$ and its own $Seq = 5000$.
  - Client replies with `ACK`: sets $Seq = 1001$ and $Ack = 5001$.

### B. TCP Flow Control vs. Congestion Control
1. **Flow Control (End-to-End):** Prevents the fast sender from overwhelming a slow receiver. Governed by the **Receive Window (`Win` / RWIN)** advertised in the TCP header: the sender may not transmit more unacknowledged bytes than the receiver's available buffer capacity.
2. **Congestion Control (Network Health):** Prevents the sender from overwhelming intermediate network routers:
   - **Slow Start:** Starts with a small Congestion Window (cwnd = 10 MSS); doubles `cwnd` every RTT exponentially.
   - **Congestion Avoidance:** When `cwnd` hits Slow Start Threshold (`ssthresh`), transitions to linear growth (+1 MSS per RTT).
   - **Modern Algorithms:** Google **BBR (Bottleneck Bandwidth and RTT)** optimizes throughput based on estimated network bottleneck capacity rather than packet loss (unlike legacy Reno or Cubic).

---

## 3. TCP Teardown Mechanics: The 4-Way FIN Handshake and `TIME_WAIT`

Terminating a TCP connection gracefully requires closing both half-duplex directions:

```
CLIENT (Initiates Close)                                SERVER
   |                                                       |
   |--- 1. [FIN, ACK] Seq=1500, Ack=6000 ----------------->| (Client: FIN_WAIT_1 -> Server: CLOSE_WAIT)
   |<-- 2. [ACK] Seq=6000, Ack=1501 -----------------------| (Client: FIN_WAIT_2)
   |                                                       |
   |<-- 3. [FIN, ACK] Seq=6000, Ack=1501 ------------------| (Server: LAST_ACK -> Client: TIME_WAIT)
   |--- 4. [ACK] Seq=1501, Ack=6001 ---------------------->| (Server: CLOSED)
   |                                                       |
   | [ Client waits 2MSL (Maximum Segment Lifetime = 60s) ]|
   v                                                       v
(Client: CLOSED)
```

### Why the `TIME_WAIT` State Exists:
1. **Reliable Final ACK Delivery:** If the client's final ACK (packet 4) is dropped on the network, the server retransmits its `FIN`. If the client closed immediately, it would respond with an error `RST` instead of re-sending the ACK.
2. **Lingering Packet Drainage:** Ensures delayed, duplicate packets from the closed connection fully expire from intermediate internet routers before a new connection reuses the same 4-tuple (Src IP, Src Port, Dst IP, Dst Port).

---

## 4. Key Differences Matrix: TLS 1.2 vs TLS 1.3

| Feature | TLS 1.2 (Legacy RFC 5246) | TLS 1.3 (Modern RFC 8446) |
| :--- | :--- | :--- |
| **Handshake Latency** | **2 Full RTTs** (Round Trip Times) | **1 RTT** (or 0-RTT Session Resumption) |
| **Key Exchange** | Static RSA, DHE, ECDHE | **ECDHE strictly mandatory** |
| **Perfect Forward Secrecy**| Optional (Disabled if RSA key exchange used)| **Mandatory (100% Guaranteed PFS)** |
| **Insecure Ciphers** | Permitted (RC4, DES, MD5, SHA-1, CBC mode)| **Completely Removed** |
| **Certificate Encryption** | Transmitted in cleartext plaintext | **Encrypted in flight** (Protects user privacy)|
| **0-RTT Resumption Risk**| N/A | Supported, but **vulnerable to Replay Attacks**! |

---

## 5. Cybersecurity Relevance, Threats & Diagnostic Signatures

### 1. TCP SYN Flood DDoS & SYN Cookies
- **Attack Vector:** An adversary floods a server with millions of spoofed `SYN` packets without sending the final `ACK`. The server allocates Transmission Control Blocks (TCBs) in memory for each half-open connection, rapidly exhausting server RAM and refusing legitimate users.
- **Defensive Countermeasure: SYN Cookies (RFC 4987):**
  - The server **does not allocate any memory** for half-open connections.
  - Instead, the server encodes connection state into the Initial Sequence Number ($Seq$) using a cryptographic hash:
    $$ISN = \text{HMAC}(\text{SrcIP}, \text{DstIP}, \text{SrcPort}, \text{DstPort}, \text{SecretKey}, \text{Timestamp})$$
  - When the client returns the final `ACK` ($Ack = ISN + 1$), the server recalculates the hash. If authentic, it creates the socket in memory on-the-fly!

### 2. The 0-RTT Early Data Replay Vulnerability in TLS 1.3
- **Mechanism:** In TLS 1.3, clients that previously connected can transmit application data in the very first packet (0-RTT) using cached pre-shared keys.
- **Exploitation:** Because 0-RTT data is encrypted with static resumption keys, an on-path attacker captures the packet and **replays it to the server**.
- **Impact:** If the 0-RTT packet contains an idempotent action (e.g., `GET /index.html`), it is harmless. If it contains a state-changing operation (e.g., `POST /transfer?amount=1000`), the bank executes the transaction twice!
- **Defense:** Never permit non-idempotent HTTP methods (POST, PUT, DELETE) in 0-RTT early data.

### 3. Connection Reset (`RST`) Scenarios
- **RST received immediately on SYN:** Port is closed (no application listening) or dropped by a host firewall.
- **RST received mid-session:** Stateful firewall connection timeout, or IDS/IPS actively injecting TCP RST packets to terminate a malicious connection.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Network communications rely on establishing transport reliability before negotiating cryptographic trust. The TCP 3-way handshake initializes synchronization via random ISNs, negotiating flow control windows and maximum segment sizes, while mitigating SYN flood exhaustion through stateless SYN Cookies. Once established, TLS 1.3 negotiates security in a single round-trip (1-RTT), mandating ephemeral Diffie-Hellman (ECDHE) for non-negotiable Perfect Forward Secrecy while eliminating insecure legacy ciphers. On teardown, the 4-way FIN handshake transitions the initiating endpoint into `TIME_WAIT` for 2MSL to ensure lingering packets drain and final ACKs arrive safely. Understanding this lifecycle enables precise root-cause diagnostics when analyzing Wireshark traces for connection resets, packet drops, or handshake stalls."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Believing TLS 1.3 requires 2 round-trip times. *Correction:* TLS 1.3 optimizes the handshake to 1-RTT by having the client send its Diffie-Hellman key share speculatively inside the ClientHello.
- **Trap 2:** Confusing Flow Control with Congestion Control. *Correction:* Flow Control protects the receiver buffer (advertised window); Congestion Control protects intermediate network links (congestion window).
- **Trap 3:** Assuming `TIME_WAIT` is a bug or memory leak. *Correction:* `TIME_WAIT` is a required, healthy protocol state ensuring duplicate packets drain from the network and final ACKs are acknowledged.

### Expected Follow-Up Questions
1. *What causes a "Connection Refused" error in Linux?*
   - The client sent a TCP `SYN` packet to a port on the server where no application was bound and listening, causing the server kernel to respond with a TCP `RST, ACK` packet.
2. *Why is Static RSA key exchange considered obsolete in modern TLS?*
   - Because RSA key exchange lacks Perfect Forward Secrecy (PFS). If an adversary records encrypted network traffic for months and later steals the server's private RSA key, they can retroactively decrypt all recorded historical sessions.
