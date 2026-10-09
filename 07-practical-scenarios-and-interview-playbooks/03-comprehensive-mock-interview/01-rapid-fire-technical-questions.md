# 50 Rapid-Fire Technical Questions & Golden Answers Across All Domains

## Domain 1: Core Computer Science (OS, DBMS, OOP, DSA)

### Q1: What is the core difference between User Mode and Kernel Mode (Ring 3 vs Ring 0)?
> **Golden Answer:** User Mode (Ring 3) executes unprivileged application code with restricted memory access and no direct hardware control. Kernel Mode (Ring 0) executes with unrestricted CPU execution and full access to physical memory and hardware; transitioning between them requires a software interrupt or CPU system call (`syscall`).

### Q2: What is the difference between a Process and a Thread?
> **Golden Answer:** A process is an independent execution unit with its own private virtual memory space, file descriptors, and security context. A thread is a lightweight execution stream within a process that shares the parent process's memory (heap, code, global data) but maintains its own private stack and registers.

### Q3: What is the difference between Mutex and Semaphore?
> **Golden Answer:** A Mutex is a locking mechanism with ownership—only the thread that locked the mutex can unlock it. A Semaphore is a signaling mechanism with an integer counter that allows $N$ concurrent accesses, and any thread can signal/unlock it.

### Q4: Explain the difference between Stack and Heap memory.
> **Golden Answer:** Stack memory is static, LIFO, managed automatically by the CPU, extremely fast, and stores local variables and function call frames. Heap memory is dynamic, managed manually by the programmer (or garbage collector), slower, and stores dynamically allocated objects of variable size.

### Q5: How does a classic Stack-based Buffer Overflow exploit work?
> **Golden Answer:** By writing more data into a fixed-size stack buffer than it can hold, an attacker overwrites adjacent stack memory, specifically targeting the Saved Frame Pointer (SFP) and the Return Address (`$RIP`), redirecting CPU execution flow to injected shellcode or a ROP gadget chain.

### Q6: What are the ACID properties in database transactions?
> **Golden Answer:** Atomicity (all operations complete or all rollback), Consistency (database transitions from one valid state to another maintaining all constraints), Isolation (concurrent transactions execute independently without interference), and Durability (committed transactions persist permanently even across power failures).

### Q7: Explain SQL Injection and its primary mitigation.
> **Golden Answer:** SQL Injection occurs when untrusted user input is concatenated directly into SQL query strings, allowing attackers to manipulate the SQL parser. The only non-negotiable mitigation is **Parameterized Queries (Prepared Statements)**, which mathematically separate the code grammar from data variables.

### Q8: What is the difference between Clustered and Non-Clustered Indexes?
> **Golden Answer:** A Clustered Index physically dictates the on-disk order of the actual table rows (only one per table). A Non-Clustered Index is a separate B+ tree structure containing index keys and pointers (row locators) back to the actual table data (multiple allowed per table).

### Q9: Name the 4 Pillars of Object-Oriented Programming (OOP).
> **Golden Answer:** Encapsulation (bundling data and methods while restricting direct access), Abstraction (hiding complex internal implementation details), Inheritance (deriving new classes from existing classes), and Polymorphism (allowing entities to take on multiple forms or behaviors).

### Q10: What is a Bloom Filter and why is it used in security?
> **Golden Answer:** A Bloom Filter is a space-efficient probabilistic data structure used to test set membership that can yield false positives but **never false negatives**. Security engines use Bloom Filters to check billions of malicious URLs or password breach hashes in memory before triggering expensive disk/database lookups.

---

## Domain 2: Networking Fundamentals

### Q11: Explain the difference between the OSI Model and TCP/IP Model.
> **Golden Answer:** The OSI model is a 7-layer theoretical framework (Physical, Data Link, Network, Transport, Session, Presentation, Application). The TCP/IP model is a 4-layer practical networking architecture (Network Access, Internet, Transport, Application) that powers the modern internet.

### Q12: What is the TCP 3-Way Handshake?
> **Golden Answer:** The handshake establishes a reliable Layer-4 connection: Client sends `SYN` with Initial Sequence Number $X$; Server responds with `SYN-ACK` with Sequence $Y$ and Acknowledgement $X+1$; Client completes by sending `ACK` with Sequence $X+1$ and Acknowledgement $Y+1$.

