# Single Sign-On (SSO) and Federated Identity Architecture

## 1. Topic & Definition
**Single Sign-On (SSO)** is an access control architecture that enables a user to authenticate once with a trusted central credential authority and gain seamless, authorized access to multiple independent software systems and applications without being prompted to re-enter credentials.

**Identity Federation** expands SSO beyond a single organizational perimeter, establishing cryptographic trust between distinct administrative domains (e.g., an enterprise using Microsoft Entra ID logging into third-party SaaS applications like Salesforce, AWS, or ServiceNow).

### Core Architectural Entities
1. **Principal (Subject):** The human end-user requesting access via a user-agent (browser or native app).
2. **Identity Provider (IdP):** The centralized trusted authority that stores user identities, verifies primary credentials, evaluates MFA and conditional access policies, and issues cryptographically signed assertions or tokens (e.g., Okta, Ping Identity, Microsoft Entra ID).
3. **Service Provider (SP) / Relying Party (RP):** The target application or resource server providing business capabilities (e.g., GitHub, Slack, Workday). The SP does not maintain user passwords; it trusts assertions issued by the IdP.
4. **Trust Relationship (Federation Metadata):** Established out-of-band via exchange of X.509 public certificates, cryptographic keys, endpoint URLs, and entity IDs before authentication transactions occur.

---

## 2. How It Works: Federation Trust Models and Flows

### SSO Architecture & Trust Boundary

```
+-----------------------------------------------------------------------------------------+
|                               Federated SSO Ecosystem                                   |
+-----------------------------------------------------------------------------------------+

             [ User-Agent (Browser) ]
              /                    \
    1. Auth Request            4. Present Token
            /                        \
           v                          v
  +--------------------+     Trust Metadata Exchange     +--------------------+
  | Identity Provider  |<===============================>|  Service Provider  |
  |      (IdP)         |    (Public Keys & Entity IDs)   |     (SP / RP)      |
  | [Central User DB]  |                                 | [Business Logic]   |
  +--------------------+                                 +--------------------+
            |                                                      |
    2. Enforces MFA &                                     5. Validates Assertion
       Conditional Access                                    Signature & Grants
    3. Issues Signed Assertion                               Access
```

### Trust Topologies
1. **Hub-and-Spoke / Identity Broker:**
   - Multiple disparate SPs connect to a central identity broker. The broker normalizes protocols (e.g., translates internal LDAP or Kerberos into SAML 2.0 or OIDC for cloud apps).
2. **Mesh Federation:**
   - Multiple independent organizations establish pairwise direct trust relationships (common in academic networks like Shibboleth / eduroam).
3. **Social Federation:**
   - Consumer applications delegate authentication to dominant public consumer IdPs (Google, Apple, GitHub) using OIDC.

---

## 3. Practical Metadata Exchange and Configuration

### Federation Configuration Elements
To establish a federation link between an IdP and an SP, administrators exchange metadata:
- **Entity ID (Issuer URI):** Globally unique identifier for both IdP and SP (e.g., `https://identity.corp.com/entity`).
- **Assertion Consumer Service (ACS) URL:** The endpoint on the SP where the IdP posts signed tokens (e.g., `https://sp.app.com/saml/acs`).
- **Single Logout (SLO) URL:** Endpoint responsible for coordinated termination of all active sessions across all federated applications.
- **Signing / Encryption Certificates:** Public X.509 certificates used to verify signatures and decrypt assertion attributes.

### Typical Claims / Attribute Mapping
```json
{
  "nameidentifier": "alice.smith@enterprise.com",
  "givenname": "Alice",
  "surname": "Smith",
  "emailaddress": "alice.smith@enterprise.com",
  "department": "Security Operations",
  "groups": ["SecOps-L2", "Incident-Responders"]
}
```

---

## 4. Key Differences Matrix: Federation Protocols

