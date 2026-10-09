# Top 50 Essential Ports & Protocols Cheatsheet

> **Cybersecurity Interview Focus:** Know the port number, transport protocol (TCP/UDP), default encryption status, risk profile, and secure alternatives.

| Port | Protocol / Service | Transport | Security Status | Default Risk / Common Attack | Secure Alternative |
| :---: | :--- | :---: | :---: | :--- | :--- |
| **20, 21** | **FTP** (File Transfer Protocol) | TCP | Cleartext | Credential sniffing, Anonymous login abuse | **SFTP** (Port 22), **FTPS** (Port 989/990) |
| **22** | **SSH / SFTP** (Secure Shell) | TCP | Encrypted | Brute force, exposed root login, weak keys | Key-based auth, Fail2ban, non-standard port |
| **23** | **Telnet** | TCP | Cleartext | Complete packet sniffing of passwords | **SSH** (Port 22) |
| **25** | **SMTP** (Simple Mail Transfer) | TCP | Cleartext (STARTTLS) | Open mail relay, spoofing, spamming | **SMTPS** (Port 465) / STARTTLS (Port 587) |
| **49** | **TACACS+** | TCP | Encrypted payload | Man-in-the-Middle if pre-shared key is weak | RADIUS with EAP-TLS |
| **53** | **DNS** (Domain Name System) | UDP/TCP | Cleartext | DNS cache poisoning, DNS amplification DDoS | **DoH** (Port 443), **DoT** (Port 853), DNSSEC |
| **67, 68** | **DHCP** | UDP | Cleartext | Rogue DHCP server, DHCP starvation | DHCP Snooping on switches |
| **69** | **TFTP** (Trivial FTP) | UDP | Cleartext | No authentication; trivial unauthorized file exfil | SFTP / SCP |
| **80** | **HTTP** | TCP | Cleartext | Eavesdropping, MITM, session hijacking | **HTTPS** (Port 443) + HSTS |
| **88** | **Kerberos** | TCP/UDP | Encrypted | Kerberoasting, AS-REP Roasting, Ticket forgery | AES encryption, strong SPN passwords |
| **110** | **POP3** | TCP | Cleartext | Plaintext email and credential interception | **POP3S** (Port 995) |
| **123** | **NTP** (Network Time Protocol) | UDP | Cleartext | NTP amplification DDoS, time desynchronization | NTS (Network Time Security) |
| **135** | **MS RPC** (Remote Procedure Call) | TCP | Cleartext/Varies | EternalBlue precursor, lateral movement | Firewall off external access |
| **137-139**| **NetBIOS** | UDP/TCP | Cleartext | NetBIOS name poisoning (Responder), LLMNR abuse | Disable NetBIOS over TCP/IP |
| **143** | **IMAP** | TCP | Cleartext | Cleartext email credential theft | **IMAPS** (Port 993) |
| **161, 162**| **SNMP** (v1/v2c) | UDP | Cleartext | Community string brute force (`public`/`private`) | **SNMPv3** (Encrypted + Authenticated) |
| **389** | **LDAP** | TCP/UDP | Cleartext | Cleartext directory queries, credential exposure | **LDAPS** (Port 636) / StartTLS |
| **443** | **HTTPS** (TLS) | TCP | Encrypted | TLS misconfigurations, expired certs, weak ciphers | TLS 1.3, strict cipher suites |
| **445** | **SMB** (Server Message Block) | TCP | Cleartext/Enc | EternalBlue (WannaCry), Pass-the-Hash, SMB Relay | SMB Signing, SMBv3 encryption, block 445 on WAN |
| **465** | **SMTPS** | TCP | Encrypted | Legacy SSL wrappers; credential spraying | Port 587 with STARTTLS / modern TLS |
| **500** | **ISAKMP / IKE** (IPsec VPN) | UDP | Encrypted negotiation | Pre-shared key cracking, aggressive mode leaks | Main mode, strong Diffie-Hellman groups |
| **514** | **Syslog** | UDP | Cleartext | Log tampering, log spoofing, packet loss | **Syslog-over-TLS** (Port 6514) |
| **587** | **SMTP Submission** | TCP | Encrypted (STARTTLS) | Credential stuffing on auth submission | Enforce MFA, OAuth2 for email |
| **636** | **LDAPS** (LDAP over SSL) | TCP | Encrypted | Self-signed cert trust bypass | Valid enterprise CA cert verification |
| **853** | **DNS-over-TLS** (DoT) | TCP | Encrypted | Metadata/traffic analysis (fixed port) | DoH (Port 443 hides inside HTTPS) |
| **993** | **IMAPS** | TCP | Encrypted | Outdated TLS versions (TLS 1.0/1.1) | Enforce TLS 1.2+ |
| **995** | **POP3S** | TCP | Encrypted | Outdated TLS versions | Enforce TLS 1.2+ |
| **1433** | **MS SQL Server** | TCP | Cleartext/Enc | SQL injection, default `sa` brute forcing | Port binding to localhost, TLS enforce |
| **1521** | **Oracle DB** | TCP | Cleartext/Enc | TNS listener attacks, default passwords | Restrict via firewall, TLS encryption |
| **3306** | **MySQL** | TCP | Cleartext/Enc | Remote root exposure, SQL injection | Bind to `127.0.0.1`, TLS, least privilege |
| **3389** | **RDP** (Remote Desktop) | TCP/UDP | Encrypted | BlueKeep, brute force, ransomware gateway | RDP Gateway, NLA enabled, MFA, VPN only |
| **5432** | **PostgreSQL** | TCP | Cleartext/Enc | Misconfigured `pg_hba.conf` trust auth | Host-based TLS authentication |
| **5985, 5986**| **WinRM** (HTTP / HTTPS) | TCP | HTTP / Encrypted | PowerShell remoting lateral movement | Enforce 5986 (HTTPS), restrict admin IPs |
| **6379** | **Redis** | TCP | Cleartext | Unauthenticated remote code execution (RCE) | Requirepass, bind to localhost, TLS |
| **8080, 8443**| **HTTP/HTTPS Alternate** | TCP | Cleartext/Enc | Shadow IT, unpatched administrative consoles | Inventory scanning, enforce WAF/HTTPS |
| **8888, 9000**| **App Dev / Portainer** | TCP | Cleartext/Enc | Default admin setups exposed to the public internet | Restrict access via VPN/Zero Trust |

---

## 🎙️ Interview Hot Take: What Interviewers Listen For
1. **Never say "Port 22 is secure".** Say: *"Port 22 runs SSH, which encrypts transport, but remains vulnerable to brute force and key mismanagement unless hardened with key-only auth, rate limiting, and fail2ban."*
2. **Differentiate 389 vs 636 immediately:** *"389 is cleartext LDAP (or StartTLS); 636 is LDAPS (LDAP over SSL/TLS)."*
3. **Know the SMB vulnerability history:** *"Port 445 is SMB. It was the vector for MS17-010 EternalBlue exploited by WannaCry and NotPetya. It should NEVER be exposed to the public internet."*
