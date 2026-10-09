# The Systematic 5-Step Network Connection Troubleshooting Playbook

## 1. Scenario & Methodology
Network outages and failed client-server connections represent the most frequent operational incidents in engineering and infrastructure security.

Rather than guessing randomly, senior engineers execute a **Deterministic Bottom-Up OSI Troubleshooting Methodology**, systematically verifying connectivity from the physical network interface up to application-layer TLS handshakes.

---

## 2. The 5-Step Diagnostic Decision Flowchart

```mermaid
flowchart TD
    Start([Issue: "Cannot Connect to https://api.corp.internal"]) --> Step1[Step 1: Interface & Local IP Check]
    Step1 --> CheckIP{Interface UP and valid IP assigned?}
    CheckIP -- No --> FixDHCP[Check Cable / Re-request DHCP: sudo dhclient]
    CheckIP -- Yes --> Step2[Step 2: Gateway & ARP Resolution]
    
    Step2 --> PingGW{Can ping Default Gateway?}
    PingGW -- No --> FixL2[Check ARP Table: arp -a / VLAN misconfig]
    PingGW -- Yes --> Step3[Step 3: Remote IP Path Routing]
    
    Step3 --> PingPublic{Can ping public IP 8.8.8.8?}
    PingPublic -- No --> FixRoute[Check Routing Table: ip route / ISP Outage]
    PingPublic -- Yes --> Step4[Step 4: DNS Name Resolution]
    
    Step4 --> DigHost{Does 'dig api.corp.internal' resolve?}
    DigHost -- No --> FixDNS[DNS Failure: Check /etc/resolv.conf & DNS Server]
    DigHost -- Yes --> Step5[Step 5: Transport Port & TLS Handshake]
    
    Step5 --> TestPort{Does 'nc -zv <IP> 443' succeed?}
    TestPort -- No (Timeout) --> FWDrop[Firewall / Security Group dropping packets!]
    TestPort -- No (RST/Refused) --> AppDown[Port closed: Target service is NOT running!]
    TestPort -- Yes --> TestTLS{Does 'curl -vvv' complete TLS handshake?}
    TestTLS -- Cert Error --> CertIssue[Expired Cert / Untrusted CA / SNI mismatch]
    TestTLS -- 200 OK --> AppSuccess([SUCCESS: Connection Fully Functional!])
```

---

## 3. Step-by-Step Diagnostic Execution & CLI Tooling

### Step 1: Interface & Local IP Verification (Layer 1 & 2)
Verify that the physical network adapter is connected and has received a valid IP address:
```bash
# Linux: Check interface link state and IP
ip -br addr show
# Windows:
ipconfig /all
```
- **Diagnostic Triage:**
  - Interface state is `DOWN`: Physical cable unplugged, virtual NIC disconnected, or switch port disabled.
  - IP is in `169.254.x.x` range (**APIPA**): **DHCP Exhaustion / Failure**. The client could not reach a DHCP server and self-assigned a non-routable link-local address.

---

### Step 2: Gateway Reachability & ARP Resolution (Layer 2 & 3)
Verify communication within the local subnet and check if the Default Gateway is alive:
```bash
# 1. Identify Default Gateway IP
ip route show | grep default
# Output: default via 192.168.1.1 dev eth0

# 2. Ping the Gateway
ping -c 3 192.168.1.1

# 3. Inspect the Layer-2 ARP Cache
ip neighbor show
# Or: arp -n
```
- **Diagnostic Triage:**
  - `Destination Host Unreachable` or ARP entry shows `INCOMPLETE`: Layer-2 failure. Mismatched subnet mask, VLAN tag misconfiguration, or default gateway is powered off.

---

### Step 3: Remote IP Routing & Path Tracing (Layer 3)
Determine whether packets can cross the local router to reach remote internet networks:
```bash
# 1. Test raw internet routing using an IP address (bypassing DNS entirely)
ping -c 3 8.8.8.8

# 2. Trace the hop-by-hop packet path
traceroute -n 8.8.8.8
# Modern Linux: mtr -n --report 8.8.8.8
```
- **Diagnostic Triage:**
  - Pinging `8.8.8.8` succeeds: External routing is healthy! The problem is likely DNS or upper layers.
  - `traceroute` stops at hop 1 (Gateway): The router's WAN link is down or NAT is misconfigured.
  - `traceroute` shows `* * *` intermediate hops: Routers dropping ICMP (normal), but if it stops completely mid-path, indicates an ISP routing loop or BGP outage.

---