### Q13: Why is UDP faster than TCP?
> **Golden Answer:** UDP is connectionless and stateless with zero handshake latency, zero sequence tracking, zero flow control, and zero retransmission overhead, featuring a tiny 8-byte header compared to TCP's 20-byte header.

### Q14: How does DNS work?
> **Golden Answer:** A client queries a Recursive Resolver, which iteratively queries the Root Nameserver (`.`), the TLD Nameserver (e.g., `.com`), and finally the Authoritative Nameserver for the specific domain, which returns the IP address record (`A`/`AAAA`).

### Q15: What is ARP and how does ARP Poisoning work?
> **Golden Answer:** Address Resolution Protocol (ARP) maps Layer-3 IP addresses to Layer-2 physical MAC addresses on local subnets. Because ARP is stateless and unauthenticated, an attacker sends unsolicited gratuitous ARP replies associating the gateway's IP with the attacker's MAC, executing an Adversary-in-the-Middle (AiTM) interception.

### Q16: What is the difference between NAT and PAT?
> **Golden Answer:** NAT (Network Address Translation) maps private IPs to public IPs (typically 1:1). PAT (Port Address Translation / NAT Overload) maps thousands of private internal IPs to a single public IP by tracking unique source Layer-4 port numbers.

### Q17: What is the difference between IDS and IPS?
> **Golden Answer:** An IDS (Intrusion Detection System) operates out-of-band via a network TAP/SPAN port, passively monitoring traffic and generating alerts without affecting throughput. An IPS (Intrusion Prevention System) sits directly inline in the packet path, actively dropping malicious packets in real time.

### Q18: What is the difference between a Forward Proxy and a Reverse Proxy?
> **Golden Answer:** A Forward Proxy sits in front of internal clients to filter, log, and anonymize outbound requests to the internet. A Reverse Proxy sits in front of internal web servers to provide load balancing, TLS termination, caching, and WAF protection for incoming traffic.

### Q19: What is the difference between WAF and Network Firewall?
> **Golden Answer:** A Network Firewall inspects Layer-3 and Layer-4 packet headers (IP addresses, ports, protocols). A Web Application Firewall (WAF) inspects Layer-7 application payloads, detecting web-specific attacks like SQL Injection, XSS, and CSRF inside HTTP requests.

### Q20: Explain the 5-step network connection troubleshooting process.
> **Golden Answer:** 1. Verify physical link and local IP assignment (Layer 1/2); 2. Ping default gateway and check ARP (Layer 2/3); 3. Ping external public IP `8.8.8.8` to verify routing (Layer 3); 4. Query DNS via `dig` to verify name resolution (Layer 7); 5. Test transport port reachability via `nc -zv` or `curl -vvv` (Layer 4/7).

---

## Domain 3: Cybersecurity Fundamentals

### Q21: What is the CIA Triad?
> **Golden Answer:** Confidentiality (preventing unauthorized disclosure of data), Integrity (preventing unauthorized modification or tampering), and Availability (ensuring timely, reliable access to resources for authorized users).

### Q22: What is the difference between Symmetric and Asymmetric Encryption?
> **Golden Answer:** Symmetric encryption uses the single shared secret key for both encryption and decryption (fast, bulk data; e.g., AES-256-GCM). Asymmetric encryption uses mathematically linked keypairs: a Public Key to encrypt and a Private Key to decrypt (slower, key exchange & signatures; e.g., RSA, ECC).

### Q23: Why do we Salt passwords before hashing?
> **Golden Answer:** A salt is a cryptographically random unique string appended to a password before hashing. It ensures that identical passwords produce completely different hash outputs, defeating pre-computed Rainbow Table attacks and preventing bulk dictionary cracking across multiple accounts.

### Q24: What is Perfect Forward Secrecy (PFS)?
> **Golden Answer:** PFS ensures that the compromise of a server's long-term private key does not compromise past recorded session communications. It achieves this by generating ephemeral Diffie-Hellman keypairs for every session that are deleted immediately after use.

