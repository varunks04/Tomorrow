# Active Directory Attacks, Persistence, and Enterprise Defenses

## 1. Topic & Definition
Active Directory (AD) is the primary target in enterprise post-exploitation. Once an adversary gains an initial foothold on a domain-joined workstation, their objective is **Lateral Movement and Privilege Escalation** toward **Tier-0 Assets (Domain Controllers and Domain Admins)**.

This guide analyzes the core offensive techniques utilized by adversaries (e.g., Kerberoasting, Golden Tickets, DCSync) and the enterprise-grade defense models (Tiering Architecture, LAPS, gMSA, Event ID telemetry) required to stop them.

---

## 2. How AD Attacks Work: Mechanics and Exploitation Paths

```
+---------------------------------------------------------------------------------------------------+
|                            Active Directory Attack Progression Graph                              |
+---------------------------------------------------------------------------------------------------+

   [ Initial Foothold ] (Phished Workstation / Tier-2)
            |
            |-- (LSASS Dumping / Mimikatz / Windows LAPS not deployed)
            v
   [ Local Admin Access on Workstation ]
            |
            |-- (BloodHound LDAP Enumeration: Graph Path Analysis)
            |-- (Kerberoasting: Request SPN TGS tickets -> Crack offline)
            |-- (AS-REP Roasting: Accounts with DONT_REQ_PREAUTH)
            v
   [ Domain User / Service Account Compromise ]
            |
            |-- (Privilege Escalation via Unconstrained Delegation / GPO Abuse)
            v
   [ Tier-1 Member Server / Domain Controller Admin Access ]
            |
            +---> [ DCSync Attack (MS-DRSR API: Extract all NTDS.DIT password hashes) ]
            |
            +---> [ Golden Ticket Generation (KRBTGT hash -> 10-year root persistence) ]
            |
            +---> [ Complete Enterprise Forest Compromise ]
```

### Deep Dive: Signature Active Directory Attacks

#### 1. Kerberoasting
- **Target:** Service accounts configured with a Service Principal Name (SPN).
- **Mechanism:** Any domain user requests a TGS Service Ticket via `TGS-REQ`. The KDC encrypts the ticket using the service account's NTLM password hash. The attacker extracts the ticket from memory, exports it as a hash (`$krb5tgs$23$...`), and runs an offline dictionary attack (`hashcat -m 13100`).

#### 2. AS-REP Roasting
- **Target:** User accounts where `Do not require Kerberos preauthentication` (`DONT_REQ_PREAUTH` = 0x400000) is checked.
- **Mechanism:** Attacker requests an AS-REQ for that username without providing credentials. The KDC returns an AS-REP containing encrypted session data. Attacker extracts and cracks the hash offline (`hashcat -m 18200`).

#### 3. DCSync (Directory Replication Service Remote Protocol)
- **Mechanism:** Utilizes the legitimate Microsoft Directory Replication Service Remote Protocol (MS-DRSR).
- **Exploitation:** If an attacker compromises an account with `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All` permissions (default for Domain Admins and Domain Controllers), they simulate a Domain Controller replication request.
- **Impact:** Dumps all password hashes directly from `NTDS.DIT`—including the critical `krbtgt` account hash—over the network without executing code on the DC!

#### 4. Golden Ticket vs Silver Ticket
- **Golden Ticket:** Created by forging a TGT using the decrypted **`krbtgt` account hash**. Grants arbitrary group memberships (Enterprise Admins, Domain Admins) and custom validity (e.g., 10 years). Can access any service across the entire forest.
- **Silver Ticket:** Created by forging a TGS Service Ticket using a compromised target **Service Account hash**. Operates without contacting the KDC; grants total admin control over that single service (e.g., CIFS, MSSQL).

---

## 3. Practical Commands and SOC Event ID Detection

### Attack Detection: Essential Windows Security Event IDs

| Event ID | Activity Monitored | SOC Triage Significance |
| :--- | :--- | :--- |
| **4768** | Kerberos TGT Request (AS-REQ) | Detects AS-REP Roasting (Ticket Encryption: `0x17` RC4) |
| **4769** | Kerberos Service Ticket Request (TGS-REQ)| Detects Kerberoasting (High volume of TGS requests with RC4) |
| **4771** | Kerberos Pre-Authentication Failed | Detects password brute-forcing or Kerberos password spraying |
| **4624** | Successful Logon | Type 3 = Network (lateral movement), Type 10 = Remote Desktop (RDP) |
| **4662** | Operation Performed on Object | Detects **DCSync** when matching GUIDs `1131f6aa-9c07-11d1-f79f-00c04fc2dcd2` |
| **4672** | Special Privileges Assigned | Detects sensitive administrative logons (SeDebugPrivilege, etc.) |

### Defensive PowerShell Diagnostics
```powershell
# Find all accounts vulnerable to AS-REP Roasting
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $True} -Properties DoesNotRequirePreAuth | Select Name, SamAccountName

# Find all accounts with Service Principal Names vulnerable to Kerberoasting
Get-ADUser -Filter {ServicePrincipalName -like "*"} -Properties ServicePrincipalName | Select SamAccountName, ServicePrincipalName

# Check members of Protected Users Security Group
Get-ADGroupMember -Identity "Protected Users" | Select Name, SamAccountName
```