### Step 4: DNS Name Resolution (Layer 7)
Verify that the hostname resolves to the correct IP address:
```bash
# 1. Query DNS using dig
dig +noall +answer api.corp.internal

# 2. Query authoritative servers directly (trace delegation)
dig +trace api.corp.internal @8.8.8.8

# 3. Check local resolver configuration
cat /etc/resolv.conf
```
- **Diagnostic Triage:**
  - `NXDOMAIN`: Domain does not exist or has a typo.
  - `SERVFAIL`: Authoritative DNS server unreachable or DNSSEC validation failed.
  - `Connection timed out; no servers could be reached`: Local DNS server (Port 53 UDP) is down or blocked by an internal firewall.

---

### Step 5: Transport Layer & Application Port Sockets (Layer 4 & 7)
Verify if the target application is actively listening and reachable on the requested TCP port:
```bash
# 1. Test Layer-4 TCP port reachability
nc -zv 192.0.2.50 443
# Or: telnet 192.0.2.50 443

# 2. Test full HTTP application & TLS handshake
curl -vvv https://api.corp.internal

# 3. (On Target Server): Verify if process is listening on the port
ss -tulpn | grep :443
# Or: netstat -tulpn | grep :443
```

---

## 4. Diagnostic Signature Matrix: Root Cause Fingerprinting

| Command Output Signature | Layer | Root Cause Diagnosis | Remediation Action |
| :--- | :--- | :--- | :--- |
| IP assigned: `169.254.42.88` | L3 | **APIPA (DHCP Failure)** | Check DHCP scope, server availability, VLAN |
| `arp -n` shows `INCOMPLETE` | L2 | **ARP resolution failure** | Check gateway IP, local switch VLAN tagging |
| `ping 8.8.8.8` OK, `curl host` fails | L7 | **DNS resolution failure** | Check `/etc/resolv.conf`, test `dig @8.8.8.8` |
| `Connection timed out` on port | L4 | **Packet DROP (Firewall)** | Security Group / iptables dropping SYN packets |
| `Connection refused` (TCP RST) | L4 | **Port CLOSED (Host alive)** | Target daemon (Nginx/App) is crashed or stopped |
| `SSL_ERROR_BAD_CERT_DOMAIN` | L7 | **TLS SNI / SAN Mismatch** | Certificate hostname does not match requested URL |

---

## 5. Packet-Level Diagnostics with `tcpdump`

When higher-level tools provide ambiguous results, capture the raw packet wire:

```bash
# Capture raw TCP handshakes and resets on port 443
sudo tcpdump -nn -i any port 443 -c 10
```

### Packet Analysis Patterns:
1. **Outgoing `[SYN]` followed by complete silence:** Packets are being silently dropped by an upstream firewall or security group.
2. **Outgoing `[SYN]` followed immediately by incoming `[RST, ACK]`:** Packets reached the server, but the operating system kernel actively rejected the connection because no process was bound to port 443.
3. **Outgoing `[SYN]` followed by `ICMP Destination Unreachable (Admin Prohibited)`:** Explicitly blocked by a router ACL or local iptables reject rule.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Systematic network troubleshooting follows a rigorous bottom-up OSI approach. We verify Layer 1 and 2 by checking network interface link status and ensuring an IP address was successfully assigned via DHCP rather than falling back to an APIPA link-local address. Next, we verify Layer 2/3 local gateway reachability via ARP and ping. To isolate DNS from general network failure, we ping a public IP like `8.8.8.8`; if the IP succeeds but hostnames fail, we troubleshoot DNS via `dig`. Finally, we evaluate Layer 4 transport reachability using `nc -zv` or `curl -vvv`. In transport analysis, the output provides an exact signature: a 'Connection Timeout' indicates an upstream firewall dropping packets, while a 'Connection Refused' TCP RST proves the network path is fully open but the backend service daemon is stopped."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Testing with `ping google.com` immediately. *Correction:* Pinging a hostname tests DNS, ICMP routing, and network connectivity all at once. If it fails, you don't know if the issue is DNS, firewall, or physical link. Ping by IP address first!
- **Trap 2:** Confusing "Connection Timed Out" with "Connection Refused". *Correction:* Timed Out = firewall dropped packet; Refused = server received packet and replied with TCP RST because the port is closed.
- **Trap 3:** Assuming ping works if web works. *Correction:* Many corporate firewalls and cloud security groups intentionally block ICMP echo (ping) while allowing TCP port 443; always test the specific application port via `nc` or `curl`.

### Expected Follow-Up Questions
1. *What is the difference between `traceroute` on Linux vs `tracert` on Windows?*
   - Linux `traceroute` defaults to sending high-port UDP packets (or TCP with `-T`), whereas Windows `tracert` defaults to sending ICMP Echo Request packets.
2. *Why does a curl command return `curl: (60) SSL certificate problem: self-signed certificate`?*
   - Because the client's local trust store does not contain the Root CA certificate that issued the server's certificate. In corporate networks, this frequently indicates SSL Inspection / Decryption proxies intercepting traffic.
