# Active Directory (AD DS) and LDAP Architecture

## 1. Topic & Definition
**Active Directory Domain Services (AD DS)** is Microsoft's distributed, hierarchical directory service that provides centralized identity management, authentication, policy enforcement, and resource administration across Windows-centric enterprise networks.

**Lightweight Directory Access Protocol (LDAP — RFC 4511)** is the vendor-neutral, network application protocol used to query, search, and modify directory service data over IP networks.

### Core Active Directory Hierarchical Structure
```
[Enterprise Forest: root.corp] 
      |
      +---> Domain Trees: americas.corp, emea.corp, apac.corp
                 |
                 +---> Domains: child.americas.corp
                            |
                            +---> Organizational Units (OUs): IT, Finance, HR
                                       |
                                       +---> Leaf Objects: Users, Computers, Groups, Service Accounts
```

1. **Forest:** The highest security and administrative boundary in AD. Shares a single common logical schema, directory configuration, and Global Catalog.
2. **Domain:** The administrative partition of the directory database (`ntds.dit`). Boundaries for account policies, password requirements, and Kerberos realms.
3. **Organizational Unit (OU):** Logical containers within a domain used to organize objects and apply Group Policy Objects (GPOs) and delegated administrative controls.
4. **Schema:** The master blueprint defining all object classes (e.g., `user`, `group`, `computer`) and the attributes (e.g., `sAMAccountName`, `userPrincipalName`, `memberOf`) that can exist in the directory.
5. **Global Catalog (GC):** A domain controller containing a full replica of all objects in its own domain and a partial, searchable replica of every object in all other domains across the entire forest (operates on port 3268 / 3269 SSL).

---

## 2. How It Works: LDAP Operations and Directory Querying

### A. Distinguished Names (DN) & Relative Distinguished Names (RDN)
Every object in Active Directory has a globally unique path in the directory tree:
```
CN=Alice Smith,OU=SecOps,OU=IT,DC=americas,DC=corp,DC=local
\___________/ \_________________/ \_______________________/
     RDN              OUs                   Domain Components
```
- `CN` (Common Name): Name of the leaf object (`CN=Alice Smith`).
- `OU` (Organizational Unit): Parent container path.
- `DC` (Domain Component): Each segment of the DNS domain name (`DC=americas`, `DC=corp`, `DC=local`).

### B. LDAP Communication & Protocol Flow
```
Client (Workstation / App)                         Domain Controller (LDAP / LDAPS)
        |                                                        |
        |--- 1. TCP SYN (Port 389 LDAP / Port 636 LDAPS) ------->|
        |<-- 2. TCP SYN-ACK ------------------------------------|
        |--- 3. TCP ACK ---------------------------------------->|
        |                                                        |
        |--- 4. LDAP Bind Request (Credentials / SASL Kerberos) ->|
        |<-- 5. LDAP Bind Response (Success) -------------------|
        |                                                        |
        |--- 6. LDAP Search Request ---------------------------->|
        |    BaseDN: "DC=americas,DC=corp,DC=local"              |
        |    Scope: Subtree (Whole branch)                       |
        |    Filter: (&(objectClass=user)(sAMAccountName=alice)) |
        |    Attributes: [mail, memberOf, department]            |
        |                                                        |
        |<-- 7. LDAP Search Result Entry (Object Attributes) ---|
        |<-- 8. LDAP Search Result Done -------------------------|
        |                                                        |
        |--- 9. LDAP Unbind Request ---------------------------->|
```

---

## 3. Practical Commands, LDAP Queries, and Filters

### Standard LDAP Search Filter Syntax (RFC 4515)
- **Equality:** `(sAMAccountName=jdoe)`
- **Presence:** `(mail=*)`
- **AND Operator `(& ... )`:** `(&(objectClass=user)(department=Engineering))`
- **OR Operator `(| ... )`:** `(|(department=Finance)(department=Accounting))`
- **NOT Operator `(! ... )`:** `(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` *(Finds all active, non-disabled user accounts)*

### Diagnostic LDAP Commands
```bash
# Query active directory users over secure LDAPS using ldapsearch (Linux CLI)
ldapsearch -x -H ldaps://dc01.corp.local:636 \
  -D "CN=svc_scan,OU=ServiceAccounts,DC=corp,DC=local" -W \
  -b "DC=corp,DC=local" \
  "(&(objectCategory=person)(objectClass=user)(title=*Architect*))" \
  sAMAccountName mail memberOf

# Query Domain Controllers via PowerShell (RSAT)
Get-ADUser -Filter {Enabled -eq $true -and Department -eq "Security"} -Properties MemberOf, PasswordLastSet

# Enumerate Domain Trust relationships
nltest /domain_trusts
```

