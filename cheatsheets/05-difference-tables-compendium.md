# Critical Difference Tables Compendium

> **Cybersecurity Interview Focus:** Technical interviewers frequently probe your depth by asking *"What is the difference between X and Y?"* This compendium covers the top 10 most commonly tested technical distinctions.

---

## 1. Process vs. Thread (Operating Systems)

| Dimension | Process | Thread |
| :--- | :--- | :--- |
| **Definition** | An independent program in execution with its own address space. | A lightweight execution unit within a parent process. |
| **Memory & Address Space** | Isolated address space (Text, Data, Heap, Stack). Cannot access other processes' memory without IPC. | Shares code, data, and heap with peer threads; has its own private stack and registers. |
| **Creation & Context Switch**| High overhead; requires page table updates and cache flushes. | Low overhead; shares page tables and virtual memory space. |
| **Communication** | Requires Inter-Process Communication (IPC: Pipes, Sockets, Shared Memory). | Direct communication via shared memory / heap variables (requires synchronization). |
| **Failure Blast Radius** | If one process crashes, other processes are completely unaffected. | If one thread crashes (e.g. SegFault), it typically crashes the entire process. |
| **Security Implication** | Process boundaries enforce OS isolation (sandboxing, DAC). | Vulnerable to race conditions, TOCTOU, and heap corruption across threads. |

---

## 2. TCP vs. UDP (Transport Layer Networking)

| Dimension | TCP (Transmission Control Protocol) | UDP (User Datagram Protocol) |
| :--- | :--- | :--- |
| **Connection State** | Connection-oriented (3-way handshake required). | Connectionless (fire and forget; no handshake). |
| **Reliability** | Guaranteed delivery (acknowledgments, retransmissions, sequencing). | Best-effort delivery; packets may drop, arrive out of order, or duplicate. |
| **Flow & Congestion Control**| Yes (sliding window, slow start, congestion avoidance). | None. Transmits as fast as the application feeds the socket. |
| **Header Size** | 20 to 60 bytes. | Fixed 8 bytes. |
| **Primary Use Cases** | Web (HTTP/HTTPS), SSH, FTP, Database traffic, Email. | Streaming, Gaming, VoIP, DNS queries, SNMP, DHCP. |
| **Cybersecurity Threat Profile**| SYN flood, Connection exhaustion, Session hijacking. | UDP Amplification DDoS (DNS/NTP reflection spoofing). |

---

## 3. Authentication vs. Authorization (IAM & Security)

| Dimension | Authentication (AuthN) | Authorization (AuthZ) |
| :--- | :--- | :--- |
| **Question Answered** | *"Who are you?"* (Identity verification) | *"What are you allowed to do?"* (Permission check) |
| **Order of Execution** | **First** (Precedes authorization). | **Second** (Follows successful authentication). |
| **Mechanisms** | Passwords, MFA, Biometrics, Passkeys, Digital certificates. | RBAC, ABAC, ACLs, OAuth 2.0 scopes, IAM policies. |
| **Tokens / Artifacts** | ID Token (in OpenID Connect), Kerberos TGT. | Access Token (in OAuth 2.0), Kerberos Service Ticket. |
| **Common Attacks** | Credential stuffing, brute force, password spraying, phishing. | IDOR, Privilege Escalation (vertical & horizontal), BOLA. |

---

## 4. Encryption vs. Hashing (Cryptography)

| Dimension | Encryption | Hashing |
| :--- | :--- | :--- |
| **Reversibility** | **Two-way** (Ciphertext can be decrypted back to plaintext). | **One-way** (Digest cannot be reversed mathematically). |
| **Key Dependency** | Requires a key (symmetric secret key or asymmetric key pair). | Keyless by default (except HMACs which include a secret key). |
| **Output Size** | Proportional to input size (often slightly larger due to padding/IV). | Fixed length regardless of input (e.g., SHA-256 always outputs 256 bits). |
| **Primary Objective** | Data **Confidentiality** (protect secret data in transit/at rest). | Data **Integrity** (verify data has not been modified). |
| **Typical Use Cases** | TLS sessions, Disk encryption (BitLocker), encrypted databases. | Password storage (with salt), file integrity checksums, digital signatures. |

---

## 5. IDS vs. IPS vs. WAF (Network Security Devices)

| Dimension | IDS (Intrusion Detection System) | IPS (Intrusion Prevention System) | WAF (Web Application Firewall) |
| :--- | :--- | :--- | :--- |
| **Deployment Mode** | **Out-of-band / Passive** (via SPAN / TAP port). | **In-line** (directly in traffic flow). | **Reverse Proxy / In-line** in front of web servers. |
| **Action Taken** | Alerts only (Syslog, SNMP trap, SIEM alert). | Blocks / drops malicious packets in real time. | Inspects and terminates malicious HTTP/HTTPS requests. |
| **Network Layer** | Layer 3 and Layer 4 (Network & Transport). | Layer 3 and Layer 4 (Network & Transport). | **Layer 7 (Application Layer)** only. |
| **Inspection Capability**| Signatures and anomaly detection across IP/TCP. | Inline deep packet inspection, TCP resets. | Decodes HTTP payloads, URI parameters, JSON, cookies. |
| **Latency Impact** | Zero latency impact on live network traffic. | Adds millisecond latency to all passing packets. | Adds minimal latency to HTTP request/response pipeline. |
| **Attack Coverage** | Port scans, SYN floods, known exploit signatures. | DoS, exploit delivery, malware propagation. | SQLi, XSS, CSRF, SSRF, Path Traversal, Bot attacks. |

