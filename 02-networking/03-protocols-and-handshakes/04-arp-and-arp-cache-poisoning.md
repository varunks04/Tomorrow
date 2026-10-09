# Address Resolution Protocol (ARP), Cache Poisoning & Dynamic ARP Inspection

> **Domain:** Networking Fundamentals  
> **Sub-Domain:** Link Layer Resolution & Local Network Security  
> **Interview Importance:** Very High / Classic Man-in-the-Middle Technical Scenario  

---

## 1. Topic & Definitions

- **Address Resolution Protocol (ARP - RFC 826):** A fundamental link-layer protocol (operating conceptually between Layer 2 and Layer 3) responsible for mapping an known Layer 3 IPv4 address to its corresponding physical Layer 2 Media Access Control (MAC) address on a local broadcast domain.
- **EtherType:** ARP packets are encapsulated directly inside Ethernet frames with EtherType **`0x0806`**.
- **ARP Cache / Table:** A local memory table maintained by the operating system kernel storing recent IP-to-MAC address mappings to eliminate the overhead of sending an ARP broadcast for every individual network packet.
- **Gratuitous ARP (GARP):** An unsolicited ARP broadcast message sent by a host announcing its own IP-to-MAC mapping without being queried.

---

## 2. Standard ARP Request and Reply Flow

When Host A (`192.168.1.10`) wants to send an IP packet to Host B (`192.168.1.20`) on the same local subnet:

```text
Host A (192.168.1.10)                                      Host B (192.168.1.20)
MAC: AA:AA:AA:AA:AA:AA                                     MAC: BB:BB:BB:BB:BB:BB
         │                                                            │
         ├────── Step 1: ARP Request (BROADCAST) ────────────────────►│
         │       "Who has 192.168.1.20? Tell 192.168.1.10"           │
         │       (Dest MAC: FF:FF:FF:FF:FF:FF - Received by all hosts) │
         │                                                            │
         │◄───── Step 2: ARP Reply (UNICAST) ─────────────────────────┤
         │       "192.168.1.20 is at BB:BB:BB:BB:BB:BB"               │
         ▼       (Dest MAC: AA:AA:AA:AA:AA:AA)                        ▼
Host A updates ARP Cache:                                  Host B updates ARP Cache:
[192.168.1.20 -> BB:BB:BB:BB:BB:BB]                       [192.168.1.10 -> AA:AA:AA:AA:AA:AA]
```

### Gratuitous ARP (Legitimate Operational Uses)
1. **IP Address Conflict Detection:** When a machine boots up with an IP address, it broadcasts a GARP for its own IP. If another host replies, the OS detects a duplicate IP conflict and alerts the user.
2. **High Availability Gateway Failover:** In protocols like HSRP, VRRP, or CARP, when the active router fails, the standby router takes over the Virtual IP (VIP) and immediately broadcasts a GARP. This updates the CAM tables of all switches and the ARP caches of all hosts to point to the new router's physical MAC address.

---

## 3. The Vulnerability: ARP Cache Poisoning (Man-in-the-Middle)

### The Inherent Flaw in ARP
**ARP is completely stateless and has zero authentication.**  
Operating systems accept and update their ARP cache upon receiving **ANY** ARP reply, even if they never sent an ARP request!

### Attack Mechanics Step-by-Step

```text
[ Victim Client ]                   [ Attacker ]                   [ Default Gateway ]
IP: 192.168.1.50                 IP: 192.168.1.100                  IP: 192.168.1.1
MAC: AA:AA:AA:AA:AA:AA           MAC: CC:CC:CC:CC:CC:CC             MAC: GG:GG:GG:GG:GG:GG
        │                               │                                     │
        │◄── 1. Spoofed ARP Reply ──────┤                                     │
        │    "192.168.1.1 is at CC:CC"  │                                     │
        │    (Victim poisoned!)         │                                     │
        │                               ├────── 2. Spoofed ARP Reply ────────►│
        │                               │       "192.168.1.50 is at CC:CC"    │
        │                               │       (Gateway poisoned!)           │
        │                               │                                     │
        ├────── 3. Outbound Internet Packet (Destined for Gateway) ──────────►│
        │       (Delivered to Attacker's MAC: CC:CC)                          │
        │                               ├─ Sniffs / logs credentials          │
        │                               ├─ Forwards packet to genuine Gateway ┤
        ▼                               ▼                                     ▼
```

