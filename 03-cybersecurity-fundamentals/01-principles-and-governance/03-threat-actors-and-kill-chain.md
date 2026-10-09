# Threat Actors, The Cyber Kill Chain & MITRE ATT&CK Framework

> **Domain:** Cybersecurity Fundamentals  
> **Sub-Domain:** Threat Intelligence & Attack Modeling  
> **Interview Importance:** Very High / Standard Incident Response & SOC Question  

---

## 1. Topic & Definitions

- **Threat Actor:** An individual or organized group that executes malicious actions targeting computer systems, networks, or digital infrastructure.
- **Advanced Persistent Threat (APT):** A state-sponsored or highly sophisticated threat group that maintains long-term, stealthy, undetected access to a target network for strategic espionage, intelligence gathering, or sabotage.
- **The Lockheed Martin Cyber Kill Chain:** A military-derived 7-stage operational model that outlines the sequential phases an adversary must complete to execute a cyber attack. Defeating an attack at *any single stage* breaks the chain and stops the breach.
- **MITRE ATT&CK (Adversarial Tactics, Techniques, and Common Knowledge):** A globally accessible, comprehensive knowledge base of real-world adversary behavior, cataloging the specific **Tactics** (goals) and **Techniques** (methods) used by threat groups.

---

## 2. Threat Actor Taxonomy & Motivations

```text
+─────────────────────────────────────────────────────────────────────────────+
|                         THREAT ACTOR PROFILES MATRIX                        |
+─────────────────────────────────────────────────────────────────────────────+
| 1. NATION-STATE / APTs (Advanced Persistent Threats):                       |
|    • Motivation: Geopolitical espionage, military advantage, IP theft.     |
|    • Attributes: Unlimited budgets, elite developers, custom 0-days,        |
|      stealthy persistence (dwell times of months/years).                    |
|─────────────────────────────────────────────────────────────────────────────|
| 2. ORGANIZED CYBERCRIME (e-Crime / Ransomware Syndicates):                  |
|    • Motivation: Purely financial profit (extortion, fraud, card theft).    |
|    • Attributes: Highly structured corporations, Ransomware-as-a-Service    |
|      (RaaS), fast execution speed, monetization via cryptocurrency.         |
|─────────────────────────────────────────────────────────────────────────────|
| 3. INSIDER THREATS (Malicious or Negligent):                                |
|    • Motivation: Financial bribes, revenge, espionage, or pure negligence.  |
|    • Attributes: Already possess legitimate internal credentials and VPN;   |
|      bypasses perimeter firewalls completely. Most difficult to detect.     |
|─────────────────────────────────────────────────────────────────────────────|
| 4. HACKTIVISTS:                                                             |
|    • Motivation: Ideological, political, religious, or social causes.       |
|    • Attributes: Website defacements, DDoS attacks, public data leaks.      |
|─────────────────────────────────────────────────────────────────────────────|
| 5. SCRIPT KIDDIES:                                                          |
|    • Motivation: Thrill-seeking, curiosity, internet prestige.              |
|    • Attributes: Zero exploit development capability; rely exclusively on   |
|      pre-built scripts, Metasploit, or automated scanner tools.             |
+─────────────────────────────────────────────────────────────────────────────+
```

---

## 3. The Lockheed Martin Cyber Kill Chain (7 Sequential Phases)

```text
Phase 1: RECONNAISSANCE ──► Phase 2: WEAPONIZATION ──► Phase 3: DELIVERY
      │                             │                            │
      ▼                             ▼                            ▼
Phase 4: EXPLOITATION   ──► Phase 5: INSTALLATION  ──► Phase 6: COMMAND & CONTROL (C2)
      │
      ▼
Phase 7: ACTIONS ON OBJECTIVES (Data Exfiltration / Ransomware Deployment)
```

