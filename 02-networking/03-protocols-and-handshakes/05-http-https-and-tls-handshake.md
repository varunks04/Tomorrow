# HTTP, HTTPS, Cookies & The TLS 1.2 / 1.3 Handshake

> **Domain:** Networking Fundamentals & Web Security  
> **Sub-Domain:** Application Layer Protocols & Cryptographic Handshakes  
> **Interview Importance:** Critical / Universal Cybersecurity Interview Focus  

---

## 1. Topic & Definitions

- **Hypertext Transfer Protocol (HTTP):** The stateless, application-layer protocol powering the World Wide Web, operating in cleartext on **Port 80**.
- **HTTPS (HTTP Secure):** HTTP layered directly over a cryptographic transport protocol (**TLS - Transport Layer Security**) running on **Port 443**, providing Confidentiality, Data Integrity, and Server Authentication.
- **TLS Handshake:** The initial cryptographic negotiation between client and server that authenticates the server's identity (via X.509 certificates), agrees on cipher suites, and establishes shared symmetric session keys using Ephemeral Diffie-Hellman.
- **SSL vs. TLS:** SSL (Secure Sockets Layer) is the obsolete predecessor to TLS. SSL 2.0 and 3.0 are cryptographically broken (vulnerable to POODLE, BEAST attacks) and forbidden in modern infrastructure. The current industry standards are **TLS 1.2** and **TLS 1.3**.

---

## 2. HTTP Request-Response Mechanics & Status Codes

```text
HTTP Request Structure:
GET /api/v1/user HTTP/1.1                 <-- Request Line (Method, Path, Protocol)
Host: example.com                        <-- Mandatory Header in HTTP/1.1
Authorization: Bearer eyJhbGci...        <-- Auth Token Header
User-Agent: Mozilla/5.0...               <-- Client Metadata
Accept: application/json
[CRLF Blank Line]
{ ...optional JSON payload... }          <-- Request Body (in POST/PUT/PATCH)

HTTP Response Structure:
HTTP/1.1 200 OK                          <-- Status Line (Protocol, Status Code, Reason)
Content-Type: application/json           <-- Response Headers
Set-Cookie: session_id=xyz; Secure; HttpOnly; SameSite=Strict
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
[CRLF Blank Line]
{ "id": 101, "username": "alice" }       <-- Response Body
```

### Essential HTTP Status Codes

| Category | Code & Name | Meaning & Cybersecurity Context |
| :---: | :--- | :--- |
| **2xx** | **200 OK** | Request succeeded. |
| **2xx** | **201 Created** | Resource successfully created (common response to `POST`). |
| **2xx** | **204 No Content** | Request succeeded; no response body returned (common in `DELETE`). |
| **3xx** | **301 Moved Permanently** | Resource relocated permanently. Browsers cache this redirect aggressively. |
| **3xx** | **302 Found / 307 Temp**| Temporary redirect. Used in OAuth/SAML authentication flows. |
| **3xx** | **304 Not Modified** | Client cached version is still fresh (Conditional ETag/If-Modified-Since). |
| **4xx** | **400 Bad Request** | Malformed client request or syntax error. |
| **4xx** | **401 Unauthorized** | **Missing Authentication:** Client is unauthenticated; needs valid credentials. |
| **4xx** | **403 Forbidden** | **Authenticated but Lacking Authorization:** Server knows identity, denies access. |
| **4xx** | **404 Not Found** | Requested URI path does not exist. |
| **4xx** | **405 Method Not Allowed** | HTTP method (e.g. `PUT`) disabled on target endpoint. |
| **4xx** | **429 Too Many Requests**| Rate-limiting threshold triggered. Defense against brute force/credential stuffing. |
| **5xx** | **500 Internal Error** | Unhandled server exception. May leak stack traces if misconfigured. |
| **5xx** | **502 Bad Gateway** | Reverse proxy (Nginx/Cloudflare) received invalid response from backend server. |
| **5xx** | **503 Unavailable** | Backend server is overloaded, crashed, or down for maintenance (DDoS symptom). |
| **5xx** | **504 Gateway Timeout** | Reverse proxy timed out waiting for backend database or upstream service. |

---

## 3. Cookie Security Attributes (Preventing Session Hijacking & XSS)

When a server issues a session identifier in the `Set-Cookie` header, it must specify three security flags:

```text
Set-Cookie: session_id=xyz123; Secure; HttpOnly; SameSite=Strict; Path=/; Max-Age=3600
```

1. **`HttpOnly`:** Instructs the browser that the cookie **cannot be accessed by client-side JavaScript** (`document.cookie`).  
   - *Defense:* Completely defeats token theft via **Cross-Site Scripting (XSS)**.
2. **`Secure`:** Instructs the browser to transmit the cookie **exclusively over encrypted HTTPS connections**.  
   - *Defense:* Prevents cookie interception over cleartext HTTP on unencrypted public Wi-Fi.
