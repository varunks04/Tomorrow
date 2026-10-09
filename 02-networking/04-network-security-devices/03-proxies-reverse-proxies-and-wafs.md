# Forward Proxies, Reverse Proxies & Web Application Firewalls (WAF)

> **Domain:** Networking Fundamentals & Application Security  
> **Sub-Domain:** Proxy Architectures & Layer 7 Defense  
> **Interview Importance:** Very High / Common System Design & Security Question  

---

## 1. Topic & Definitions

- **Proxy Server:** An intermediary server that acts as a gateway between an endpoint and the broader network, terminating the client's connection and initiating a new request on the client's behalf.
- **Forward Proxy:** Sits in front of a group of **internal clients** and acts on their behalf to mediate outgoing connections to the external Internet.
- **Reverse Proxy:** Sits in front of one or more **backend origin web servers**, accepting incoming requests from the public Internet and routing them to the appropriate backend server.
- **Web Application Firewall (WAF):** A specialized **Layer 7** security firewall that monitors, filters, and blocks HTTP and HTTPS traffic to and from a web application, specifically inspecting application payloads for OWASP Top 10 vulnerabilities (SQLi, XSS, CSRF, SSRF).

---

## 2. Forward Proxy vs. Reverse Proxy Architecture

```text
FORWARD PROXY (Protects & Mediates CLIENTS):
[ Internal Client 1 ] ──┐
[ Internal Client 2 ] ──┼──► [ FORWARD PROXY ] ──── (Public Internet) ────► [ External Web Servers ]
[ Internal Client 3 ] ──┘   • Filters outgoing URLs
                            • Enforces corporate DLP
                            • Hides internal client IPs

REVERSE PROXY (Protects & Mediates SERVERS):
[ Public User 1 ] ─────┐
[ Public User 2 ] ─────┼──► [ REVERSE PROXY ] ──── (Private LAN) ────► [ Backend Web App 1 ]
[ Malicious Prober ] ──┘   • SSL/TLS Termination                      [ Backend Web App 2 ]
                           • Load Balancing                           [ Database Cluster ]
                           • Hides backend server IPs
```

### Detailed Functional Comparison

| Feature | Forward Proxy | Reverse Proxy |
| :--- | :--- | :--- |
| **Whom does it protect?** | Protects the **Client** (Internal employee). | Protects the **Server** (Origin web infrastructure). |
| **Visibility to Client** | Client explicitly configures proxy settings or PAC file. | Completely transparent; user believes they are talking to the origin. |
| **Primary Use Cases** | - Corporate URL categorization/filtering<br>- Data Loss Prevention (DLP)<br>- Caching web downloads<br>- Client IP anonymity | - SSL/TLS termination & certificate management<br>- Load balancing (Round Robin, Least Connections)<br>- Web caching & CDN integration<br>- DDoS buffering |
| **Representative Tech** | Squid, Blue Coat, Zscaler, Cisco WSA. | Nginx, HAProxy, Envoy, Cloudflare, Traefik. |

---

## 3. The Web Application Firewall (WAF) Deep Dive

While traditional network firewalls operate at Layer 3 and 4 (filtering IP addresses and TCP ports), a **WAF operates exclusively at Layer 7 (Application Layer)**.

```text
Incoming HTTP Request:
GET /products.php?id=1' UNION SELECT username, password FROM users-- HTTP/1.1
Host: bank.com
User-Agent: sqlmap/1.6
Cookie: session=abc

[ Layer 3/4 Network Firewall ]:
Checks: Dest IP = Bank IP? Yes. Dest Port = 443? Yes. ──► PERMITS PACKET! (Blind to attack!)

[ Layer 7 WAF (e.g. ModSecurity / AWS WAF) ]:
1. Decrypts TLS.
2. Inspects User-Agent: "sqlmap" detected ──► BLOCK!
3. Inspects query param 'id': Regex matches SQL injection syntax ──► DROPS REQUEST!
4. Returns: HTTP 403 Forbidden with security incident ID.
```

---

## 4. Architectural Comparison: Network Firewall vs. IPS vs. WAF

| Evaluation Dimension | Network Firewall (L3/L4) | Intrusion Prevention System (IPS) | Web Application Firewall (WAF) |
| :--- | :--- | :--- | :--- |
| **Primary Layer** | Layer 3 & Layer 4 (IP, Port). | Layer 3 & Layer 4 (with some L7). | **Layer 7 exclusively (HTTP/HTTPS)**. |
| **Protocol Scope** | All IP protocols (TCP, UDP, ICMP, GRE). | All IP protocols (DNS, SMB, TCP, ICMP).| Web protocols: **HTTP, HTTPS, WebSockets**. |
| **Inspection Depth** | IP headers, TCP flags, Port numbers. | Deep Packet Inspection of raw bytes and network protocols. | Decodes URI parameters, HTTP headers, JSON payloads, XML, Cookies, Multi-part files. |
| **Focus Attacks** | Unauthorized port access, SYN floods, IP spoofing. | Exploit delivery, buffer overflows, worm propagation, port scans. | **OWASP Top 10:** SQL Injection, XSS, CSRF, SSRF, Path Traversal, Bot attacks. |
| **SSL/TLS Handling** | Passes encrypted traffic blindly. | Requires dedicated SSL inspection engine. | Natively terminates and inspects decrypted HTTPS traffic. |

---

## 5. WAF Detection Engines & The OWASP Core Rule Set (CRS)

1. **Signature / Pattern Matching:** Uses regular expressions from curated rulesets (such as the **OWASP ModSecurity Core Rule Set - CRS**) to match known attack vectors in request parameters.
2. **Behavioral / Bot Analysis:** Evaluates request rates, CAPTCHA challenges, TLS client fingerprints (JA3/JA4), and headless browser automation signatures.
3. **Anomaly Scoring Mode:** Instead of blocking on the very first matched rule, each suspicious pattern adds points to an anomaly score. If the cumulative score exceeds a threshold (e.g. 5), the request is blocked, dramatically reducing false positives.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"A Forward Proxy mediates outbound traffic for clients to enforce URL filtering, data loss prevention, and anonymity, whereas a Reverse Proxy mediates inbound traffic for backend web servers to handle SSL termination, load balancing, and topology hiding. A Web Application Firewall (WAF) is a Layer 7 security appliance deployed in front of web applications to inspect decrypted HTTP and HTTPS traffic. While traditional L4 firewalls only inspect ports and IP headers, a WAF parses application parameters, cookies, and JSON bodies to detect and block OWASP Top 10 vulnerabilities like SQL Injection, Cross-Site Scripting, and SSRF."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Believing a traditional firewall replaces the need for a WAF.  
  *Correction:* An enterprise firewall must leave Port 443 open for web traffic to function; once port 443 is open, an attacker can send SQL injection payloads through that port unimpeded. Only a Layer 7 WAF can inspect inside the decrypted HTTP traffic to stop application exploits.
- **Trap:** Forgetting SSL termination in proxy architecture.  
  *Correction:* A reverse proxy often performs **SSL Offloading / Termination**, decrypting HTTPS at the edge so backend web applications and WAF engines can process plain HTTP requests without incurring cryptographic CPU overhead.
