# CIDR Subnetting Calculations & Step-by-Step Worked Examples

> **Domain:** Networking Fundamentals  
> **Sub-Domain:** IP Addressing & Subnetting  
> **Interview Importance:** Very High / Common Live Technical Round Calculation  

---

## 1. Topic & Definitions

- **Subnetting:** The practice of dividing a large physical IP network into multiple smaller, logically isolated subnetworks (**Subnets**).
- **Why Subnet?**
  1. **Security Isolation:** Restricts network traffic between departments (e.g. Finance cannot talk to Guest Wi-Fi).
  2. **Broadcast Containment:** Shrinks the size of Layer 2 broadcast domains, reducing network congestion.
  3. **Address Conservation:** Allocates address blocks according to actual host needs instead of wasting entire Class A/B/C blocks.
- **CIDR (Classless Inter-Domain Routing):** The modern notation replacing legacy classful networks (Class A, B, C) by appending a slash (`/`) followed by the number of network bits (e.g. `/24`).
- **Subnet Mask:** A 32-bit number where the network portion consists of contiguous binary `1`s and the host portion consists of contiguous binary `0`s.

---

## 2. The Core Mathematical Formulas

For any given IPv4 address with CIDR prefix $/n$:
1. **Network Bits ($n$):** The CIDR prefix length (e.g., in `/26`, $n = 26$).
2. **Host Bits ($h$):** The remaining bits in the 32-bit address:
   $$h = 32 - n$$
3. **Total IP Addresses:**
   $$\text{Total IPs} = 2^h$$
4. **Usable Host Addresses:**
   $$\text{Usable Hosts} = 2^h - 2$$
   *(Subtracting 2 because the very first address is reserved as the **Network ID** and the very last address is reserved as the **Broadcast Address**).*
5. **Magic Number (Block Size):**
   $$\text{Block Size} = 256 - \text{Interesting Octet of Subnet Mask}$$

---

## 3. Subnetting Reference Cheat Table (/24 to /32)

| CIDR Prefix | Subnet Mask | Block Size | Total IPs | Usable Hosts | Typical Security Use Case |
| :---: | :--- | :---: | :---: | :---: | :--- |
| **/24** | `255.255.255.0` | 256 | 256 | **254** | Standard corporate departmental LAN |
| **/25** | `255.255.255.128`| 128 | 128 | **126** | Medium subnet |
| **/26** | `255.255.255.192`| 64 | 64 | **62** | Server farm / DMZ subnet |
| **/27** | `255.255.255.224`| 32 | 32 | **30** | Branch office / Database tier |
| **/28** | `255.255.255.240`| 16 | 16 | **14** | Management network / Admin bastion tier |
| **/29** | `255.255.255.248`| 8 | 8 | **6** | Small public IP allocation / HA pair |
| **/30** | `255.255.255.252`| 4 | 4 | **2** | Classic point-to-point router link |
| **/31** | `255.255.255.254`| 2 | 2 | **2** | RFC 3021 Point-to-Point links (No Net/Bcast loss) |
| **/32** | `255.255.255.255`| 1 | 1 | **1** | Single host route / Loopback address / Firewall rule |

---

## 4. Worked Calculation Walkthroughs

### Example 1: Finding Network, Broadcast & Usable Range for `192.168.10.145/26`

**Step 1: Determine Network and Host bits**
- Prefix: `/26` $\implies$ 26 network bits, $32 - 26 = 6$ host bits.

**Step 2: Calculate the Subnet Mask**
- Octet 1-3: $8 + 8 + 8 = 24$ bits $\implies 255.255.255.$
- Octet 4 (Interesting Octet): 2 bits $\implies 11000000_2 = 128 + 64 = 192$.
- Mask: `255.255.255.192`.

**Step 3: Calculate the Magic Number (Block Size)**
- $\text{Block Size} = 256 - 192 = 64$.

**Step 4: Determine the Subnet Blocks in the 4th Octet**
- $0, 64, 128, 192, 256\dots$
- Our IP's 4th octet is **145**.
- 145 falls cleanly between **128** and **192**.

