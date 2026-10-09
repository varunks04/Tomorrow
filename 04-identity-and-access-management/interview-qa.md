# IAM & Identity Protocols — Technical Interview Q&A

### Q1: Explain the difference between OAuth 2.0 and OpenID Connect (OIDC).
> **Model Answer:**
> - **OAuth 2.0** is an **Authorization Framework**. It allows a third-party application to obtain limited access to a user's resources on an API without knowing the user's password. It issues an **Access Token**.
> - **OpenID Connect (OIDC)** is an **Authentication Layer** built directly on top of OAuth 2.0. It enables clients to verify the identity of the end user. It introduces the **ID Token** (a signed JWT containing identity claims like `sub`, `email`, `aud`).
> *Interview Rule:* Use OAuth when delegating access to APIs; use OIDC when logging users in.

---

### Q2: How does Kerberos authentication work? Explain TGT and Service Tickets.
> **Model Answer:**
> Kerberos relies on a trusted Key Distribution Center (KDC) consisting of the Authentication Service (AS) and Ticket Granting Service (TGS):
> 1. **AS-REQ / AS-REP:** Client sends username; AS validates and returns a **Ticket Granting Ticket (TGT)** encrypted with the KDC secret key (`krbtgt`), along with a session key encrypted with the client's password hash.
> 2. **TGS-REQ / TGS-REP:** Client presents TGT to TGS requesting access to a specific service (e.g. file share). TGS validates TGT and returns a **Service Ticket** encrypted with the service account's secret key.
> 3. **AP-REQ / AP-REP:** Client sends Service Ticket to target server to gain authorized access.

---

### Q3: What is Kerberoasting and how do you protect against it?
> **Model Answer:**
> Any domain user can legitimately request a Kerberos Service Ticket (TGS) for any service that has a registered Service Principal Name (SPN).
> Because the Service Ticket is encrypted with the hash of the service account's password, an attacker can extract the ticket from memory, take it offline, and perform brute-force dictionary attacks to recover the plaintext password.  
> *Mitigation:* Use Group Managed Service Accounts (gMSA) with 128-character automatically rotated passwords, AES encryption over RC4, and alert on anomalous TGS request volumes.
