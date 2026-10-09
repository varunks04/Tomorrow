# DHCP Protocol, DORA Handshake, Rogue Servers & DHCP Snooping

> **Domain:** Networking Fundamentals  
> **Sub-Domain:** Network Addressing & Infrastructure Protocols  
> **Interview Importance:** High / Core Local Network Security  

---

## 1. Topic & Definitions

- **Dynamic Host Configuration Protocol (DHCP - RFC 2131):** A client-server network management protocol used to dynamically assign IP addresses, subnet masks, default gateways, and DNS server addresses to devices on an IP network.
- **Port Numbers:** Operates over UDP.
  - **Port 67:** Listened to by the **DHCP Server**.
  - **Port 68:** Listened to by the **DHCP Client**.
- **DHCP Lease:** The temporary duration of time for which a client is authorized to use an assigned IP address before it must renew or release it.
- **DHCP Snooping:** A Layer 2 switch security feature that acts as a firewall between untrusted hosts and DHCP servers, inspecting DHCP messages and dropping unauthorized DHCP offers.

---

## 2. The 4-Step DORA Handshake Walkthrough

```text
DHCP Client (Port 68)                                   DHCP Server (Port 67)
      │                                                           │
      ├─────── Step 1: DHCPDISCOVER ─────────────────────────────►│
      │        (Layer 2 Broadcast: FF:FF:FF:FF:FF:FF)             │
      │        (Layer 3 Broadcast: 255.255.255.255)               │
      │        "Is there a DHCP server? I need an IP address!"    │
      │                                                           │
      │◄────── Step 2: DHCPOFFER ─────────────────────────────────┤
      │        (Unicast or Broadcast depending on OS flag)        │
      │        "Here is IP 192.168.1.50, Mask, Gateway, DNS, 24h" │
      │                                                           │
      ├─────── Step 3: DHCPREQUEST ──────────────────────────────►│
      │        (Layer 2/3 Broadcast: Informs ALL servers)         │
      │        "I accept Server A's offer of 192.168.1.50!"       │
      │                                                           │
      │◄────── Step 4: DHCPACK ───────────────────────────────────┤
      │        "Confirmed! IP 192.168.1.50 is leased to your MAC" │
      ▼                                                           ▼
Client configures NIC with IP, Subnet Mask, Gateway & DNS!
```

### Why is `DHCPREQUEST` Broadcasted?
If multiple DHCP servers reside on the same broadcast domain, all of them may send a `DHCPOFFER`. The client chooses one and broadcasts the `DHCPREQUEST`. The broadcast serves two purposes:
1. It requests the selected IP from the chosen server.
2. It implicitly notifies all **other** DHCP servers that their offers were rejected, allowing them to release their reserved IPs back into their available pools.

### Lease Timers: T1 and T2
- **T1 Timer (Renewal - 50% of Lease Time):** Client attempts to renew the lease by sending a unicast `DHCPREQUEST` directly to the original DHCP server.
- **T2 Timer (Rebinding - 87.5% of Lease Time):** If the original server does not respond by 87.5% of the lease, the client broadcasts a `DHCPREQUEST` to *any* available DHCP server on the network.
- **Lease Expiration (100%):** If no renewal occurs, the client immediately drops the IP and restarts the full `DHCPDISCOVER` process (often falling back to APIPA `169.254.x.x`).

---

## 3. DHCP Relay Agent (Option 82)

- Routers do **not forward Layer 2 broadcasts** (`255.255.255.255`).
- If the corporate DHCP server resides in a different subnet or centralized data center, the local default gateway router acts as a **DHCP Relay Agent** (Cisco `ip helper-address`).
- The relay agent converts the client's local broadcast `DHCPDISCOVER` into a unicast packet forwarded directly to the centralized DHCP server, appending **DHCP Option 82** (Circuit ID / Remote ID) to inform the server which subnet pool to draw from.

---

## 4. Cybersecurity Relevance & Threat Vectors

### 1. DHCP Starvation Attack (Denial of Service)
- **Mechanism:** An attacker runs an automated script (e.g. `yersinia` or `dhcpstarv`) that rapidly floods the network with thousands of `DHCPDISCOVER` packets, each using a randomly generated spoofed MAC address.
- **Impact:** The DHCP server allocates all available IP addresses in its scope to fake MAC addresses. When legitimate employees connect to the network, no IPs are available, preventing network access.

### 2. Rogue DHCP Server & Man-in-the-Middle (MitM)
- Once an attacker exhausts legitimate DHCP addresses (via starvation), or by simply responding faster than the legitimate server:
  - The attacker spawns a **Rogue DHCP Server** (e.g. using `dnsmasq`).
  - The rogue server replies to new client discoveries, offering valid IPs but assigning the **Attacker's IP as the Default Gateway and Primary DNS Server**!
  - All outgoing and incoming internet traffic from the victim is now routed directly through the attacker's machine, enabling full credential sniffing and TLS stripping.

---

## 5. The Definitive Defense: DHCP Snooping

```text
[ Rogue Attacker (Port 4) ] ── (Sends fake DHCPOFFER) ──► [ UNTRUSTED PORT: DROPPED! ❌ ]
                                                                   │
                                                      [ Layer 2 Switch with DHCP Snooping ]
                                                                   │
[ Genuine DHCP Server (Port 24) ] ── (Sends DHCPOFFER) ─► [ TRUSTED PORT: PERMITTED! ✅ ]
```

### How DHCP Snooping Protects the Network:
1. **Port Classification:**
   - **Trusted Ports:** Only switch ports connected to verified, legitimate DHCP servers or upstream switches are marked as `trusted`. DHCP server messages (`DHCPOFFER`, `DHCPACK`) are allowed.
   - **Untrusted Ports:** All standard edge access ports connected to end-user workstations. If any device on an untrusted port attempts to send a `DHCPOFFER` or `DHCPACK`, the switch **instantly drops the packet and err-disables the port**.
2. **Rate Limiting:** Drops anomalous floods of `DHCPDISCOVER` packets on untrusted ports to defeat Starvation attacks.
3. **Builds the DHCP Snooping Binding Database:**
   The switch intercepts valid `DHCPACK` packets and builds a secure database:
   `{ MAC Address, Leased IP, Lease Time, VLAN ID, Switch Port }`
   This database is the indispensable prerequisite for **Dynamic ARP Inspection (DAI)** and **IP Source Guard (IPSG)**.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"DHCP automates network configuration via the 4-step DORA process: Discover (broadcast), Offer (server provides IP and gateway), Request (client accepts chosen IP broadcast), and Acknowledgment (server commits lease). It operates over UDP ports 67 for servers and 68 for clients. Because DORA uses unauthenticated broadcasts, attackers can launch DHCP Starvation attacks to exhaust address pools or deploy Rogue DHCP servers to hand out malicious gateways for Man-in-the-Middle attacks. The standard enterprise mitigation is Layer 2 DHCP Snooping, which designates switch ports as trusted or untrusted, dropping rogue offers and constructing a binding database used to secure ARP and IP communications."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing DHCP ports.  
  *Correction:* Client uses **UDP 68**, Server uses **UDP 67**. (Mnemonic: *C*lient comes first alphabetically, but in ports Server = 67, Client = 68).
- **Trap:** Believing a router forwards DHCP Discover packets by default.  
  *Correction:* Routers drop all broadcast traffic. To support centralized DHCP servers across subnets, you must configure a **DHCP Relay Agent** (`ip helper-address`) on the router interface.