### Q25: Explain Cross-Site Scripting (XSS) and its primary defense.
> **Golden Answer:** XSS occurs when an application includes untrusted data in an HTTP response without validation, executing malicious JavaScript in the victim's browser. Defenses require context-aware HTML/JavaScript output encoding, the `HttpOnly` cookie flag, and strict Content Security Policy (CSP).

### Q26: What is CSRF and how does `SameSite` mitigate it?
> **Golden Answer:** Cross-Site Request Forgery (CSRF) tricks an authenticated user's browser into submitting unauthorized HTTP requests to a target website that trusts the user's cookies. `SameSite=Lax/Strict` prevents the browser from automatically sending session cookies with cross-site requests, neutralizing CSRF.

### Q27: What is SSRF (Server-Side Request Forgery)?
> **Golden Answer:** SSRF occurs when an attacker forces a backend web server to make unauthorized HTTP requests to arbitrary domains or internal resources (such as the cloud Instance Metadata Service `169.254.169.254` or internal microservices).

### Q28: What is the difference between EDR and Antivirus?
> **Golden Answer:** Traditional Antivirus relies primarily on static file signatures to detect known malicious files on disk. Endpoint Detection and Response (EDR) continuously monitors endpoint behavioral telemetry (process lineage, memory injections, network connections), detecting fileless and novel attacks with remote isolation capabilities.

### Q29: What are the phases of the NIST Incident Response lifecycle?
> **Golden Answer:** 1. Preparation; 2. Detection and Analysis; 3. Containment, Eradication, and Recovery; 4. Post-Incident Activity (Lessons Learned).

### Q30: What is a Golden Ticket attack?
> **Golden Answer:** A post-exploitation attack where an adversary uses the compromised NTLM hash of the Active Directory `krbtgt` account to forge a Ticket Granting Ticket (TGT), granting unrestricted, permanent Domain Admin access across the entire Active Directory forest.

---

## Domain 4: Identity & Access Management (IAM)

### Q31: What is the difference between Authentication (AuthN) and Authorization (AuthZ)?
> **Golden Answer:** Authentication verifies identity ("Who are you?"); Authorization determines permissions and access rights ("What are you allowed to do?"). Authentication must always precede authorization.

### Q32: What is the difference between OAuth 2.0 and OpenID Connect (OIDC)?
> **Golden Answer:** OAuth 2.0 is an **Authorization Framework** for delegating API permissions using Access Tokens. OpenID Connect is an **Authentication Layer** built on top of OAuth 2.0 that introduces the signed JWT **ID Token** to verify user identity.

### Q33: Why is PKCE (Proof Key for Code Exchange) mandatory for public clients in OAuth 2.0?
> **Golden Answer:** Public clients (mobile apps, SPAs) cannot securely store client secrets. PKCE dynamically binds the authorization code exchange to a one-time SHA-256 hashed code challenge, preventing attackers on the client device from intercepting and redeeming the authorization code.

### Q34: What are the three parts of a JSON Web Token (JWT)?
> **Golden Answer:** Header (algorithm & token type), Payload (claims like `sub`, `exp`, `iss`), and Signature (cryptographic hash verifying token integrity). Parts are Base64URL-encoded and separated by periods (`.`).

### Q35: How does Kerberos authentication work at a high level?
> **Golden Answer:** Client requests a Ticket Granting Ticket (TGT) from the KDC Authentication Service (AS-REQ/AS-REP). Client presents the TGT to the Ticket Granting Service to obtain an application Service Ticket (TGS-REQ/TGS-REP). Client presents the Service Ticket to the target server to gain access (AP-REQ/AP-REP).

### Q36: What is Kerberoasting?
> **Golden Answer:** Any authenticated domain user requests a Kerberos Service Ticket (TGS) for a service account registered with a Service Principal Name (SPN). The ticket is encrypted with the service account's password hash, which the attacker extracts from memory and cracks offline using dictionary attacks.

### Q37: What is SAML 2.0 and where is it primarily used?
> **Golden Answer:** SAML 2.0 is an XML-based federated identity standard exchanging signed authentication assertions between an Identity Provider (IdP) and Service Provider (SP), predominantly used for enterprise B2B Software-as-a-Service (SaaS) Single Sign-On.

