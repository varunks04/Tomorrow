# Cybersecurity Fundamentals — Technical Interview Q&A

### Q1: What is Perfect Forward Secrecy (PFS), and how is it achieved?
> **Model Answer:**
> Perfect Forward Secrecy ensures that if a server's long-term private key is compromised in the future, past recorded encrypted communications **cannot be decrypted**.
> It is achieved using **Ephemeral Diffie-Hellman (DHE or ECDHE)** key exchanges, where temporary, disposable key pairs are negotiated uniquely for each session and discarded immediately after session key derivation.

---

### Q2: Compare Reflected XSS, Stored XSS, and DOM-based XSS.
> **Model Answer:**
> - **Stored XSS (Persistent):** Malicious script is saved in the database (e.g. comment section) and executes in every user's browser who views that page.
> - **Reflected XSS (Non-Persistent):** Malicious script is injected into a request (e.g. search query parameter) and reflected back immediately in the server's response.
> - **DOM-based XSS:** Vulnerability exists entirely on the client side; client JavaScript reads untrusted data from a source (`location.search`) and passes it into an unsafe sink (`eval()`, `innerHTML`) without server involvement.
> *Defense:* Context-aware output encoding, strict Content Security Policy (CSP), and avoiding unsafe sinks.

---

### Q3: What are the 6 stages of the NIST Incident Response Framework?
> **Model Answer (NIST SP 800-61 Rev 2):**
> 1. **Preparation:** Policies, playbooks, tools, training, and log readiness.
> 2. **Detection & Analysis:** Alert triage, log analysis, validating true vs false positive.
> 3. **Containment:** Short-term isolation (disconnect host from network) and long-term containment.
> 4. **Eradication:** Removing malware, deleting attacker accounts, patching vulnerabilities.
> 5. **Recovery:** Restoring systems from clean backups, verifying normal operation.
> 6. **Post-Incident Activity (Lessons Learned):** Debriefing, documentation, playbook updates.