| Dimension | SAML 2.0 | OAuth 2.0 | OpenID Connect (OIDC) |
| :--- | :--- | :--- | :--- |
| **Primary Purpose** | **Authentication & Enterprise SSO** | **Authorization / API Delegation** | **Authentication & User Profile SSO** |
| **Data Payload Format** | Heavyweight XML Assertions | JSON (JWT format common) | JSON (JWT ID Tokens) |
| **Transport Medium** | HTTP POST binding / Redirects | HTTP Headers (`Authorization: Bearer`) | HTTP Headers / Redirects |
| **Primary Use Case** | Legacy Enterprise & SaaS Apps | REST APIs & Microservices | Modern Cloud, Web & Mobile Apps |
| **Tokens Issued** | SAML Assertion (AuthN & Attributes) | Access Token & Refresh Token | ID Token (Identity) + Access Token |
| **Mobile App Friendliness**| Low (Heavy XML parsing in apps) | High | High |
| **Standardization Body** | OASIS | IETF (RFC 6749) | OpenID Foundation |

---

## 5. Cybersecurity Relevance, Threats & Attack Vectors

### 1. Identity Provider as a Single Point of Failure (SPOF) & Blast Radius
- **Risk:** In an SSO ecosystem, if an attacker compromises the IdP (or an admin account on Okta/Entra ID), they gain immediate, unfettered access to **every connected downstream application** across the entire enterprise.
- **Real-World Incident:** The 2022 Lapsus$ breach of Okta sub-contractors demonstrated how compromising support tools at the IdP level creates existential enterprise risk.
- **Defense:** Enforce hardware-backed FIDO2 Passkeys for all IdP administrators, break-glass accounts with alerting, and out-of-band monitoring for IdP configuration modifications.

### 2. Incomplete Single Logout (SLO) Vulnerabilities
- **Mechanism:** When a user clicks "Logout" in a single SP application, the SP destroys its local session, but the user's session at the IdP remains active.
- **Exploitation:** On a shared workstation, a subsequent user navigates to another federated application; the browser transparently re-authenticates via the lingering IdP session without prompting for credentials.
- **Defense:** Implement Front-Channel or Back-Channel SLO protocols where the IdP notifies and invalidates sessions across all registered SPs simultaneously.

### 3. Assertion Replay Attacks
- **Mechanism:** Attacker intercepts a signed SSO assertion in transit and replays it to the SP to establish an authenticated session.
- **Defense:** SPs must enforce strict validation of temporal validity (`NotBefore`, `NotOnOrAfter`) and track unique assertion IDs (`jti` or SAML `ID`) to ensure one-time consumption.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Single Sign-On (SSO) and identity federation eliminate credential sprawl by centralizing authentication at a trusted Identity Provider (IdP) like Okta or Entra ID, decoupling authentication from individual Service Providers (SPs). Trust is established out-of-band via public key certificate exchanges and entity identifiers. While SAML 2.0 has historically dominated enterprise legacy SaaS federation using XML assertions, modern cloud-native, web, and mobile ecosystems have standardized on OpenID Connect (OIDC) built on top of OAuth 2.0 due to its lightweight JSON/JWT architecture. Centralizing auth vastly improves policy enforcement but turns the IdP into a high-value target that requires strict FIDO2 MFA and continuous anomaly detection."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Equating OAuth 2.0 with SSO. *Correction:* OAuth 2.0 is an authorization framework for delegating API permissions, not an authentication protocol. SSO is achieved via OIDC or SAML.
- **Trap 2:** Assuming logout from an application logs the user out of the IdP. *Correction:* Without coordinated Single Logout (SLO), logging out of an SP leaves the central IdP session fully active.

### Expected Follow-Up Questions
1. *What is the difference between SP-Initiated and IdP-Initiated SSO?*
   - SP-initiated begins when the user visits the target application directly (e.g., `app.slack.com`) and gets redirected to the IdP with an auth request. IdP-initiated starts at the enterprise portal (e.g., Okta dashboard) where clicking the app tile pushes a signed assertion directly to the SP's ACS endpoint.
2. *Why is IdP-Initiated SAML considered a security risk?*
   - IdP-initiated flows do not generate a cryptographic request state (e.g., SAML `InResponseTo` or OAuth `state`), making the endpoint vulnerable to unsolicited Assertion Injection and Cross-Site Request Forgery (CSRF).