| Kill Chain Stage | Adversary Objective | Example Activity | Defensive Countermeasure & Disruption Point |
| :--- | :--- | :--- | :--- |
| **1. Reconnaissance** | Harvest intelligence on target systems, employees, and perimeter. | OSINT scraping on LinkedIn, Nmap scanning, Harvester email enumeration. | Threat intel monitoring, web server banner grabbing obfuscation, strict social media policies. |
| **2. Weaponization** | Combine an exploit with a malicious payload (executable/macro). | Embedding an exploit in a weaponized PDF or Microsoft Office macro. | Threat intelligence analysis of weaponized binaries, Antivirus/YARA signature generation. |
| **3. Delivery** | Transmit the weaponized payload to the intended victim. | Spear-phishing email with malicious attachment, drive-by download, rogue USB drop. | Email security gateway (Secure Email Gateway - SEG), SPF/DKIM/DMARC, disabling USB ports. |
| **4. Exploitation** | Trigger the exploit on the victim's hardware to execute code. | Exploiting a buffer overflow in Adobe Reader, unpatched Log4Shell vulnerability. | Timely vulnerability patching, Endpoint Detection and Response (EDR), DEP/ASLR memory protections. |
| **5. Installation** | Establish a persistent foothold on the target operating system. | Creating a Windows Run registry key, scheduled task, or dropping a web shell. | Application allowlisting (AppLocker), File Integrity Monitoring (FIM), restricting admin rights. |
| **6. Command & Control** | Establish an interactive, remote communications channel back to C2. | Malware beacons back to attacker server over port 443 or DNS queries. | Outbound firewall egress filtering, DNS sinkholing, TLS interception proxy, blocking known C2 IPs. |
| **7. Actions on Objectives**| Accomplish the end goal of the attack campaign. | Stealing and exfiltrating intellectual property, encrypting drives with ransomware. | Data Loss Prevention (DLP), network microsegmentation, immutable air-gapped backups. |

---

## 4. The MITRE ATT&CK Matrix: Tactics vs. Techniques vs. Procedures

Unlike the sequential Kill Chain, the **MITRE ATT&CK Framework** is a non-linear, matrix-based knowledge repository reflecting real-world operations across 14 enterprise **Tactics**:

```text
TACTIC (The Adversary's Objective - "WHY"):
└─► e.g., TA0006: Credential Access

    TECHNIQUE (The Method Used - "WHAT"):
    └─► e.g., T1003: OS Credential Dumping
        └─► SUB-TECHNIQUE: T1003.001: LSASS Memory Dumping

            PROCEDURE (The Specific Implementation - "HOW"):
            └─► APT29 executed Mimikatz with command "sekurlsa::logonpasswords"
```

### The 14 MITRE Enterprise Tactics:
1. **Reconnaissance:** Gathering information to plan future adversary operations.
2. **Resource Development:** Establishing infrastructure (buying domains, C2 servers).
3. **Initial Access:** Gaining an entry foothold in the network (phishing, public exploit).
4. **Execution:** Running malicious code (PowerShell, Command Prompt, Python).
5. **Persistence:** Maintaining access across system reboots (registry keys, services).
6. **Privilege Escalation:** Gaining higher permissions (root / SYSTEM).
7. **Defense Evasion:** Avoiding detection (disabling antivirus, obfuscating code).
8. **Credential Access:** Stealing account usernames and password hashes (Mimikatz, spraying).
9. **Discovery:** Observing the environment (scanning active directories, listing shares).
10. **Lateral Movement:** Traversing from one system to another across the network (Pass-the-Hash, PsExec).
11. **Collection:** Gathering data of interest (bundling files, clipboard grabbing).
12. **Command and Control:** Communicating with compromised systems under operator control.
13. **Exfiltration:** Stealing and transferring collected data out of the network.
14. **Impact:** Manipulating, interrupting, or destroying systems (ransomware disk wipe).

---

## 5. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Threat actors span unsophisticated script kiddies, ideologically motivated hacktivists, financially driven cybercrime cartels running Ransomware-as-a-Service, and nation-state APTs characterized by advanced tradecraft, unlimited budgets, and stealthy persistence. We model adversary operations using the Lockheed Martin Cyber Kill Chain—a 7-stage sequential framework (Reconnaissance, Weaponization, Delivery, Exploitation, Installation, C2, Actions on Objectives) where breaking any single link terminates the attack. In modern SOC and threat hunting environments, we map detections against the MITRE ATT&CK matrix, which categorizes adversarial actions into 14 Tactics representing goals and hundreds of Techniques representing methods, enabling engineering teams to validate coverage against specific threat actor profiles."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing the Cyber Kill Chain with the MITRE ATT&CK Matrix.  
  *Correction:* The **Cyber Kill Chain** is a linear, chronological 7-phase model describing the high-level progression of an intrusion. **MITRE ATT&CK** is a non-linear, granular matrix detailing specific real-world techniques used at any point during post-compromise activity.
- **Trap:** Treating Insiders as simple disgruntled employees.  
  *Correction:* In technical discussions, emphasize that **negligent insiders** (clicking phishing links, exposing cloud storage) cause far more operational incidents than malicious saboteurs, which is why technical controls (least privilege and DLP) are mandatory.
