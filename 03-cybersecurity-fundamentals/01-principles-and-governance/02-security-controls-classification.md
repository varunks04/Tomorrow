# Security Controls Classification, Defense in Depth & Governance

> **Domain:** Cybersecurity Fundamentals  
> **Sub-Domain:** Governance, Risk & Architecture  
> **Interview Importance:** Very High / Standard Framework & CISSP Concept  

---

## 1. Topic & Definitions

- **Security Controls:** Safeguards, countermeasures, and technical mechanisms prescribed for an information system to protect the confidentiality, integrity, and availability of system assets and satisfy specific security requirements.
- **Two-Dimensional Classification System:** Every security control is classified across two distinct axes:
  1. **By Implementation Architecture:** *Administrative (Managerial)*, *Technical (Logical)*, or *Physical*.
  2. **By Functional Action:** *Preventive*, *Detective*, *Corrective*, *Deterrent*, *Compensating*, or *Recovery*.
- **Defense in Depth:** An architectural strategy employing layered security controls throughout an IT infrastructure so that if one defensive layer fails or is bypassed, subsequent layers prevent a breach.
- **Separation of Duties (SoD):** A business practice requiring that critical, high-risk tasks be split between multiple individuals, ensuring no single employee has end-to-end unilateral capability to execute fraud or sabotage.

---

## 2. Classification Matrix: Implementation vs. Functional Category

```text
                                       FUNCTIONAL ACTIONS
                   ┌─────────────────┬─────────────────┬─────────────────┐
                   │   PREVENTIVE    │    DETECTIVE    │   CORRECTIVE    │
  ┌────────────────┼─────────────────┼─────────────────┼─────────────────┤
  │ ADMINISTRATIVE │ Background check│ Security audit  │ Incident policy │
I │ (Managerial)   │ Acceptable use  │ Account review  │ Termination     │
M │                │ policy          │                 │ procedures      │
P ├────────────────┼─────────────────┼─────────────────┼─────────────────┤
L │ TECHNICAL      │ Firewall rule   │ SIEM alerting   │ Automated patch │
E │ (Logical)      │ MFA enforce     │ IDS signatures  │ Antivirus       │
M │                │ Parameterized Q │ EDR agent       │ quarantine      │
E ├────────────────┼─────────────────┼─────────────────┼─────────────────┤
N │ PHYSICAL       │ Mantrap door    │ CCTV cameras    │ Fire suppression│
T │                │ Biometric lock  │ Motion sensor   │ Backup generator│
  └────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

---

## 3. Deep Dive: Functional Control Categories

### 1. Preventive Controls (Stops Attacks Before Occurrence)
- **Objective:** Proactively prevent an unauthorized action, intrusion, or error from occurring.
- **Examples:**
  - *Technical:* Firewall blocking inbound port 23; MFA blocking brute-force logins; Data Execution Prevention (DEP) blocking shellcode execution on the stack.
  - *Physical:* Biometric scanner on data center doors; fences.
  - *Administrative:* Separation of duties policy; strict onboarding background checks.

### 2. Detective Controls (Identifies Ongoing or Past Incidents)
- **Objective:** Identify and alert security personnel to violations, policy breaches, or anomalies as they occur.
- **Examples:**
  - *Technical:* SIEM correlation rules; Snort IDS signatures; File Integrity Monitoring (FIM); honeytokens / honeypots.
  - *Physical:* Security guard patrols; CCTV recording cameras.
  - *Administrative:* Quarterly user access rights audits; financial ledger reconciliation.

### 3. Corrective Controls (Repairs Damage and Remediates Systems)
- **Objective:** Mitigate the impact of an active incident, eradicate the threat, and restore normal operational state.
- **Examples:**
  - *Technical:* EDR isolating an infected endpoint; Antivirus moving malware to quarantine; applying a hotfix patch to a compromised web server.
  - *Administrative:* Incident Response Playbooks; disaster recovery execution.

### 4. Deterrent Controls (Discourages Attackers)
- **Objective:** Dissuade potential adversaries by warning them of adverse consequences or legal prosecution.
- **Examples:** Login warning banners (*"Unauthorized access is strictly prohibited and subject to criminal prosecution"*); prominent warning signs; visible guard dogs and security booths.

### 5. Compensating Controls (Alternative Secondary Safeguards)
- **Objective:** An alternative security measure deployed to meet a regulatory requirement (e.g. PCI-DSS) when the primary control cannot be implemented due to technical or business constraints.
- **Classic Scenario:** An enterprise runs an unpatchable legacy ERP application on Windows Server 2008 with a critical known RCE vulnerability.
  - *Primary Control:* Upgrading the operating system (Impossible due to legacy app incompatibility).
  - *Compensating Control:* Placing a Web Application Firewall (WAF) in front of the application with a virtual patch rule, isolating the server in a private non-routable VLAN, and restricting access strictly to an internal IP allowlist.

### 6. Recovery Controls (Restores Core Business Functionality)
- **Objective:** Rebuilds capabilities and data after catastrophic hardware failure or ransomware destruction.
- **Examples:** Restoring systems from offline air-gapped backup repositories; failing over to a secondary disaster recovery (DR) data center site.

---

## 4. Defense in Depth Illustrated

```text
[ Adversary on Public Internet ]
              │
              ▼ [ Layer 1: Perimeter ]: Cloudflare DDoS Scrubbing & WAF
              │
              ▼ [ Layer 2: Network ]: External Firewall & DMZ Microsegmentation
              │
              ▼ [ Layer 3: Identity ]: Centralized IdP with Conditional Access & FIDO2 MFA
              │
              ▼ [ Layer 4: Host / Endpoint ]: Hardened OS, EDR Agent, Disabled USB Ports
              │
              ▼ [ Layer 5: Application ]: Input Validation, Parameterized SQL Queries, CSP
              │
              ▼ [ Layer 6: Data ]: AES-256 Encryption at Rest with AWS KMS Key Rotation
