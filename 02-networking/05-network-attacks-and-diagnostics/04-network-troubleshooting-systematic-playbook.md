# Network Troubleshooting: The Systematic 5-Step Interview Playbook

> **Domain:** Networking Fundamentals & Incident Response  
> **Sub-Domain:** Network Diagnostics & Systems Troubleshooting  
> **Interview Importance:** Critical / Classic Technical Scenario Interview Question  

---

## 1. Topic & Definitions

- **Systematic Network Troubleshooting:** The structured, bottom-up or divide-and-conquer methodology used by systems engineers, network administrators, and security incident responders to isolate, identify, and resolve network connectivity failures.
- **The Core Rule in Interviews:** **Never guess randomly.** Always follow a methodical, layer-by-layer diagnostic sequence, demonstrating to the interviewer that you understand the underlying protocol dependencies.

---

## 2. The 5-Step Bottom-Up Diagnostic Flowchart

```text
[ START: "I cannot connect to https://portal.internal.company.com" ]
                             │
                             ▼
  STEP 1: LOCAL INTERFACE & IP CONFIGURATION
  • Run: ip a / ipconfig
  • Check: Link UP? Valid IP assigned? (Or APIPA 169.254.x.x?) Default Gateway configured?
                             │
                  [OK: Valid IP & Gateway]
                             ▼
  STEP 2: DEFAULT GATEWAY & LAYER 3 REACHABILITY
  • Run: ping 127.0.0.1 (Tests local TCP/IP stack)
  • Run: ping <Default_Gateway_IP> (Tests local Layer 2/3 link to switch/router)
  • Run: ping 8.8.8.8 (Tests WAN routing past perimeter firewall)
                             │
                  [OK: Gateway & WAN reachable]
                             ▼
  STEP 3: DNS NAME RESOLUTION
  • Run: nslookup portal.internal.company.com / dig +trace portal.internal.company.com
  • Check: Does hostname resolve to valid expected IP? (Or NXDOMAIN / SERVFAIL?)
                             │
                  [OK: IP successfully resolved]
                             ▼
  STEP 4: LAYER 4 TRANSPORT & PORT CONNECTIVITY
  • Run: nc -zv <Target_IP> 443 / Test-NetConnection -ComputerName <Target_IP> -Port 443
  • Check: Does the TCP 3-way handshake succeed? (Or Connection Refused / Timed Out?)
                             │
                  [OK: TCP Port 443 Open]
                             ▼
  STEP 5: LAYER 7 APPLICATION & TLS VERIFICATION
  • Run: curl -Iv https://portal.internal.company.com
  • Run: openssl s_client -connect <Target_IP>:443 -servername portal.internal.company.com
  • Check: Is the TLS certificate expired or untrusted? Is web server returning 502/503 error?
                             │
                  [ ISSUE ISOLATED & RESOLVED! ✅ ]
```

---

## 3. Step-by-Step Triage Breakdown with Commands

### Step 1: Verify Local Interface & Addressing
```bash
# Linux
ip a                           # Check IP, CIDR mask, and state (UP/DOWN)
ip route show                  # Verify Default Gateway exists

# Windows PowerShell
Get-NetIPAddress               # Inspect active interface IPs
Get-NetRoute -DestinationPrefix "0.0.0.0/0" # Verify Default Gateway
```
- **Triage Decision:**
  - If IP is `169.254.x.x` $\implies$ **DHCP failure**. Check physical cable, VLAN configuration, or DHCP server scope.
  - If default gateway is missing $\implies$ Host cannot route traffic outside local subnet.

---

### Step 2: Layer 3 Reachability (ICMP Ping)
```bash
ping 127.0.0.1                 # Step 2a: Test internal kernel TCP/IP stack
ping 192.168.1.1               # Step 2b: Ping local Default Gateway
ping 8.8.8.8                   # Step 2c: Ping external WAN IP
```
- **Triage Decision:**
  - If ping to Gateway fails $\implies$ Layer 2 issue (bad cable, wrong VLAN, switch port disabled, ARP issue).
  - If Gateway responds but `8.8.8.8` fails $\implies$ Gateway cannot reach Internet (ISP outage, WAN firewall blocking outbound ICMP, upstream router down).

---

