# CSRF vs. SSRF: Request Forgery Mechanics & Enterprise Mitigations

> **Domain:** Cybersecurity Fundamentals & Application Security (AppSec)  
> **Sub-Domain:** Web Vulnerabilities & Request Forgery  
> **Interview Importance:** Very High / Frequently Tested Architectural Comparison  

---

## 1. Topic & Definitions

- **Request Forgery:** An attack class where an adversary tricks an entity into sending an unauthorized, forged HTTP request on the adversary's behalf.
- **Cross-Site Request Forgery (CSRF - CWE-352):** Exploits the trust a website has in a **victim's browser**. An attacker tricks a victim's browser into transmitting unauthorized commands to a web application where the victim is currently authenticated, abusing the browser's automatic inclusion of session cookies (**Ambient Authority**).
- **Server-Side Request Forgery (SSRF - CWE-918):** Exploits the trust a backend network has in a **vulnerable server**. An attacker forces the backend web server to make unauthorized HTTP/TCP requests to internal, private, or remote infrastructure that the attacker cannot reach directly.

---

## 2. Key Differences: CSRF vs. SSRF

| Dimension | CSRF (Cross-Site Request Forgery) | SSRF (Server-Side Request Forgery) |
| :--- | :--- | :--- |
| **Who makes the request?** | The **Victim's Browser (Client)** makes the request. | The **Backend Web Server** makes the request. |
| **Whose trust is abused?**| The server's trust in the *user's session cookies*. | The internal network's trust in the *server's IP/identity*.|
| **Direction of Traffic** | Inbound from client to target web server. | Outbound from target web server to internal network / cloud. |
| **Typical Target** | Changing user state (email, password, wire transfer).| Cloud metadata (`169.254.169.254`), internal microservices, DBs.|
| **Can attacker see response?**| **NO** (Blind to response due to Same-Origin Policy). | **YES** (If in-band) or **NO** (If blind SSRF). |
| **Primary Defenses** | Anti-CSRF Tokens & `SameSite=Strict` cookies. | Allowlisting domains, blocking RFC 1918 IPs, IMDSv2. |

---

## 3. Cross-Site Request Forgery (CSRF) Deep Dive

```text
1. User Alice logs into https://bank.com.
   Bank sets session cookie: Set-Cookie: session_id=abc123xyz; (Lacks SameSite attribute!)

2. While still logged in, Alice visits malicious site: https://evil.com.

3. evil.com contains hidden HTML form that auto-submits via JavaScript:
   <form id="pwn" action="https://bank.com/transfer" method="POST">
       <input type="hidden" name="to_account" value="AttackerAccount" />
       <input type="hidden" name="amount" value="10000" />
   </form>
   <script>document.getElementById('pwn').submit();</script>

4. Alice's browser executes the POST request to bank.com.
   THE FATAL STEP: The browser automatically attaches bank.com's session_id cookie!

5. bank.com receives request: verifies cookie is Alice -> Transfers $10,000 to Attacker!
```

### Essential Defenses Against CSRF

```text
+─────────────────────────────────────────────────────────────────────────────+
|                         DEFENSE-IN-DEPTH AGAINST CSRF                       |
+─────────────────────────────────────────────────────────────────────────────+
| 1. ANTI-CSRF SYNCHRONIZER TOKENS:                                           |
|    • Server generates a cryptographically random, unpredictable token       |
|      tied to the user's session and inserts it into HTML forms:              |
|      <input type="hidden" name="csrf_token" value="d9f8a7b6c5e4..." />      |
|    • Upon form submission, the server verifies that the submitted token     |
|      matches the session token. Evil.com cannot guess or read this token!   |
|─────────────────────────────────────────────────────────────────────────────|
| 2. SAMESITE COOKIE ATTRIBUTE:                                               |
|    • Set-Cookie: session_id=xyz; SameSite=Strict; Secure; HttpOnly          |
|    • SameSite=Strict: Browser NEVER sends the cookie on cross-site requests.|
|    • SameSite=Lax: Cookie blocked on cross-site POST/state-changing requests.|
|─────────────────────────────────────────────────────────────────────────────|
| 3. RE-AUTHENTICATION / STEP-UP AUTH:                                        |
|    • Require users to re-enter their password or MFA before executing       |
|      high-risk actions (password reset, email change, wire transfer).        |
|─────────────────────────────────────────────────────────────────────────────|
| 4. CUSTOM HEADERS (For REST APIs):                                          |
|    • Require custom headers like X-Requested-With: XMLHttpRequest. Browsers |
|      block cross-origin sites from setting custom headers via CORS.         |
+─────────────────────────────────────────────────────────────────────────────+
```

