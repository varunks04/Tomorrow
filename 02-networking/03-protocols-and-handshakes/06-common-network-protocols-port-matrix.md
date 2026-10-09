# Common Network Protocols, Services & Security Port Matrix

> **Domain:** Networking Fundamentals  
> **Sub-Domain:** Network Services, Protocols & Threat Profiles  
> **Interview Importance:** Very High / Foundational Service Triage & Port Enumeration  

---

## 1. Remote Administration Protocols

### 1. SSH (Port 22 / TCP) vs. Telnet (Port 23 / TCP)
- **Telnet (Port 23):** Completely **unencrypted cleartext** protocol. Transmits usernames, passwords, and terminal keystrokes in raw ASCII packets. Anyone running Wireshark or an ARP spoofing attack on the local subnet captures root credentials instantaneously. Completely obsolete.
- **SSH (Secure Shell - Port 22):** Cryptographically secures remote command execution, file transfer (SFTP), and port forwarding using public-key cryptography and AES/ChaCha20 symmetric encryption.
- **Hardening SSH:**
  - Disable root password login: `PermitRootLogin prohibit-password` or `no`.
  - Disable password authentication: `PasswordAuthentication no` (Enforce Ed25519/RSA public-key authentication).
  - Deploy **Fail2ban** or rate limiting to thwart credential brute-force attempts.

### 2. RDP (Remote Desktop Protocol - Port 3389 / TCP & UDP)
- Proprietary Microsoft protocol providing a graphical remote desktop interface to Windows systems.
- **Threat Profile:** **Primary initial access vector for enterprise Ransomware groups.** Attackers scan the public internet for exposed port 3389 and launch brute-force or credential-stuffing attacks. Historical vulnerabilities include **BlueKeep (CVE-2019-0708)** allowing pre-authentication Remote Code Execution (RCE).
- **Hardening RDP:** Never expose port 3389 directly to the public WAN; require VPN or Remote Desktop Gateway; enforce **Network Level Authentication (NLA)** and MFA.

---

## 2. File Transfer Protocols

```text
+─────────────────────────────────────────────────────────────────────────────+
|                         FILE TRANSFER PROTOCOL MATRIX                       |
+─────────────────────────────────────────────────────────────────────────────+
| FTP (Ports 20 & 21 / TCP):                                                  |
|   • Port 21 = Control/Commands | Port 20 = Data channel.                    |
|   • Insecure cleartext. Anonymous login abuse. Active vs Passive FTP modes. |
|─────────────────────────────────────────────────────────────────────────────|
| SFTP (Port 22 / TCP):                                                       |
|   • SSH File Transfer Protocol. Runs completely inside an SSH tunnel.       |
|   • Single port (22), encrypted authentication and data transfer.           |
|─────────────────────────────────────────────────────────────────────────────|
| FTPS (Ports 990 / 989 or Explicit 21 with TLS):                             |
|   • FTP over SSL/TLS. Complex multi-port firewall traversal.                |
|─────────────────────────────────────────────────────────────────────────────|
| TFTP (Port 69 / UDP):                                                       |
|   • Trivial FTP. No authentication, zero encryption. Used strictly for      |
|     local network PXE netbooting and Cisco router firmware uploads.         |
|─────────────────────────────────────────────────────────────────────────────|
| SMB (Port 445 / TCP):                                                       |
|   • Server Message Block. Windows network file and printer sharing.         |
|   • Target of MS17-010 EternalBlue (WannaCry). Must be blocked on WAN!     |
+─────────────────────────────────────────────────────────────────────────────+
```

---

## 3. Email Infrastructure Protocols

```text
                   OUTBOUND MAIL                          INBOUND MAIL
                (MTA to MTA Routing)                  (Mailbox Retrieval)
                         │                                     │
                         ▼                                     ▼
                   [ SMTP / SMTPS ]                   [ IMAP / POP3 ]
```

### 1. SMTP (Simple Mail Transfer Protocol)
- **Port 25 (Cleartext / Relay):** Server-to-server mail routing between Mail Transfer Agents (MTAs). Susceptible to Open Relay exploitation if unauthenticated.
- **Port 587 (Submission):** Modern client-to-server mail submission enforcing authentication and opportunistic TLS encryption via `STARTTLS`.
- **Port 465 (SMTPS):** Implicit SSL/TLS encrypted mail submission.

### 2. POP3 vs. IMAP
- **POP3 (Port 110 / Cleartext, Port 995 / POP3S Encrypted):** Downloads emails from server to local client and deletes them from the server by default. Single-device workflow.
- **IMAP (Port 143 / Cleartext, Port 993 / IMAPS Encrypted):** Synchronizes emails across multiple devices, leaving messages stored on the centralized mail server in folders.

---

## 4. Directory & Network Management Protocols

### 1. LDAP (Port 389 / TCP) vs. LDAPS (Port 636 / TCP)
- **Lightweight Directory Access Protocol (LDAP):** Protocol for querying and modifying centralized directory services (Active Directory, OpenLDAP) to retrieve user accounts, group memberships, and computer records.
- **The Security Risk:** Standard LDAP (Port 389) sends directory queries and user bind authentication passwords in **cleartext**.
- **The Standard:** Enforce **LDAPS (Port 636)** over TLS or require LDAP StartTLS over Port 389 with channel binding to defeat credential sniffing.