### Step 3: Layer 7 DNS Resolution
```bash
# Query specific DNS server directly
nslookup portal.company.com 192.168.1.1
dig @8.8.8.8 portal.company.com +trace
```
- **Triage Decision:**
  - If `nslookup` fails with `SERVFAIL` or `Connection timed out` $\implies$ Local DNS server is down or unconfigured in `/etc/resolv.conf`.
  - If `nslookup` returns `NXDOMAIN` $\implies$ Domain name does not exist or DNS zone file lacks an `A` record.
  - If `nslookup` returns an incorrect IP $\implies$ DNS poisoning, rogue hosts file entry, or stale DNS cache.

---

### Step 4: Layer 4 Transport Handshake & Path Tracing
```bash
# Path Tracing (Isolating intermediate hop failure)
traceroute -n portal.company.com    # Linux (uses UDP/ICMP by default)
traceroute -T -p 443 target.com     # TCP SYN traceroute (slips past firewalls!)
tracert -d portal.company.com       # Windows (uses ICMP)

# Direct Port 443 Handshake Test
nc -zv 10.0.0.50 443                # Netcat zero-I/O port probe
Test-NetConnection -ComputerName 10.0.0.50 -Port 443 # PowerShell port test
```
- **Triage Decision:**
  - If port test returns **`Connection Refused`** $\implies$ The target server is active and the network path is open, but **no web service daemon is listening on port 443** (Nginx/Apache is crashed or stopped).
  - If port test returns **`Connection Timed Out`** $\implies$ A **firewall (host-based iptables/Windows Firewall or network firewall) is silently dropping packets**.

---

### Step 5: Layer 7 Application & TLS Handshake
```bash
# Test HTTP Status & Response Headers
curl -Iv https://portal.company.com

# Inspect TLS Handshake & Certificate Chain Directly
openssl s_client -connect 10.0.0.50:443 -servername portal.company.com
```
- **Triage Decision:**
  - If OpenSSL returns `certificate has expired` or `self-signed certificate in certificate chain` $\implies$ Browser drops connection due to untrusted PKI certificate.
  - If curl returns `HTTP 502 Bad Gateway` $\implies$ The reverse proxy (Nginx) is alive, but the upstream application service (Node.js/Python) is crashed.

---

## 4. Master Diagnostic Command Cross-Reference Matrix

| Diagnostic Stage | Linux Command | Windows Command / PowerShell |
| :--- | :--- | :--- |
| **Check IP & Interface** | `ip a` / `ifconfig` | `Get-NetIPAddress` / `ipconfig /all` |
| **Check Routing Table** | `ip route show` | `Get-NetRoute` / `route print` |
| **Test Sockets & Ports** | `ss -tulnp` / `nc -zv <IP> <Port>` | `Get-NetTCPConnection` / `Test-NetConnection -Port` |
| **DNS Resolution** | `dig <domain>` / `nslookup` | `Resolve-DnsName <domain>` / `nslookup` |
| **Path Tracing** | `traceroute -n <IP>` / `mtr <IP>` | `tracert -d <IP>` |
| **Packet Sniffing** | `tcpdump -nn -i any port 443` | `netsh trace start` / Wireshark GUI |
| **Inspect TLS Handshake** | `openssl s_client -connect <IP>:443` | `openssl s_client` / Edge DevTools |

---

## 5. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"When troubleshooting a failed network connection, I follow a systematic bottom-up diagnostic workflow. First, I verify local IP configuration and the default gateway using `ip a` to rule out APIPA (169.254.x.x) or DHCP issues. Second, I test Layer 3 connectivity by pinging the default gateway, followed by an external IP like `8.8.8.8` to confirm routing. Third, I verify DNS name resolution using `nslookup` or `dig` to confirm the hostname maps to the expected IP. Fourth, I test Layer 4 port accessibility using Netcat (`nc -zv`) or `Test-NetConnection` on port 443—noting that a 'Connection Refused' signifies the web daemon is down while a 'Timed Out' signifies a firewall drop. Finally, I validate Layer 7 and TLS certificate chains using `openssl s_client` and `curl -Iv` to isolate certificate expiration or reverse proxy 502 gateway errors."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Immediately blaming DNS or the web server when a user says "the site is down".  
  *Correction:* Always verify local Layer 1 to 3 connectivity first. If the client machine unplugged their Ethernet cable or has an APIPA address, running DNS lookups is completely irrelevant.
- **Trap:** Believing `ping` failure proves the server is down.  
  *Correction:* Many enterprise firewalls and cloud servers (AWS EC2 security groups) explicitly block ICMP Echo Requests by default. Always test the specific TCP application port (using Netcat or curl) before concluding a host is unreachable.
