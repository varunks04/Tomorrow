# User Authentication: The Login-to-Authorization Lifecycle

## 1. Topic & Definition
The **Login-to-Authorization Lifecycle** defines the complete sequence of computational and cryptographic transformations that occur from the moment a human user types their credentials into a login portal to the moment an authorized API request executes a privileged backend database query.

This end-to-end workflow illustrates how defense-in-depth integrates **Brute-Force Rate Limiting**, **Argon2id Password Hashing**, **Phishing-Resistant MFA Verification**, **Secure Session/Token Issuance**, and **Middleware-Enforced Role-Based Access Control (RBAC)**.

---

## 2. End-to-End Architectural Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor User as User (Browser)
    participant Edge as Edge / WAF / Rate Limiter (Redis)
    participant AuthAPI as Authentication Service
    participant DB as Identity Database (PostgreSQL)
    participant MFA as MFA / WebAuthn Engine
    participant SessionStore as Distributed Cache (Redis)
    participant BusinessAPI as Protected Business API

    User->>Edge: 1. POST /api/v1/auth/login (Username, Password, CSRF-Token) over TLS
    Edge->>Edge: Checks Sliding Window Rate Limit: (5 attempts / 60s per IP+User) -> PASSED
    Edge->>AuthAPI: Forwards sanitized payload
    AuthAPI->>DB: 2. SELECT id, password_hash, mfa_secret, role FROM users WHERE email = $1
    DB-->>AuthAPI: Returns user record
    Note over AuthAPI: 3. Computes Argon2id Verification:<br/>Argon2id(Submitted_Password, Stored_Salt, Memory=64MB, Iterations=3) == Stored_Hash? -> MATCH!
    AuthAPI-->>User: 4. Returns 403 MFA Challenge Required (mfa_token: temp_jwt)
    User->>MFA: 5. Submits FIDO2 WebAuthn Signed Assertion / TOTP (Code: 839201)
    MFA->>MFA: Validates TOTP time-step window / FIDO2 public key signature -> VALID!
    AuthAPI->>SessionStore: 6. Creates Session Object: SET session:7f8a9b... {user_id: 101, role: "auditor", exp: 3600}
    AuthAPI-->>User: 7. 200 OK + Set-Cookie: sid=7f8a9b...; Secure; HttpOnly; SameSite=Strict; Path=/
    
    Note over User,BusinessAPI: SUBSEQUENT PRIVILEGED REQUEST EXECUTION
    User->>BusinessAPI: 8. GET /api/v1/financial-reports/q3 (Cookie: sid=7f8a9b...)
    BusinessAPI->>SessionStore: 9. GET session:7f8a9b...
    SessionStore-->>BusinessAPI: Returns {user_id: 101, role: "auditor", active: true}
    Note over BusinessAPI: 10. RBAC Middleware Enforcement:<br/>Does role "auditor" have permission "read:financial-reports"?<br/>Policy: ALLOW.
    BusinessAPI-->>User: 11. Returns 200 OK (Financial Data JSON Payload)