---

## 4. Key Differences Matrix: LDAP (389) vs LDAPS (636) vs StartTLS

| Feature | Plain LDAP | LDAP over SSL (LDAPS) | LDAP with StartTLS |
| :--- | :--- | :--- | :--- |
| **Default Port** | TCP 389 | TCP 636 (Global Catalog: 3269) | TCP 389 (upgraded) |
| **Encryption** | None (Plaintext) | TLS/SSL from initial connection | Starts plaintext; upgrades via extended op |
| **Security Risk** | Credentials sent in cleartext (Sniffing) | Secure; requires trusted DC certificate | Secure once negotiated; downgrade vulnerable |
| **Modern Standard** | Strongly discouraged / Disabled | Enterprise Standard | RFC 4511 Preferred fallback |

---

## 5. Cybersecurity Relevance, Threats & Attack Vectors

### 1. LDAP Injection Attacks
- **Mechanism:** Applications that dynamically construct LDAP query filters using untrusted user input without sanitization.
- **Exploitation:**
  - Legitimate query: `(&(user=" + input_user + ")(password=" + input_pass + "))`
  - Attacker enters: `input_user = *` and `input_pass = *)(|(objectClass=*)`
  - Resulting Filter: `(&(user=*)(password=*)(|(objectClass=*)))`
  - The query evaluates to true for the first record in the directory, bypassing authentication.
- **Defense:** Use parameterized LDAP interfaces or sanitize input by escaping LDAP special characters (`(`, `)`, `*`, `\`, `&`, `|`, `!`).

### 2. Anonymous LDAP Bind & Cleartext Sniffing
- **Risk:** If Domain Controllers permit Anonymous LDAP Binds, unauthenticated adversaries connected to the internal LAN can enumerate all domain user accounts, groups, and organizational charts.
- **Defense:** Enforce LDAP signing and channel binding (ADV190023 / CVE-2017-8563), disable cleartext LDAP bind over port 389, and disallow Anonymous Binds.

### 3. Active Directory Reconnaissance via LDAP Enumeration
- **Tools:** `BloodHound`, `SharpHound`, `ldapdomaindump`.
- **Mechanism:** Any standard, low-privileged domain user account possesses default read permissions over the majority of the Active Directory database via LDAP queries. Attackers map domain trust paths, local admin rights, and privilege escalation paths to Domain Admin.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Active Directory Domain Services (AD DS) is Microsoft's hierarchical enterprise directory architecture organized into Forests, Domains, and Organizational Units (OUs), centered around the `ntds.dit` database and replicated via Domain Controllers. LDAP is the querying protocol operating over TCP 389 (or encrypted LDAPS over TCP 636). In cybersecurity, LDAP is critical because default AD permissions allow any unprivileged domain user to execute LDAP searches to map out every user, group membership, and server in the network. Securing AD DS requires enforcing LDAPS with channel binding, eliminating unconstrained delegation, and applying the Principle of Least Privilege across OU delegation."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Believing a Domain is the ultimate security boundary. *Correction:* The **Forest** is the ultimate security boundary in Active Directory. A Domain Admin in a child domain can compromise the root domain unless separated by isolated forest boundaries.
- **Trap 2:** Confusing LDAP with Kerberos. *Correction:* LDAP is a directory querying and modification protocol; Kerberos is a ticket-based authentication protocol.
- **Trap 3:** Assuming non-admin users cannot query Active Directory. *Correction:* By default in Windows AD, any domain user can read almost all object attributes in the entire domain via LDAP.

### Expected Follow-Up Questions
1. *What is the difference between a Global Catalog server and a standard Domain Controller?*
   - A standard DC stores full attribute data for its own domain only. A Global Catalog (GC) DC stores a full replica of its own domain plus a partial, indexed replica of all objects across the entire forest for universal cross-domain lookups.
2. *What is LDAP Channel Binding?*
   - A security enhancement that cryptographically binds the outer TLS transport channel to the inner LDAP SASL authentication session, preventing Adversary-in-the-Middle (AiTM) relay attacks (e.g., NTLM relaying to LDAP).
