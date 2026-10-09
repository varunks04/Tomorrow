# DNS Resolution Flow, Record Types, DNSSEC & Security Exploits

> **Domain:** Networking Fundamentals  
> **Sub-Domain:** Core Network Protocols & Infrastructure  
> **Interview Importance:** Critical / Fundamental System Architecture Question  

---

## 1. Topic & Definitions

- **Domain Name System (DNS):** The hierarchical, decentralized naming system that translates human-readable hostnames (e.g., `www.example.com`) into machine-routable IP addresses (e.g., `93.184.216.34`). Operates at Layer 7 on **Port 53** (UDP for standard lookups, TCP for responses exceeding 512 bytes, zone transfers, and DNSSEC).
- **Recursive Resolver (Local DNS Server):** A DNS server (provided by your ISP, corporate network, or public resolvers like `8.8.8.8` / `1.1.1.1`) that accepts recursive queries from client stub resolvers and traverses the global DNS hierarchy on their behalf to return the final answer.
- **Authoritative Nameserver:** The official nameserver that holds the actual definitive DNS record zone files for a specific domain name (e.g., Cloudflare or AWS Route 53 hosting records for `example.com`).

---

## 2. Step-by-Step DNS Resolution Architecture

```text
[ Client Browser / Stub Resolver ]
       │
       │ 1. Recursive Query: "What is the IP of www.example.com?"
       ▼
[ Recursive Resolver (e.g., 8.8.8.8) ] ── (Checks local cache first; if cache miss...)
       │
       ├─ 2. Iterative Query: "Who knows www.example.com?" ──► [ 13 Root Servers (.) ]
       │◄─ 3. Referral: "Ask the .com TLD Server at IP X" ────┘
       │
       ├─ 4. Iterative Query: "Who knows www.example.com?" ──► [ .com TLD Nameservers ]
       │◄─ 5. Referral: "Ask Authoritative NS for example.com" ┘
       │
       ├─ 6. Iterative Query: "Give me IP for www.example.com" ─► [ Authoritative NS (ns1.example.com) ]
       │◄─ 7. Authoritative Answer: "www.example.com A 93.184.216.34" ┘
       │
       │ 8. Caches answer for TTL seconds & returns IP to Client
       ▼
[ Client Browser ] ──► Connects to 93.184.216.34 on Port 443!
```

### Recursive Query vs. Iterative Query
- **Recursive Query:** The client asks the resolver: *"Find me the complete answer; do not return until you have the final IP address."*
- **Iterative Query:** The resolver asks upstream servers: *"Give me the IP or give me the next server to ask."* The server responds with referrals.

---

## 3. Essential DNS Record Types

| Record Type | Purpose & Description | Practical Example | Cybersecurity Importance |
| :---: | :--- | :--- | :--- |
| **`A`** | Maps a hostname to a 32-bit **IPv4 address**. | `example.com. IN A 93.184.216.34` | Primary target for DNS spoofing and phishing redirects. |
| **`AAAA`** | Maps a hostname to a 128-bit **IPv6 address**. | `example.com. IN AAAA 2606:2800:220:1:248:1893:25c8:1946` | Must be protected identically to A records. |
| **`CNAME`** | **Canonical Name (Alias):** Maps one domain name to another domain. | `www.example.com. IN CNAME example.com.` | Subdomain Takeover: If target points to dangling cloud resource, attacker claims it. |
| **`MX`** | **Mail Exchange:** Specifies mail servers responsible for accepting incoming email. | `example.com. IN MX 10 mail.example.com.` | Email routing, phishing defense, intercepting corporate mail. |
| **`TXT`** | Stores arbitrary text data. | `v=spf1 include:_spf.google.com ~all` | **Crucial:** Hosts email security policies (**SPF, DKIM, DMARC**) and domain ownership verification. |
| **`NS`** | **Name Server:** Delegates a DNS zone to authoritative nameservers. | `example.com. IN NS ns1.cloudflare.com.` | Hijacking NS records compromises entire domain control. |
| **`PTR`** | **Pointer Record:** Performs **Reverse DNS (rDNS)** lookup (IP to domain). | `34.216.184.93.in-addr.arpa. IN PTR example.com.` | Anti-spam mail server verification; network forensic attribution. |
| **`SOA`** | **Start of Authority:** Contains core zone administration metadata. | Serial number, Refresh timer, Retry timer, Expiry, Minimum TTL. | Zone synchronization and replication tracking between primary and secondary servers. |

