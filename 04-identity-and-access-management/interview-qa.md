# Identity & Access Management (IAM) — Technical Interview Q&A Compendium

## Core Concepts & Protocol Architecture

### Q1: What is the core difference between Authentication (AuthN) and Authorization (AuthZ)?
> **Model Answer:**
> - **Authentication (AuthN)** verifies *who an entity is* (Identity verification). It answers "Are you truly Alice?" using credentials, biometrics, or cryptographic proofs. Standard protocols: OpenID Connect, SAML 2.0, Kerberos.
> - **Authorization (AuthZ)** determines *what an authenticated entity is permitted to do* (Access permissions). It answers "Does Alice have permission to delete this database?" using access control policies. Standard frameworks: OAuth 2.0, RBAC, ABAC.
> *Golden Rule:* Authentication must always precede Authorization. An entity cannot be granted scoped privileges until its identity is reliably established.

---

### Q2: Compare OAuth 2.0, OpenID Connect (OIDC), and SAML 2.0. When would you choose each?
> **Model Answer:**
> - **OAuth 2.0:** An authorization framework designed for delegated API access using Access Tokens. Used when a client app needs permission to access backend APIs on behalf of a user.
> - **OpenID Connect (OIDC):** An identity authentication layer built directly on top of OAuth 2.0. It introduces the signed JWT **ID Token** and UserInfo endpoint. Used for modern web, mobile, and cloud-native Single Sign-On.
> - **SAML 2.0:** An enterprise XML-based federation standard exchanging signed SAML Assertions. Used primarily for legacy enterprise B2B SaaS integrations (e.g., Salesforce, Workday, legacy portals).

---

### Q3: Explain the OAuth 2.0 Authorization Code Flow with PKCE. Why is PKCE mandatory for public clients?
> **Model Answer:**
> - In public clients (Single Page Apps, native mobile apps), client secrets cannot be stored securely because code runs on the end user's device.
> - **PKCE (Proof Key for Code Exchange — RFC 7636)** prevents authorization code interception attacks:
>   1. The client generates a random secret (`code_verifier`) and hashes it with SHA-256 (`code_challenge`).
>   2. The client sends `code_challenge` in the initial `/authorize` request.
>   3. When the authorization server returns the authorization code, the client exchanges it by presenting the original `code_verifier` in the POST `/token` body.
>   4. The server hashes `code_verifier` and verifies it matches the original challenge before issuing tokens.
> - Even if an attacker intercepts the authorization code on a mobile OS redirect, they cannot exchange it without the unhashed `code_verifier`.

---

### Q4: How does a JSON Web Token (JWT) work, and how do you protect against the `alg: none` and Key Confusion attacks?
> **Model Answer:**
> A JWT consists of three Base64URL-encoded parts: `Header.Payload.Signature`.
> - **`alg: none` Attack:** Attackers tamper with payload claims (e.g., setting `admin: true`), set `"alg": "none"` in the header, and strip the signature. If the server naively trusts the header, it skips signature verification.  
>   *Defense:* Hardcode the verification algorithm in backend code (e.g., whitelist only `RS256`); never allow user-controlled headers to dictate the algorithm.
> - **HMAC vs RSA Key Confusion:** Server expects RS256 using a public key. Attacker alters header to `HS256` (symmetric HMAC) and signs the token using the server's public key as the symmetric secret string. Because the server uses the same public key bytes to verify the HMAC signature, it succeeds!  
>   *Defense:* Enforce strict algorithm whitelisting and ensure public key objects cannot be parsed into HMAC secret validators.

---

