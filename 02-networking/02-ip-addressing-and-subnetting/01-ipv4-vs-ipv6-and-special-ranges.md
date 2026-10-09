# IPv4 vs. IPv6 Addressing, Address Classes & Special Purpose Ranges

> **Domain:** Networking Fundamentals  
> **Sub-Domain:** IP Addressing & Routing  
> **Interview Importance:** Very High / Foundational Networking  

---

## 1. Topic & Definitions

- **Internet Protocol (IP) Address:** A unique numerical identifier assigned to every device connected to a computer network that uses the Internet Protocol for communication. Operates at Layer 3 (Network Layer).
- **IPv4:** A 32-bit numeric address expressed in dotted-decimal format (`192.168.1.1`), providing a theoretical total of $2^{32} \approx 4.29 \text{ billion}$ unique addresses.
- **IPv6:** A 128-bit hexadecimal address expressed in 8 colon-separated groups (`2001:0db8:85a3:0000:0000:8a2e:0370:7334`), providing $2^{128} \approx 3.4 \times 10^{38}$ unique addresses, created to permanently solve IPv4 address exhaustion.
- **RFC 1918 Private IP Ranges:** Reserved non-routable IP blocks designated for internal private enterprise networks that cannot be routed across the public Internet.

---

## 2. Comparison: IPv4 vs. IPv6 Architecture

| Feature / Dimension | IPv4 | IPv6 |
| :--- | :--- | :--- |
| **Address Length** | 32 bits (4 bytes). | 128 bits (16 bytes). |
| **Address Notation** | Dotted Decimal (e.g. `192.168.1.1`). | Hexadecimal with Colons (e.g. `fe80::1`). |
| **Total Address Space** | $2^{32} \approx 4.29 \times 10^9$ addresses. | $2^{128} \approx 3.4 \times 10^{38}$ addresses. |
| **Header Size** | Variable: 20 to 60 bytes (Options field). | Fixed: strictly **40 bytes** (Simplifies router processing). |
| **Checksum in Header** | Yes (Calculated and verified at every hop). | **None** (Removed to accelerate routing; handled at L2/L4). |
| **Address Configuration**| Manual or DHCP. | SLAAC (Stateless Address Autoconfiguration) or DHCPv6. |
| **Security (IPsec)** | Optional add-on. | Built natively into the protocol specification. |
| **Broadcasting** | Supports Broadcast (`255.255.255.255`). | **Broadcast eliminated!** Replaced by efficient Multicast. |

---

## 3. Special Purpose IPv4 Address Ranges You Must Know

```text
+─────────────────────────────────────────────────────────────────────────────+
|                         SPECIAL PURPOSE IPV4 RANGES                         |
+─────────────────────────────────────────────────────────────────────────────+
| RFC 1918 PRIVATE RANGES (Non-routable on public internet):                  |
|   • Class A: 10.0.0.0/8         (10.0.0.0 to 10.255.255.255)                |
|   • Class B: 172.16.0.0/12      (172.16.0.0 to 172.31.255.255)              |
|   • Class C: 192.168.0.0/16     (192.168.0.0 to 192.168.255.255)            |
|─────────────────────────────────────────────────────────────────────────────|
| SPECIAL ADDRESSES:                                                          |
|   • Loopback:     127.0.0.0/8   (127.0.0.1 / localhost - internal to OS)    |
|   • APIPA:        169.254.0.0/16 (Link-Local auto-assigned if DHCP fails)   |
|   • Multicast:    224.0.0.0/4   (Class D: 224.0.0.0 to 239.255.255.255)     |
|   • Experimental: 240.0.0.0/4   (Class E: Reserved)                         |
|   • Broadcast:    255.255.255.255 (Limited local broadcast)                 |
|   • Default Route:0.0.0.0/0     (Represents "any network" / default gateway)|
+─────────────────────────────────────────────────────────────────────────────+
```

---

## 4. Troubleshooting Clues: APIPA & Loopback

### What Does `169.254.X.X` Mean in an Interview Scenario?
- **APIPA (Automatic Private IP Addressing):** If a client operating system (Windows/macOS) is configured for DHCP but fails to receive a response from a DHCP server during the DORA handshake, it automatically generates a random IP in the `169.254.0.0/16` range.
- **Diagnostic Diagnosis:** An APIPA address is an instant indicator of **DHCP server failure, unplugged physical cable, wrong VLAN assignment, or network connectivity loss**. The device can only talk to other devices on the same physical link with `169.254.x.x` addresses; it **cannot access the Internet or default gateway**.

### Loopback Address (`127.0.0.1` vs `::1`)
- Packets sent to `127.0.0.1` never touch the physical Network Interface Card (NIC) or network wire; the operating system kernel immediately routes them back to the local software stack.
- Used to test local services (web servers, databases) and verify that the local TCP/IP stack is functioning correctly.

---

## 5. Cybersecurity Relevance & Threat Vectors

### 1. Bogon and Martian Packet Filtering
- **Bogon Filtering:** Packets arriving on an external Internet-facing interface that claim to originate from private RFC 1918 ranges, APIPA (`169.254.0.0/16`), or loopback (`127.0.0.0/8`) are mathematically impossible and indicate **IP Spoofing**.
- **Defense:** Edge routers and firewalls must enforce **uRPF (Unicast Reverse Path Forwarding)** and drop all incoming bogon traffic.

### 2. Cloud Metadata SSRF via Link-Local IP (`169.254.169.254`)
- In major cloud providers (AWS, Azure, GCP), the Instance Metadata Service (IMDS) runs on the link-local IP `http://169.254.169.254`.
- If an application suffers from a **Server-Side Request Forgery (SSRF)** vulnerability, an attacker instructs the server to fetch `http://169.254.169.254/latest/meta-data/iam/security-credentials/`, extracting cloud IAM administrative access tokens directly.
- **Mitigation:** Enforce AWS **IMDSv2**, which requires session token headers (`X-aws-ec2-metadata-token`) that simple SSRF payloads cannot forge.

### 3. IPv6 Dual-Stack Security Blind Spots
- Many enterprise networks deploy IPv6 without realizing it. If firewalls and SIEM logging are configured only to monitor IPv4, attackers can establish command-and-control (C2) or exfiltrate data over unmonitored IPv6 connections.
- **Hardening:** Either fully configure security controls on IPv6 or explicitly disable IPv6 on sensitive hosts.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"IPv4 utilizes 32-bit addresses providing 4.3 billion combinations, split into public addresses and RFC 1918 private ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) which require NAT to access the Internet. IPv6 provides a virtually infinite 128-bit address space, eliminating NAT, replacing broadcast with multicast, and fixing the header size at 40 bytes. Special ranges include 127.0.0.1 for loopback and 169.254.0.0/16 for APIPA, which signals DHCP failure in troubleshooting. In security, private IPs arriving on WAN interfaces signify IP spoofing (bogon traffic), and the link-local address 169.254.169.254 is the primary target for cloud SSRF metadata credential theft."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Forgetting the Class B private range boundaries.  
  *Correction:* It is NOT `172.0.0.0/8`. The RFC 1918 Class B range is strictly **`172.16.0.0` to `172.31.255.255`** (`/12` prefix).
- **Trap:** Stating IPv6 uses broadcasts.  
  *Correction:* IPv6 has completely deprecated broadcasts to reduce network noise. It uses **Anycast, Multicast, and Unicast**.