---

## 4. Key Differences Matrix: Golden Ticket vs Silver Ticket

| Feature | Golden Ticket | Silver Ticket |
| :--- | :--- | :--- |
| **Ticket Type Forged** | **TGT (Ticket Granting Ticket)** | **TGS (Service Ticket)** |
| **Required Secret** | **`krbtgt` account password hash** | **Target Service Account password hash** |
| **KDC Interaction** | Must contact KDC for subsequent TGS requests | **Zero KDC interaction** (completely stealthy from DC logs) |
| **Scope of Impact** | **Entire Forest & All Services** | **Target Service Only** |
| **Privilege Level** | Arbitrary (Domain Admin, Enterprise Admin) | Full control over the specific application |
| **Remediation** | Must reset `krbtgt` password **twice** | Reset the specific service account password |

---

## 5. Enterprise Defensive Architecture

### A. The Microsoft Tiering / Enterprise Access Model
Segregates administrative identities into non-overlapping tiers to prevent credential inheritance from low to high trust:
- **Tier 0 (Control Plane):** Domain Controllers, PKI/CAs, Entra Connect, PAM systems. Managed strictly from dedicated, hardened Privileged Access Workstations (PAWs). Tier-0 credentials must **never** log into Tier 1 or Tier 2 machines.
- **Tier 1 (Management Plane):** Enterprise member servers, databases, cloud hypervisors, internal web applications.
- **Tier 2 (Workstation Plane):** End-user workstations, printers, VoIP devices.

```
+---------------------------------------------------------------------------------+
|                        Tiered Administrative Boundary                           |
+---------------------------------------------------------------------------------+
|   TIER 0: Domain Controllers, IdPs, PKI   <--- Strictly Isolated Admin PAWs     |
+---------------------------------------------------------------------------------+
|                     | NO CREDENTIAL REUSE ALLOWED DOWNWARD                      |
|                     v                                                           |
|   TIER 1: Member Servers, Databases, Apps                                       |
+---------------------------------------------------------------------------------+
|                     | NO CREDENTIAL REUSE ALLOWED DOWNWARD                      |
|                     v                                                           |
|   TIER 2: End-User Laptops & Workstations                                       |
+---------------------------------------------------------------------------------+
```

### B. Core Hardening Technologies
1. **Windows LAPS (Local Administrator Password Solution):** Automatically randomizes and rotates the local Administrator password on every endpoint, storing the complex password in encrypted attributes on the DC. Completely stops lateral Pass-the-Hash movement using shared local admin passwords.
2. **Group Managed Service Accounts (gMSA):** Eliminates service account passwords entirely. Windows automatically handles 128-character password generation and 30-day rotation, making Kerberoasting ineffective.
3. **Protected Users Security Group:**
   - Disables NTLM authentication.
   - Disables DES and RC4 encryption in Kerberos (forces AES).
   - Prevents credential caching in LSASS (cannot be extracted via Mimikatz).
   - Reduces TGT lifetime to 4 hours.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Active Directory attacks focus on leveraging low-privilege footholds to escalate laterally to Tier-0 Domain Controllers. Primary attack vectors include Kerberoasting against SPN service accounts, AS-REP Roasting against pre-auth-disabled users, and DCSync via directory replication rights to extract the full `NTDS.DIT` database. Enterprise defense requires dismantling these paths: implementing Windows LAPS to eliminate shared local administrator passwords, transitioning service accounts to gMSAs with automated 128-character rotation, enforcing the Microsoft Tiering model to keep Tier-0 credentials off Tier-2 endpoints, and monitoring Event IDs 4769 and 4662 for anomalous ticket requests and DCSync replication calls."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Believing resetting the `krbtgt` password once eliminates a Golden Ticket. *Correction:* Active Directory keeps the current and previous `krbtgt` password in memory to prevent service disruption. To invalidate Golden Tickets, the `krbtgt` password must be **reset twice**, with replication delay observed between resets.
- **Trap 2:** Confusing Pass-the-Hash with Pass-the-Ticket. *Correction:* Pass-the-Hash uses NTLM hashes; Pass-the-Ticket uses stolen Kerberos TGT/TGS tickets without cracking or touching the underlying password hash.
- **Trap 3:** Thinking DCSync requires running tools directly on the Domain Controller. *Correction:* DCSync runs remotely over the network from any compromised domain-joined machine using the MS-DRSR replication API.

### Expected Follow-Up Questions
1. *Why is BloodHound so dangerous to an unprepared enterprise?*
   - BloodHound uses graph theory to visualize indirect, non-obvious attack paths (e.g., User A is in Group B, which has GenericAll on Group C, which has local admin on Server D, where a Domain Admin session is cached). It discovers escalation paths invisible in standard flat reports.
2. *How does Windows LAPS prevent lateral movement?*
   - Historically, organizations set the same local admin password across thousands of laptops. An attacker compromising one workstation cracked the local hash and used Pass-the-Hash to compromise every computer in the fleet. LAPS ensures every single endpoint has a unique, rotating password.