---

## 4. DNS Security Enhancements: DNSSEC, DoH & DoT

### 1. DNSSEC (DNS Security Extensions)
- Traditional DNS has **zero cryptographic authentication**; responses can be forged by any attacker between the resolver and authoritative server.
- **How DNSSEC Works:**
  - Authoritative servers cryptographically sign DNS records using public-key cryptography.
  - Adds new record types: **`RRSIG`** (Signature of record set), **`DNSKEY`** (Public key), and **`DS`** (Delegation Signer hash placed in parent zone).
  - Establishes a cryptographic **Chain of Trust** anchored in the ICANN Root Zone.
  - **Guarantee:** Guarantees **Data Integrity** and **Origin Authenticity**. (Does NOT encrypt queries; queries remain readable in cleartext).

### 2. DNS-over-HTTPS (DoH - Port 443) vs. DNS-over-TLS (DoT - Port 853)
- Traditional DNS transmits queries in cleartext on Port 53, allowing local eavesdroppers and ISPs to monitor visited websites.
- **DoT (Port 853):** Encapsulates DNS queries inside a dedicated TLS tunnel. Easily monitored and controlled by enterprise firewalls.
- **DoH (Port 443):** Encapsulates DNS queries inside standard HTTPS traffic on port 443. Indistinguishable from ordinary web browsing, defeating ISP surveillance but complicating enterprise security inspection.

---

## 5. Cybersecurity Relevance & Threat Vectors

### 1. DNS Cache Poisoning (Kaminsky Attack)
- An attacker floods a recursive resolver with forged UDP responses guessing the 16-bit **Transaction ID (TXID)** and source port before the authoritative server can respond.
- If successful, the resolver caches the attacker's fraudulent IP address, redirecting thousands of legitimate users to phishing replicas.
- **Mitigation:** Source Port Randomization (SPR) and universal DNSSEC validation.

### 2. DNS Tunneling & Covert Data Exfiltration
- Firewalls almost never block outbound DNS queries on port 53.
- Attackers run custom malware that breaks sensitive stolen files (e.g. passwords, credit cards) into Base64/Hex strings and issues DNS queries:
  `416c696365536563.exfil.attacker-c2.com`
- The query naturally routes through corporate DNS servers directly to the attacker's authoritative server.
- **Detection:** Monitoring for high query volumes, high entropy in subdomains, and anomalous TXT record requests.

### 3. Subdomain Takeover
- A company creates a CNAME pointing `blog.company.com` to an AWS S3 bucket or GitHub Pages site. Later, the marketing team deletes the S3 bucket but forgets to delete the DNS CNAME record.
- An attacker registers the abandoned S3 bucket name, gaining complete control over `blog.company.com`, stealing cookies and bypassing Cross-Origin Resource Sharing (CORS).

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"DNS is the internet's phonebook, translating hostnames to IP addresses across a hierarchical system of Root servers, TLD nameservers, and Authoritative servers. When a client performs a recursive query, its resolver traverses this hierarchy iteratively. Essential records include A for IPv4, AAAA for IPv6, CNAME for aliases, MX for mail, and TXT for SPF and DMARC email authentication. Because traditional DNS uses cleartext UDP port 53 without authentication, it is vulnerable to Cache Poisoning, spoofing, and covert data tunneling. We mitigate these risks using DNSSEC for cryptographic response signing, Source Port Randomization, and encrypted transport via DoH and DoT."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Believing DNSSEC encrypts DNS traffic.  
  *Correction:* DNSSEC provides **Authentication and Integrity** via digital signatures; it provides **ZERO Confidentiality**. Anyone sniffing traffic can still read the queried domain names. Confidentiality requires DoH or DoT.
- **Trap:** Forgetting that DNS uses TCP.  
  *Correction:* While standard DNS queries use UDP port 53 for speed, DNS automatically falls back to **TCP port 53** whenever a response payload exceeds 512 bytes (such as large DNSSEC responses) or during Zone Transfers (`AXFR`).