---

## 6. Containers vs. Virtual Machines (DevOps & Infrastructure)

| Dimension | Container (e.g., Docker) | Virtual Machine (VM) |
| :--- | :--- | :--- |
| **Architecture** | Shares the host OS kernel; isolates via namespaces & cgroups. | Runs a full guest OS on top of a Hypervisor (Type 1 or 2). |
| **Startup Time** | Milliseconds to seconds. | Minutes (full OS boot cycle). |
| **Size & Overhead** | Megabytes (lightweight user-space packages). | Gigabytes (contains full kernel, drivers, system utilities). |
| **Performance** | Near bare-metal CPU & I/O performance. | Virtualization overhead for CPU, memory, and disk I/O. |
| **Isolation Strength** | **Process-level isolation:** weaker isolation boundary. | **Hardware-level virtualization:** strict hardware boundaries. |
| **Security Risk** | Kernel exploit in container can compromise host (container breakout).| Hypervisor breakout is exceedingly rare; stronger multi-tenant security. |

---

## 7. Symmetric vs. Asymmetric Encryption (Cryptography)

| Dimension | Symmetric Encryption | Asymmetric Encryption |
| :--- | :--- | :--- |
| **Number of Keys** | 1 key (shared secret for both encryption and decryption). | 2 keys (mathematically paired: Public and Private). |
| **Speed & Computation** | Extremely fast (hardware-accelerated AES-NI instructions). | Slow (computationally expensive modular exponentiation). |
| **Key Distribution** | Difficult: how do you share the key securely before communicating?| Easy: Public key can be broadcast openly to the world. |
| **Key Lengths (Comparable)**| 128 - 256 bits (AES-256 is quantum-resistant). | 2048 - 4096 bits (RSA); 256 - 384 bits (ECC). |
| **Role in TLS** | Encrypts bulk application data once session is established. | Authenticates server (certificates) and exchanges the symmetric key. |

---

## 8. Git Merge vs. Git Rebase (Version Control)

| Dimension | `git merge` | `git rebase` |
| :--- | :--- | :--- |
| **History Structure** | Preserves complete non-linear branch history with merge commits. | Rewrites commit history into a single linear sequence. |
| **Safety** | Non-destructive; never alters existing commit hashes. | **Destructive on shared branches:** rewrites commit SHAs. |
| **Traceability** | Easy to trace when a feature branch was integrated. | Cleans up messy micro-commits before pull request review. |
| **Golden Rule** | Use for integrating branches into main/production. | **Never rebase a public branch that others are working on!** |

---

## 9. SAST vs. DAST vs. SCA (DevSecOps Security Testing)

| Dimension | SAST (Static Application Security Testing) | DAST (Dynamic Application Security Testing) | SCA (Software Composition Analysis) |
| :--- | :--- | :--- | :--- |
| **Testing State** | **White-box:** Analyzes source code at rest. | **Black-box:** Tests running application from outside. | Dependency manifest analysis (code at rest). |
| **Execution Required?**| No execution needed. | Application must be deployed and running. | No execution needed. |
| **Pipeline Stage** | Early (Pre-commit, Pull Request build). | Late (Staging, Pre-production deployment). | Continuous (Build time & repository scan). |
| **Finds Best** | Hardcoded secrets, bad crypto, SQLi code patterns. | Authentication bugs, server misconfigs, runtime XSS. | Known CVEs in open-source libraries (npm, PyPI). |
| **Limitation** | High false-positive rate; cannot see runtime context. | Cannot pinpoint exact source line number causing bug. | Only detects known CVEs; blind to custom business logic bugs. |

---

## 10. OAuth 2.0 vs. OpenID Connect (OIDC) vs. SAML 2.0 (Identity)

| Dimension | OAuth 2.0 | OpenID Connect (OIDC) | SAML 2.0 |
| :--- | :--- | :--- | :--- |
| **Primary Purpose** | **Authorization** (delegated access to APIs). | **Authentication** (identity layer on top of OAuth). | **Both AuthN & AuthZ** (enterprise SSO federation). |
| **Format** | JSON / HTTP (Access tokens, Refresh tokens). | JSON / REST (ID Tokens as JWTs + Access tokens). | XML-based Assertions and SOAP messages. |
| **Target Environment**| Mobile apps, Single Page Apps, Modern REST APIs. | Modern web & cloud platforms, Google/Apple login. | Legacy Enterprise, Active Directory, Okta, Workday. |
| **Token Representation**| Opaque token or JWT (Access Token). | JWT (ID Token containing user identity claims). | Base64-encoded XML document with digital signature. |