---

## 4. Server-Side Request Forgery (SSRF) Deep Dive

SSRF occurs when an application exposes features that fetch resources from user-supplied URLs (e.g. *"Import image from URL"*, webhook testing, PDF generation, URL previews).

```text
Attacker sends:
POST /api/fetch-preview
{ "image_url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-role" }
                                │
                                ▼
[ Vulnerable Web Server (in AWS EC2) ]
                                │
                                ▼ Server makes HTTP request to local link-local metadata IP
[ AWS Instance Metadata Service (169.254.169.254) ]
                                │
                                ▼ Returns temporary AWS Secret Access Key & Session Token!
[ Vulnerable Web Server ]
                                │
                                ▼ Server reflects metadata back to Attacker!
[ Attacker Steals Cloud Administrative Credentials -> Full Cloud Compromise! ]
```

### Other SSRF Targets:
1. **Loopback Services:** `http://127.0.0.1:6379/` (Accessing unauthenticated Redis instances in memory to achieve RCE).
2. **Internal Network Pivoting:** `http://192.168.1.100:8080/admin` (Scanning internal corporate servers protected behind firewalls).

### Essential Defenses Against SSRF
1. **Strict Protocol & Domain Allowlisting:**
   Only allow specific protocols (`https://`) and explicitly permitted external hostnames (e.g. `images.trusted-cdn.com`).
2. **IP Resolution & Blacklisting (RFC 1918 & Link-Local):**
   Resolve the destination hostname to an IP address **before issuing the request**. If the resolved IP belongs to:
   - `127.0.0.0/8` (Loopback)
   - `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` (Private RFC 1918)
   - `169.254.0.0/16` (Link-Local / Cloud Metadata)
   $\implies$ **Immediately drop the request!**
3. **Defense Against DNS Rebinding (TOCTOU in SSRF):**
   Ensure the socket connection connects to the *exact verified IP address*, preventing DNS rebinding where an attacker's DNS server returns a public IP during validation but resolves to `127.0.0.1` during connection.
4. **Enforce AWS IMDSv2:**
   Requires a `PUT` request with `X-aws-ec2-metadata-token-ttl-seconds` header to acquire a session token before fetching metadata, defeating simple SSRF exploits.

---

## 5. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"CSRF and SSRF represent client-side versus server-side request forgery. In CSRF, an attacker exploits the browser's automatic inclusion of session cookies to trick a logged-in user's browser into executing unauthorized state-changing actions against a target web app. We mitigate CSRF using unique server-validated Anti-CSRF Synchronizer Tokens and setting `SameSite=Strict` or `Lax` on session cookies. In contrast, SSRF tricks the backend web server into making unauthorized requests to internal resources, private subnets, or cloud metadata endpoints (`169.254.169.254`) that are inaccessible to the public. We defend against SSRF by allowlisting allowed domains, resolving DNS and dropping RFC 1918 and link-local private IP addresses prior to connecting, and mandating AWS IMDSv2 session tokens."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing CSRF with XSS.  
  *Correction:* In **XSS**, the attacker injects and runs *malicious JavaScript inside the vulnerable site*. In **CSRF**, the attacker *does not execute code on the target site*; they merely forge an HTTP request from an external site, relying on the browser to attach valid cookies.
- **Trap:** Believing CORS prevents CSRF.  
  *Correction:* The Same-Origin Policy (SOP) and CORS prevent a cross-origin site from **reading the response** of an HTTP request. But CORS does **NOT prevent the browser from sending the request and modifying backend state**! Anti-CSRF tokens are still mandatory.