3. **`SameSite`:** Controls whether cookies are sent with cross-site requests, providing defense against **Cross-Site Request Forgery (CSRF)**:
   - **`SameSite=Strict`:** Cookie is never sent with third-party cross-site requests (e.g. clicking an external link).
   - **`SameSite=Lax` (Modern default):** Cookie is sent on safe top-level navigations (GET requests from external links), but blocked on cross-site POST requests.
   - **`SameSite=None`:** Cookie sent on all cross-site requests (requires `Secure` attribute).

---

## 4. The TLS 1.3 Handshake (Modern Standard)

TLS 1.3 (RFC 8446) streamlined the handshake down to a single Round Trip Time (**1-RTT**), eliminated insecure legacy ciphers (RSA key exchange, CBC ciphers, SHA-1), and mandated **Ephemeral Diffie-Hellman** for Perfect Forward Secrecy.

```text
Client Browser                                                          Server
      │                                                                   │
      ├─────── 1. ClientHello ───────────────────────────────────────────►│
      │        • Supported TLS versions (1.3)                             │
      │        • Supported Cipher Suites (e.g. TLS_AES_256_GCM_SHA384)    │
      │        • Client Random value                                      │
      │        • Key Share (Client's public Ephemeral Diffie-Hellman key) │
      │        • SNI (Server Name Indication: example.com)                │
      │                                                                   │
      │◄────── 2. ServerHello ────────────────────────────────────────────┤
      │        • Selected Cipher Suite                                    │
      │        • Server Random value                                      │
      │        • Server Key Share (Server's public ECDH key)              │
      │        ════════════ (BOTH DERIVE SYMMETRIC MASTER KEY HERE) ══════│
      │        • { EncryptedExtensions }                                  │
      │        • { Certificate: Server's X.509 Certificate Chain }        │
      │        • { CertificateVerify: Server's Digital Signature }        │
      │        • { Finished: HMAC of handshake transcript }               │
      │                                                                   │
      ├─────── 3. { Finished } ──────────────────────────────────────────►│
      │        (Client confirms symmetric key derivation)                 │
      │                                                                   │
      ▼                                                                   ▼
      ════════════ ENCRYPTED APPLICATION DATA FLOW (HTTPS) ════════════════
```

### Why Did TLS 1.3 Remove RSA Key Exchange?
- In TLS 1.2, servers could use RSA to encrypt the pre-master secret with the server's public key.
- **The Fatal Flaw:** If an intelligence agency or attacker records years of encrypted traffic and steals the server's private key 5 years later, they can retroactively decrypt all historical conversations.
- **TLS 1.3 Mandate:** Mandates **ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)**, guaranteeing **Perfect Forward Secrecy (PFS)**. Each session generates disposable keys that are immediately destroyed upon connection termination.

---

## 5. PKI, Certificate Authorities & SSL Stripping Mitigation

### How the Browser Validates an X.509 Certificate:
1. **Chain of Trust:** The server sends its certificate signed by an **Intermediate CA**, which is signed by a **Root CA**. The client browser has the Root CA public key pre-installed in its trusted operating system Certificate Store.
2. **Domain Match:** Validates that the domain name matches the **SAN (Subject Alternative Name)** field.
3. **Date Validity:** Checks `Not Before` and `Not After` timestamps.
4. **Revocation Check:** Verifies the certificate has not been revoked before expiration via **CRL (Certificate Revocation List)** or **OCSP (Online Certificate Status Protocol) Stapling**.

### Preventing SSL Stripping: HSTS (HTTP Strict Transport Security)
- **SSL Stripping Attack:** An attacker on a public Wi-Fi intercepts an initial unencrypted HTTP redirect (`http://bank.com` $\to$ `https://bank.com`), presenting HTTP to the victim while talking HTTPS to the bank.
- **The Defense: HSTS Header:**
  ```http
  Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
  ```
  Instructs the browser to **never send cleartext HTTP requests to this domain** for the next year, forcing immediate internal rewrite to `https://`. Domains in the **HSTS Preload List** are hardcoded into Chromium and Firefox binaries out of the factory.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"HTTP is a stateless application protocol running over cleartext port 80, whereas HTTPS encrypts traffic over port 443 using TLS. To protect session state, cookies must include `HttpOnly` to defeat XSS script theft, `Secure` to enforce transmission over HTTPS, and `SameSite=Strict/Lax` to prevent Cross-Site Request Forgery. Modern secure web traffic relies on TLS 1.3, which completes in a single round-trip (1-RTT) by combining cipher negotiation with Ephemeral Diffie-Hellman key shares, establishing Perfect Forward Secrecy while eliminating vulnerable static RSA ciphers. Identity authenticity is verified through X.509 PKI certificate chains, and downgrade attacks like SSL stripping are prevented by enforcing HSTS."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing 401 Unauthorized with 403 Forbidden.  
  *Correction:* **401 Unauthorized** means *unauthenticated* (the server does not know who you are; supply credentials). **403 Forbidden** means *unauthorized* (the server knows your identity, but your role lacks permission to view the resource).
- **Trap:** Claiming TLS 1.3 uses RSA for key exchange.  
  *Correction:* TLS 1.3 completely banned RSA key exchange. RSA is now allowed **only for digital signatures in certificates**, never for session key exchange. Key exchange is strictly Ephemeral Diffie-Hellman.