```

If the attacker bypasses the perimeter firewall, they are stopped by MFA; if they compromise a user token, they are stopped by database parameterized queries; if they dump the database files, they are stopped by AES-256 disk encryption.

---

## 5. Separation of Duties (SoD) & Least Privilege

- **Principle of Least Privilege (PoLP):** Users and service accounts are granted only the minimum permissions necessary to complete their assigned job functions.
- **Separation of Duties (SoD):** High-impact actions require multiple independent actors:
  - *Software Engineering:* The developer who writes code cannot be the same person who approves the Pull Request or deploys the binary to production.
  - *Finance:* The clerk who enters an invoice cannot be the manager who approves the bank wire transfer.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Security controls are classified administratively (policies and training), technically (firewalls and encryption), and physically (guards and locks). Functionally, they act as Preventive (stopping threats upfront like MFA), Detective (identifying intrusions like SIEM and IDS), Corrective (remediating damage like EDR quarantine), Deterrent (discouraging attacks), Compensating (temporary alternative safeguards when primary controls cannot be deployed), and Recovery (restoring operations from backups). The overarching architecture must follow Defense in Depth, ensuring no single point of security failure exists, and enforce Separation of Duties to prevent rogue actions or unilateral fraud."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing Corrective controls with Compensating controls.  
  *Correction:* **Corrective** controls remediate damage after an attack occurs (e.g. restoring backups, killing malicious processes). **Compensating** controls are alternative workarounds deployed in advance to offset the inability to implement a primary control (e.g. putting a WAF in front of a legacy unpatchable server).
- **Trap:** Forgetting Administrative controls in technical interviews.  
  *Correction:* Strong technical firewalls are useless if employees fall for social engineering or write passwords on whiteboards. Always emphasize that administrative policies and employee awareness training form the foundation of defense.
