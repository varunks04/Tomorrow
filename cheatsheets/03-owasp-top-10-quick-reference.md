# OWASP Top 10 Web Application Security — Quick Reference

> **Cybersecurity Interview Focus:** Know the exact risk name, root vulnerability cause, realistic attack vector payload, and industry-standard defensive mitigations.

---

## The OWASP Top 10 Matrix

| Category ID & Risk Name | Root Cause / Vulnerability | Real-World Attack Payload / Scenario | Golden Defense / Remediation |
| :--- | :--- | :--- | :--- |
| **A01: Broken Access Control** | Failure to enforce authorization checks on requested objects or functions. | Changing URL parameter from `GET /api/user/101/invoice` to `/api/user/102/invoice` (IDOR). | Enforce server-side session-based ownership verification; adopt RBAC/ABAC models. |
| **A02: Cryptographic Failures** | Sensitive data transmitted in plaintext, weak algorithms (MD5, DES), hardcoded keys. | Intercepting HTTP traffic on public Wi-Fi; harvesting credit cards stored without AES-256-GCM. | Enforce TLS 1.3, HSTS header, encrypt data at rest, store secrets in Key Vaults (AWS KMS/HashiCorp). |
| **A03: Injection** | Untrusted user data interpreted as executable code/commands by an interpreter. | `' OR 1=1; DROP TABLE users; --` or `cat /etc/passwd` via unsanitized `os.system()` input. | Parameterized queries (Prepared Statements), ORMs, strict allowlist input validation. |
| **A04: Insecure Design** | Architectural flaws where business logic lacks threat modeling or rate limiting. | Attacker abuses password reset flow lacking rate limits to brute force 6-digit OTP codes. | Threat modeling (STRIDE), secure-by-design patterns, rate-limiting, defense-in-depth. |
| **A05: Security Misconfiguration** | Default credentials, verbose error stack traces, unhardened cloud permissions, open S3 buckets. | Accessing `http://target/admin` with `admin:admin`, reading stack traces revealing database credentials. | Hardened base images, automated CIS benchmark audits, disable debug mode in production. |
| **A06: Vulnerable & Outdated Components** | Using third-party open-source libraries or frameworks with known CVEs. | Log4Shell (`CVE-2021-44228`), Apache Struts (`CVE-2017-5638`), vulnerable npm/PyPI dependencies. | Software Composition Analysis (SCA) in CI/CD (Snyk, Dependabot), SBOM tracking, continuous patching. |
| **A07: Identification & Auth Failures** | Weak password policies, lack of MFA, credential stuffing, broken session timeouts. | Attacker replays stolen list of credentials from a breach to take over user accounts. | Mandatory MFA, account lockout/rate limiting, robust password complexity, secure session invalidation. |
| **A08: Software & Data Integrity Failures** | Unsigned updates, untrusted deserialization, insecure CI/CD build pipelines. | SolarWinds build pipeline poisoning; deserializing untrusted Python `pickle` or Java payloads. | Cryptographically sign all artifacts (Sigstore/Cosign), avoid native deserialization, verify checksums. |
| **A09: Security Logging & Monitoring Failures** | Actions not logged, logs not centralized in SIEM, breach undetected for months. | Attacker spends 180 days executing reconnaissance and lateral movement without triggering an alert. | Centralized logging into SIEM, real-time alert correlation, audit logs for authentication/access. |
| **A10: Server-Side Request Forgery (SSRF)** | Server fetches a remote URL provided by user without validating destination IP. | `GET /preview?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/` | Restrict outgoing connections, block internal IP ranges (RFC 1918, link-local), enforce IMDSv2. |

---

## 🎙️ Interview Hot Take: SQL Injection Defense
```text
Interviewer: "How do you prevent SQL Injection?"

❌ Weak Answer: "I filter out quotes and sanitize inputs."
✅ Golden Answer: "The primary and definitive defense is Parameterized Queries (Prepared Statements).
By separating the SQL query structure from user data, the database engine treats user input strictly
as literal values, never as executable SQL code. Secondary defenses include ORM frameworks, stored
procedures without dynamic concatenation, and database least-privilege accounts."
```
