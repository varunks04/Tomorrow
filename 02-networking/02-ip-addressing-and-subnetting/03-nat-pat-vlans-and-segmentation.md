# NAT, PAT, VLANs & Network Microsegmentation

> **Domain:** Networking Fundamentals  
> **Sub-Domain:** Network Architecture & Perimeter Security  
> **Interview Importance:** Very High / Core Enterprise Network Design  

---

## 1. Topic & Definitions

- **NAT (Network Address Translation):** A Layer 3 routing process that modifies IP address information in packet headers while in transit across a traffic routing device, allowing an entire private network to share public IP addresses.
- **PAT (Port Address Translation / NAT Overload):** A dynamic form of NAT where multiple private internal hosts share a **single public IP address** by translating both the IP address and Layer 4 **Source Port numbers** simultaneously.
- **VLAN (Virtual Local Area Network):** A Layer 2 network segmentation technology that partitions a physical switch into multiple isolated logical broadcast domains. Hosts on different VLANs cannot communicate at Layer 2 even if connected to the same physical switch.
- **Trunk Port:** A switch port configured to carry traffic for multiple VLANs simultaneously across switches using **IEEE 802.1Q** tagging.
- **Access Port:** A switch port that belongs to a single VLAN and transmits standard untagged Ethernet frames to end-user devices.

---

## 2. NAT Types & The PAT Connection Table

```text
+─────────────────────────────────────────────────────────────────────────────+
|                                 NAT FLAVORS                                 |
+─────────────────────────────────────────────────────────────────────────────+
| 1. STATIC NAT (1-to-1):                                                     |
|    • One private IP maps permanently to one public IP.                      |
|    • Use case: Hosting public servers (DMZ web/mail servers).               |
|─────────────────────────────────────────────────────────────────────────────|
| 2. DYNAMIC NAT (Many-to-Many Pool):                                         |
|    • A private host temporarily claims an available public IP from a pool.  |
|    • Use case: Infrequently used legacy corporate outbound pools.           |
|─────────────────────────────────────────────────────────────────────────────|
| 3. PAT / NAT OVERLOAD (Many-to-1):                                          |
|    • Thousands of private IPs share a SINGLE public IP by assigning         |
|      unique Layer 4 source ports to each outbound session.                  |
|    • Use case: Home routers, corporate Wi-Fi, mobile cellular networks.     |
+─────────────────────────────────────────────────────────────────────────────+
```

### PAT State Table Step-by-Step Flow

```text
Private Host (192.168.1.50:52341) ────► [ Router / NAT Gateway ] ────► Public Server (93.184.216.34:443)
                                          (Public IP: 203.0.113.1)
```

**Inside the NAT Gateway State Translation Table:**
| Inside Local (Private IP:Port) | Inside Global (Public IP:Port) | Outside Local / Global (Destination) |
| :--- | :--- | :--- |
| `192.168.1.50:52341` | `203.0.113.1:61001` | `93.184.216.34:443` |
| `192.168.1.51:52341` | `203.0.113.1:61002` | `93.184.216.34:443` |

1. When `192.168.1.50` connects to the server, the router replaces the source IP with `203.0.113.1` and maps port `52341` to unique port `61001`.
2. When the server replies to `203.0.113.1:61001`, the router inspects the table, demultiplexes the port, replaces the destination with `192.168.1.50:52341`, and forwards the frame to the client.

---

## 3. VLANs & IEEE 802.1Q Tagging

```text
Standard Ethernet Frame (Untagged):
[ Dest MAC (6B) ] [ Src MAC (6B) ] [ Type (2B) ] [ Payload (46-1500B) ] [ FCS (4B) ]

802.1Q Tagged Frame (Inserted on Trunk Links):
[ Dest MAC ] [ Src MAC ] [ 802.1Q Tag (4 Bytes) ] [ Type ] [ Payload ] [ FCS ]
                               │
                               ├─ TPID (0x8100) -> 2 Bytes (Identifies 802.1Q frame)
                               ├─ Priority (PCP) -> 3 Bits (QoS priority)
                               ├─ DEI -> 1 Bit (Drop Eligible Indicator)
                               └─ VLAN ID (VID) -> 12 Bits (Supports VLANs 1 to 4094)
```

### Inter-VLAN Routing
Because VLANs are separate Layer 2 broadcast domains, traffic between VLANs **must be routed at Layer 3**:
1. **Router-on-a-Stick:** A single physical router interface connects via a trunk link to a switch, using sub-interfaces (e.g. `Gig0/0.10`, `Gig0/0.20`) with 802.1Q encapsulation to route traffic between VLANs.
2. **Layer 3 Switch (Multilayer Switch):** Routes traffic at hardware wire-speed using internal Switch Virtual Interfaces (SVIs), drastically reducing inter-VLAN latency.

---

## 4. Network Microsegmentation & Zero Trust

- **Traditional Perimeter Security ("Castle and Moat"):** Flat network behind a strong perimeter firewall. Once an attacker breaches the perimeter, they have unhindered access to the entire internal network.
- **Microsegmentation:** Dividing the network into granular, isolated zones down to individual workload tiers (Web $\to$ App $\to$ Database) or single workloads.
- **Zero Trust Principle:** Every boundary crossing requires strict Layer 4/7 firewall inspection and cryptographic identity verification, regardless of whether the traffic originates internally.

---

## 5. Cybersecurity Relevance & Threat Vectors

### 1. VLAN Hopping Attacks
- **Switch Spoofing:** An attacker connects a rogue laptop running Yersinia to a switch port and negotiates a dynamic trunk link using **DTP (Dynamic Trunking Protocol)**. Once trunked, the attacker can sniff and send frames to all VLANs.  
  *Mitigation:* Disable DTP on all access ports (`switchport mode access; switchport nonegotiate`).
- **Double Tagging:** Exploits the native VLAN on 802.1Q trunks. An attacker sends a frame with two VLAN tags (Outer tag = Native VLAN, Inner tag = Victim VLAN). The first switch strips the outer tag and forwards the inner-tagged frame, delivering it directly to the victim VLAN without traversing a router.  
  *Mitigation:* Change native VLAN to an unused, isolated VLAN ID across all trunk links.

### 2. Forensic Challenges of NAT
- Because hundreds of internal employees share a single public IP, when a threat intelligence report flags malicious outbound traffic from `203.0.113.1`, security analysts cannot identify the compromised internal machine without inspecting **detailed NAT session state logs** correlating ephemeral source ports and timestamps.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"NAT translates private RFC 1918 addresses into routable public IPs. Static NAT provides a 1-to-1 permanent mapping for public DMZ servers, while Port Address Translation (PAT) allows thousands of internal hosts to share a single public IP by multiplexing Layer 4 source ports in an internal translation table. At Layer 2, VLANs segment physical switches into distinct broadcast domains using IEEE 802.1Q 4-byte tags on trunk links, requiring Layer 3 routers or multilayer switches for inter-VLAN routing. In modern security architecture, VLANs and microsegmentation enforce Zero Trust by containing lateral movement, while hardening switches against VLAN hopping attacks requires disabling DTP and isolating the native VLAN."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Claiming NAT is a security firewall.  
  *Correction:* NAT provides address translation, NOT security. While PAT provides incidental ingress shielding because inbound connections cannot reach unmapped internal ports, it does not inspect packet payloads, prevent malicious outbound downloads, or filter application attacks.
- **Trap:** Forgetting the size of the 802.1Q VLAN ID field.  
  *Correction:* The VLAN ID is **12 bits**, which is why the maximum number of VLANs on a switch is $2^{12} = 4096$ (valid range: 1 to 4094).