### Q5: How does Kerberos authentication work? Detail the 6-step handshake.
> **Model Answer:**
> Kerberos (Port 88) uses a trusted Key Distribution Center (KDC) split into an Authentication Service (AS) and Ticket Granting Service (TGS):
> 1. **AS-REQ:** Client sends timestamp encrypted with user password hash (pre-authentication).
> 2. **AS-REP:** AS validates pre-auth and returns a **Ticket Granting Ticket (TGT)** encrypted with the KDC's `krbtgt` key, plus an encrypted session key.
> 3. **TGS-REQ:** Client sends TGT, an authenticator, and the target Service Principal Name (SPN).
> 4. **TGS-REP:** TGS decrypts TGT, validates authenticator, and returns a **Service Ticket** encrypted with the target service account's password hash.
> 5. **AP-REQ:** Client presents Service Ticket and session authenticator to the target server.
> 6. **AP-REP:** Target server decrypts the ticket locally with its own secret key, validates permissions via the embedded Privilege Attribute Certificate (PAC), and confirms mutual authentication.

---

### Q6: What is Kerberoasting and how do you defend against it?
> **Model Answer:**
> - **Mechanics:** Any authenticated domain user can request a TGS Service Ticket for any registered Service Principal Name (SPN) in Active Directory. Because the Service Ticket is encrypted with the password hash of the service account, the attacker extracts the ticket from memory (LSASS) and cracks it offline using dictionary attacks (`hashcat -m 13100`).
> - **Defenses:**
>   1. Migrate service accounts to **Group Managed Service Accounts (gMSA)** featuring 128-character automatically rotating passwords.
>   2. Enforce AES-256 encryption over legacy RC4.
>   3. Monitor Windows Event ID 4769 for abnormal spikes in TGS ticket requests, especially those requesting RC4 encryption.

---

### Q7: What is the difference between a Golden Ticket and a Silver Ticket?
> **Model Answer:**
> - **Golden Ticket:** Forged **Ticket Granting Ticket (TGT)** using the compromised password hash of the **`krbtgt`** account. Grants arbitrary Domain Admin privileges across the entire forest. Requires contacting the KDC for service tickets. Remediation requires resetting the `krbtgt` password **twice**.
> - **Silver Ticket:** Forged **Service Ticket (TGS)** using the compromised password hash of a specific **Service Account**. Grants full admin access to that single service only. Completely bypasses the KDC (no KDC traffic generated). Remediation requires resetting that specific service account password.

---

### Q8: What is DCSync and what permissions enable it?
> **Model Answer:**
> - DCSync utilizes the Microsoft Directory Replication Service Remote Protocol (MS-DRSR) to impersonate a Domain Controller and request replication data over the network.
> - It allows an attacker to dump all password hashes from `NTDS.DIT`—including the `krbtgt` hash—without running code or executing tools on the Domain Controller.
> - It requires two specific AD replication permissions on the domain root:
>   - `DS-Replication-Get-Changes`
>   - `DS-Replication-Get-Changes-All`
> - Monitored via Windows Event ID 4662 auditing directory replication access.

---

### Q9: Why is FIDO2 / WebAuthn considered "phishing-resistant" compared to TOTP?
> **Model Answer:**
> - Traditional MFA (SMS, TOTP apps) is vulnerable to Adversary-in-the-Middle (AiTM) reverse proxies (e.g., Evilginx) that capture and relay the OTP and resulting session cookies in real time.
> - **FIDO2 / WebAuthn** uses public-key cryptography bound directly to the browser's web origin (Relying Party ID).
> - When visiting `evil-phish-bank.com`, the browser sends `evil-phish-bank.com` as the origin to the hardware authenticator (YubiKey/Secure Enclave). The authenticator looks up keys bound only to that domain. It will **never release or sign credentials for `bank.com`**, completely stopping reverse-proxy phishing.

---

### Q10: How do you design secure token storage in modern Single Page Applications (SPAs)?
> **Model Answer:**
> - **Never store long-lived tokens in `localStorage` or `sessionStorage`:** They are vulnerable to extraction via Cross-Site Scripting (XSS).
> - **Recommended Architecture:**
>   1. Keep the short-lived (5–15 min) **Access Token** purely in **JavaScript in-memory state** (React state / closure).
>   2. Store the **Refresh Token** in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie scoped exclusively to the `/auth/refresh` endpoint.
>   3. Implement **Refresh Token Rotation (RTR) with reuse detection**: every time a refresh token is exchanged, a new one is issued; if an old token is ever replayed, immediately invalidate the entire session family.