### Consequences of Successful ARP Poisoning:
1. **Full Man-in-the-Middle (MitM):** The attacker inspects, modifies, or logs all unencrypted HTTP, DNS, FTP, and Telnet traffic flowing between the victim and the Internet.
2. **SSL Stripping:** Downgrading HTTPS requests to unencrypted HTTP using tools like `sslstrip`.
3. **Denial of Service (Blackholing):** The attacker poisons the victim's ARP cache with a non-existent MAC address, dropping all outbound packets.

---

## 4. The Enterprise Defense: Dynamic ARP Inspection (DAI)

To defeat ARP poisoning, enterprise Layer 2 switches deploy **Dynamic ARP Inspection (DAI)**.

```text
[ Attacker ] ── (Sends forged ARP: "192.168.1.1 is at MAC CC") ──► [ Switch Port 3 (Untrusted) ]
                                                                             │
                                                                             ▼
                                                        [ Switch inspects packet via DAI ]
                                                        Checks against DHCP Snooping Binding Table:
                                                        "Port 3 is bound to IP 192.168.1.100, NOT .1!"
                                                                             │
                                                                             ▼
                                                                [ PACKET DROPPED & LOGGED! ❌ ]
```

### DAI Verification Logic:
1. When an ARP packet arrives on an **Untrusted Port**, the switch intercepts it before forwarding.
2. The switch compares the packet's Sender IP and Sender MAC against the **DHCP Snooping Binding Table**.
3. If the IP-to-MAC pair does **not match**, the switch drops the ARP packet and increments the security violation counter.
4. For static IP servers not using DHCP, network administrators define **ARP Access Control Lists (ARP ACLs)** to validate mappings.

---

## 5. Key Differences: ARP vs. DNS vs. RARP

| Feature | ARP | DNS | RARP (Reverse ARP) |
| :--- | :--- | :--- | :--- |
| **Mapping Type** | IP Address $\to$ MAC Address | Domain Name $\to$ IP Address | MAC Address $\to$ IP Address |
| **Layer** | Layer 2.5 (Encapsulated in L2 Frame) | Layer 7 (Application over UDP/TCP) | Layer 2 (Obsolete; replaced by DHCP)|
| **Scope** | **Local Subnet / Broadcast Domain only** | Global Internet Hierarchy | Local Subnet |
| **Broadcast?** | Request is Broadcast; Reply is Unicast | Queries are Unicast | Request was Broadcast |

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"ARP resolves Layer 3 IPv4 addresses to Layer 2 MAC addresses within a local broadcast domain using a broadcast request and a unicast reply. Because ARP is stateless and completely lacks authentication, operating systems accept unsolicited replies. Attackers exploit this in ARP Cache Poisoning, sending forged Gratuitous ARP packets to the victim and default gateway to associate their own MAC address with both IP endpoints, establishing a Man-in-the-Middle position to sniff or tamper with traffic. The industry-standard mitigation is Dynamic ARP Inspection (DAI) deployed on access switches, which verifies every ARP packet against the DHCP Snooping binding table and drops unauthorized mappings."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Believing an attacker outside the local network can launch an ARP spoofing attack.  
  *Correction:* ARP packets are strictly Layer 2 Ethernet frames that **cannot traverse routers**. An attacker must have direct Layer 2 access on the same physical or VLAN broadcast domain to execute ARP poisoning.
- **Trap:** Forgetting that DAI requires DHCP Snooping.  
  *Correction:* You cannot run Dynamic ARP Inspection in isolation without first enabling DHCP Snooping, because DAI depends on the DHCP Snooping binding database to validate legitimate IP-to-MAC pairings.