**Step 5: Write the Results**
- **Network ID:** `192.168.10.128` (Start of block)
- **First Usable Host:** `192.168.10.129` (Network ID + 1)
- **Last Usable Host:** `192.168.10.190` (Broadcast - 1)
- **Broadcast Address:** `192.168.10.191` (Next block - 1)
- **Total Usable Hosts:** $2^6 - 2 = 64 - 2 = 62$.

---

### Example 2: Designing Subnets for a Requirement
> *"You are given the network `10.0.0.0/24`. You must split this network to create subnets that can each support at least 25 usable hosts. What CIDR prefix should you use, and how many such subnets can you create?"*

**Step 1: Calculate Host Bits Needed**
- Formula: $2^h - 2 \ge 25$.
- If $h = 4$: $2^4 - 2 = 14$ (Too small).
- If $h = 5$: $2^5 - 2 = 30$ (Valid! $30 \ge 25$).
- Therefore, we need **$h = 5$ host bits**.

**Step 2: Calculate New CIDR Prefix**
- $\text{Prefix} = 32 - h = 32 - 5 = \mathbf{/27}$.

**Step 3: Calculate Number of Subnets Created**
- Original network was `/24`. New subnets are `/27`.
- Borrowed network bits: $27 - 24 = 3$ bits.
- Number of Subnets $= 2^3 = \mathbf{8 \text{ subnets}}$.
- Each `/27` subnet provides **30 usable host IPs**.

---

## 5. Wildcard Masks in Firewalls & ACLs

- **Wildcard Mask:** An inverted subnet mask used in Cisco Access Control Lists (ACLs) and OSPF to tell the router which bits of an IP address to inspect and which to ignore.
  - Formula: $\text{Wildcard Mask} = 255.255.255.255 - \text{Subnet Mask}$.
- **Examples:**
  - Mask `255.255.255.0` (/24) $\implies$ Wildcard `0.0.0.255`.
  - Mask `255.255.255.192` (/26) $\implies$ Wildcard `0.0.0.63`.
  - Mask `255.255.255.255` (/32 - Host) $\implies$ Wildcard `0.0.0.0`.

---

## 6. Cybersecurity Relevance & Threat Vectors

### 1. Lateral Movement Containment via Microsegmentation
- In a flat network (`10.0.0.0/16` single subnet), if an attacker compromises a single workstation, they can scan and exploit every database, domain controller, and IoT device using Layer 2 broadcast discovery without crossing a firewall.
- **Defensive Subnetting:** Isolating databases in `/28` subnets behind stateful firewalls forces all inter-subnet traffic through security inspection rules.

### 2. Misconfigured Subnet Masks & VLAN Leaks
- If an admin configures a server with `192.168.1.50/16` instead of `192.168.1.50/24`, the server believes the entire `192.168.0.0/16` space is local. It will send ARP requests directly on the local link instead of forwarding traffic to the default gateway firewall, bypassing perimeter monitoring.

---

## 7. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Subnetting divides a larger network into smaller broadcast domains to improve network performance and enforce security microsegmentation. In CIDR notation, a prefix like `/26` defines 26 network bits and 6 host bits, yielding $2^6 = 64$ total IPs and $2^6 - 2 = 62$ usable host addresses. We find subnet boundaries using the Magic Number method: subtracting the interesting octet of the mask (`192`) from `256` gives a block size of `64`. The first IP is the non-routable Network ID and the last is the Broadcast address. In security architecture, rigorous subnetting restricts lateral movement and ensures traffic between tiers crosses firewall inspection boundaries."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Forgetting to subtract 2 for usable hosts.  
  *Correction:* Total IPs is $2^h$, but usable hosts is ALWAYS $2^h - 2$ (except on modern `/31` point-to-point links).
- **Trap:** Stumbling on the block size calculation.  
  *Correction:* Always memorize the powers of 2 for the last octet: `128` (block 128), `192` (block 64), `224` (block 32), `240` (block 16), `248` (block 8), `252` (block 4).