### 2. SNMP (Simple Network Management Protocol - Ports 161 & 162 / UDP)
- Used to monitor network hardware metrics (bandwidth, CPU utilization, interface status).
- **SNMPv1 & SNMPv2c Flaw:** Uses cleartext passwords called **"Community Strings"** (commonly left as default `public` for read-only or `private` for read-write). Attackers sniff community strings to map the entire network topology or reconfigure switches.
- **SNMPv3:** Industry standard introducing **USM (User-based Security Model)** providing User Authentication (`HMAC-SHA`) and Payload Encryption (`AES-256`).

### 3. NTP (Network Time Protocol - Port 123 / UDP)
- Synchronizes clock timestamps across all network servers, routers, and domain controllers.
- **Security Role:** Kerberos authentication tickets fail if client and server clocks drift by more than 5 minutes (skew defense). SIEM log correlation across multiple firewalls requires synchronized UTC clocks.
- **Attack Vector:** NTP Amplification DDoS via the `monlist` command; time shifting attacks.

---

## 5. Master Port & Protocol Reference Matrix

| Port | Service | Transport | Security Status | Default Risk / Common Attack |
| :---: | :--- | :---: | :---: | :--- |
| **20, 21** | FTP | TCP | Cleartext | Credential sniffing, anonymous login |
| **22** | SSH / SFTP | TCP | Encrypted | Brute force, exposed root |
| **23** | Telnet | TCP | Cleartext | Plaintext credential sniffing |
| **25** | SMTP | TCP | Cleartext | Open relay, spoofing, spamming |
| **53** | DNS | UDP/TCP | Cleartext | Cache poisoning, DNS exfiltration |
| **67, 68** | DHCP | UDP | Cleartext | Starvation, Rogue DHCP MitM |
| **69** | TFTP | UDP | Cleartext | Unauthorized firmware/config exfil |
| **80** | HTTP | TCP | Cleartext | Unencrypted web traffic, MITM |
| **88** | Kerberos | TCP/UDP | Encrypted | Kerberoasting, AS-REP Roasting |
| **110** | POP3 | TCP | Cleartext | Email credential interception |
| **123** | NTP | UDP | Cleartext | NTP Amplification DDoS |
| **135** | MS RPC | TCP | Cleartext/Enc | Lateral movement, endpoint enumeration |
| **137-139**| NetBIOS | UDP/TCP | Cleartext | LLMNR/NBT-NS spoofing (Responder) |
| **143** | IMAP | TCP | Cleartext | Email credential interception |
| **161, 162**| SNMP | UDP | Cleartext (v1/v2) | Community string sniffing (`public`) |
| **389** | LDAP | TCP/UDP | Cleartext | Plaintext directory query sniffing |
| **443** | HTTPS | TCP | Encrypted | Encrypted web application traffic |
| **445** | SMB | TCP | Cleartext/Enc | EternalBlue (MS17-010), Pass-the-Hash |
| **465** | SMTPS | TCP | Encrypted | Encrypted email submission |
| **514** | Syslog | UDP | Cleartext | Log tampering, packet loss |
| **587** | SMTP Submission | TCP | STARTTLS | Encrypted mail submission |
| **636** | LDAPS | TCP | Encrypted | Encrypted directory services |
| **993** | IMAPS | TCP | Encrypted | Encrypted mail retrieval |
| **995** | POP3S | TCP | Encrypted | Encrypted mail retrieval |
| **1433** | MS SQL | TCP | Cleartext/Enc | Database brute force, SQLi |
| **3306** | MySQL | TCP | Cleartext/Enc | Remote root exposure, SQLi |
| **3389** | RDP | TCP/UDP | Encrypted | BlueKeep, ransomware initial access |

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Network services use standardized well-known ports (0 to 1023) operating over TCP for reliable connections or UDP for low-latency transmission. In modern security operations, legacy cleartext protocols—such as Telnet on 23, FTP on 21, HTTP on 80, and LDAP on 389—must be systematically eradicated and replaced with encrypted alternatives: SSH/SFTP on 22, HTTPS on 443, and LDAPS on 636. High-risk administrative interfaces like SMB (445) and RDP (3389) represent the primary vectors for lateral movement (EternalBlue) and ransomware entry; they must never be exposed to the public WAN and should be isolated behind VPNs with multi-factor authentication."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing SFTP with FTPS.  
  *Correction:* **SFTP** (SSH File Transfer Protocol) runs entirely over port 22 inside SSH. **FTPS** (FTP Secure) is traditional FTP wrapped in SSL/TLS certificates, requiring multiple ports. They are completely different protocols!
- **Trap:** Forgetting that Kerberos uses Port 88.  
  *Correction:* Kerberos operates on **Port 88** (both UDP and TCP). Memorize this port as it is central to Active Directory interview questions.