### Q38: Why are FIDO2 / WebAuthn Passkeys considered phishing-resistant?
> **Golden Answer:** Passkeys use public-key cryptography where credentials are mathematically bound to the specific browser domain origin (Relying Party ID). The hardware authenticator refuses to sign challenges for spoofed phishing domains, completely defeating reverse-proxy phishing kits like Evilginx.

---

## Domain 5: AI & Machine Learning in Cybersecurity

### Q39: Why is Precision often more critical than Recall in automated SOC blocking?
> **Golden Answer:** In automated blocking firewalls, a false positive (low precision) can shut down mission-critical business transactions or block legitimate customer checkouts. High precision ensures that automated containment actions only execute against high-confidence threats.

### Q40: What is the Base Rate Fallacy in security machine learning?
> **Golden Answer:** Because true malicious events represent a minuscule fraction of network traffic (<0.001%), even a model with 99% accuracy will produce overwhelming volumes of false positive alerts compared to true positives due to Bayes' Theorem.

### Q41: What is the difference between Direct and Indirect Prompt Injection?
> **Golden Answer:** Direct Prompt Injection occurs when a user directly prompts an LLM to override its instructions or safety filters. Indirect Prompt Injection occurs when an LLM ingests external data (web pages, PDFs, emails) containing embedded adversarial instructions that hijack the model's privileged tools.

### Q42: What is "Excessive Agency" in AI agents (OWASP LLM06)?
> **Golden Answer:** Excessive Agency occurs when an autonomous AI agent is granted broad permissions, destructive tools, or unchecked autonomy without human oversight, allowing hallucinations or prompt injections to trigger real-world damage like deleting production databases.

### Q43: What is Adversarial Machine Learning Evasion (e.g., FGSM)?
> **Golden Answer:** Evasion attacks generate subtle mathematical perturbations added to inputs at inference time (e.g., Fast Gradient Sign Method) that maximize model loss, causing neural networks to misclassify malware as benign without breaking the malware's execution.

---

## Domain 6: DevOps, Cloud & DevSecOps

### Q44: What are the four core object types in Git?
> **Golden Answer:** Blobs (raw file content bytes), Trees (directory structures and filenames), Commits (pointers to root tree, parents, author metadata), and Annotated Tags (permanent signed commit references).

### Q45: What is the difference between SAST and DAST?
> **Golden Answer:** SAST (Static Analysis) is a white-box source code scanner using AST traversal to find coding flaws early without running the app. DAST (Dynamic Analysis) is a black-box scanner that tests running applications over HTTP to find exploitable runtime vulnerabilities.

### Q46: What is a Software Bill of Materials (SBOM)?
> **Golden Answer:** A machine-readable, nested inventory of all software components, third-party open-source libraries, versions, and supply chain metadata used to build an application, standardized via CycloneDX or SPDX formats.

### Q47: What Linux kernel primitives create container isolation?
> **Golden Answer:** **Namespaces** isolate visibility (what a process can see: PID, Network, Mount, IPC, User), **Cgroups** enforce resource limits (how much CPU, memory, and PIDs a process can consume), and **OverlayFS** manages layered copy-on-write storage.

### Q48: What is the difference between AWS Security Groups and Network ACLs?
> **Golden Answer:** Security Groups are stateful firewalls operating at the instance/ENI level with allow-only rules. Network ACLs are stateless firewalls operating at the subnet boundary with ordered, numbered allow and deny rules.

### Q49: What is Envelope Encryption in AWS KMS?
> **Golden Answer:** A two-tier encryption hierarchy where a master KMS key encrypts a local Data Encryption Key (DEK). The data is encrypted locally using the plaintext DEK, which is immediately purged from RAM, while the encrypted DEK is stored alongside the ciphertext.

### Q50: How does IMDSv2 mitigate SSRF credential theft in AWS?
> **Golden Answer:** IMDSv2 requires session-oriented authentication: clients must execute an initial HTTP `PUT` request with a custom TTL header to obtain a session token before requesting metadata. Simple SSRF vulnerabilities cannot forge custom HTTP PUT headers, blocking credential extraction.