```

---

## 3. Step-by-Step Technical Execution Breakdown

### Phase 1: Edge Validation & Abuse Prevention
1. **TLS Enforcement:** Ingress traffic arrives strictly over TLS 1.3 (`Strict-Transport-Security` header enforced).
2. **Rate Limiting:** A distributed Redis cache evaluates a sliding-window counter. If more than 5 failed attempts occur within 60 seconds from the same IP or targeting the same username, the request is throttled (`429 Too Many Requests`), neutralizing automated credential stuffing.
3. **Anti-CSRF Verification:** For cookie-based applications, the server validates a cryptographic CSRF token (Double Submit Cookie or Synchronizer Token Pattern).

### Phase 2: Credential Verification & Cryptographic Math
1. **Parameterized Database Retrieval:** The service executes a parameterized SQL query:
   `SELECT id, password_hash, salt, mfa_enabled FROM users WHERE username = $1`
   *(Parameterized queries prevent SQL Injection).*
2. **Argon2id Hash Verification:**
   - The server extracts the embedded salt and cryptographic parameters from the stored hash string:
     `$argon2id$v=19$m=65536,t=3,p=4$c29tZXNhbHQ...$R7Z...`
   - It hashes the submitted password using **Argon2id** (configured with memory cost $m=64\text{ MB}$, time cost $t=3$ iterations, and parallelism $p=4$).
   - The comparison is executed via **constant-time byte comparison** (`crypto.timingSafeEqual`) to eliminate timing-attack side channels.

### Phase 3: Secondary Factor Authentication (MFA)
1. If the password is correct and MFA is enabled, the server issues a short-lived (5-minute) single-purpose `mfa_ticket`.
2. The user fulfills the challenge:
   - **TOTP:** Server calculates $HMAC-SHA1(K, \text{floor}(\text{timestamp}/30))$ across time windows $[T-1, T, T+1]$.
   - **FIDO2 / WebAuthn:** Server validates that the authenticator signed the cryptographic challenge using the user's registered public key, bound to the browser origin.

### Phase 4: Session Issuance & Cookie Hardening
Upon complete authentication, the server establishes state:
- Generates a **128-bit cryptographically secure pseudorandom token** (e.g., using `crypto.randomBytes(32)`).
- Stores the session in Redis with an explicit Time-To-Live (TTL: e.g., 3600 seconds).
- Transmits the session ID via the `Set-Cookie` HTTP response header with all four security flags:
  `Set-Cookie: sid=abc123...; Secure; HttpOnly; SameSite=Strict; Path=/`

### Phase 5: Middleware Authorization (RBAC / ABAC)
When the user accesses a protected route:
1. **Authentication Filter:** Middleware extracts the session ID from the cookie, queries Redis, validates active status, and binds the user object to the request context.
2. **RBAC Policy Check:** The authorization engine evaluates the permission matrix:
   ```python
   def require_permission(permission):
       def decorator(f):
           def wrapper(request, *args, **kwargs):
               user_roles = request.user.roles
               if not rbac_engine.has_permission(user_roles, permission):
                   return Response(status=403, body={"error": "Forbidden: Insufficient privileges"})
               return f(request, *args, **kwargs)
           return wrapper
       return decorator
   ```
3. If the role permits the action, the backend database query executes.

---

## 4. Key Differences Matrix: Session ID vs JWT Token Lifecycle

| Feature | Stateful Session ID Architecture | Stateless JWT Architecture |
| :--- | :--- | :--- |
| **Storage Location** | Server-side distributed cache (Redis) | Client-side memory / HttpOnly cookie |
| **Verification Cost**| O(1) Redis memory network lookup | O(1) Local cryptographic mathematical verification |
| **Revocation Speed** | **Instantaneous** (Delete key from Redis immediately kills session)| Difficult (Requires token blacklists or waiting for short TTL)|
| **Horizontal Scale** | Requires centralized Redis cluster | High (APIs verify tokens independently without shared cache)|
| **Payload Size** | Minimal (~32 byte UUID) | Moderate to Large (Hundreds of bytes of signed claims) |
| **Enterprise Standard**| Monoliths & traditional web apps | Distributed microservices & cross-domain APIs |

---

## 5. Cybersecurity Relevance, Threats & Attack Vectors

### 1. Timing Attacks on Password Verification
- **Vulnerability:** Comparing strings using standard equality operators (`if password_hash == input_hash:`). Standard string comparisons terminate at the first non-matching byte.
- **Exploitation:** An adversary measures microsecond variations in response times to deduce character matches.
- **Defense:** Enforce constant-time string comparison routines (`timingSafeEqual()`).

### 2. Privilege Escalation via Mass Assignment
- **Vulnerability:** An API accepts arbitrary JSON payloads during profile updates (`PUT /api/user/profile`) and maps them directly to the database user model without field whitelisting.
- **Exploitation:** The user submits: `{"name": "Alice", "role": "SuperAdmin"}`. The backend updates the `role` column, granting instant root privileges!
- **Defense:** Use strict Data Transfer Objects (DTOs) with parameter whitelisting (Pydantic / class-validator).

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"The complete authentication-to-authorization lifecycle transforms untrusted credentials into authorized resource access through layered defensive controls. At the edge, sliding-window rate limiters mitigate automated credential stuffing. The backend verifies credentials using memory-hard Argon2id hashing evaluated with constant-time comparisons to defeat timing side-channels, followed by mandatory MFA validation. Once verified, the server issues a high-entropy session identifier secured by `HttpOnly`, `Secure`, and `SameSite=Strict` cookie flags. On every subsequent API request, authorization middleware extracts the session, validates user identity in a distributed Redis cache, and evaluates declarative Role-Based Access Control (RBAC) permissions before allowing backend database execution."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Storing passwords using SHA-256 or MD5. *Correction:* General-purpose hash functions are designed to be fast, allowing GPUs to calculate billions of guesses per second; passwords must use memory-hard, slow algorithms like Argon2id, bcrypt, or scrypt.
- **Trap 2:** Confusing 401 Unauthorized with 403 Forbidden. *Correction:* 401 means "Unauthenticated" (the server does not know who you are); 403 means "Forbidden" (the server knows who you are, but your role lacks permission).
- **Trap 3:** Failing to invalidate sessions server-side on logout. *Correction:* Deleting a cookie in the client browser does not terminate the session; the server must explicitly delete the session key from Redis to prevent session token reuse.

### Expected Follow-Up Questions
1. *What is Constant-Time Comparison and why is it necessary in authentication?*
   - Constant-time comparison ensures that string comparisons execute in the exact same number of CPU clock cycles regardless of how many characters match, preventing adversaries from analyzing microscopic response time differences to crack tokens or hashes.
2. *Why is Argon2id preferred over standard bcrypt?*
   - Argon2id is a hybrid algorithm resistant to both side-channel attacks (Argon2i) and GPU/ASIC memory-parallel cracking (Argon2d), allowing administrators to configure dedicated memory, time, and parallelism costs.
