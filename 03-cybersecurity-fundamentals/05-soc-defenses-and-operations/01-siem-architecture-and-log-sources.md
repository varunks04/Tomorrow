# SIEM Architecture, Log Aggregation, Ingestion Pipelines & Correlation Rules

> **Domain:** Cybersecurity Fundamentals & Security Operations (SecOps)  
> **Sub-Domain:** SOC Engineering & Threat Detection  
> **Interview Importance:** Very High / Foundational SOC Analyst & Security Engineer Question  

---

## 1. Topic & Definitions

- **SIEM (Security Information and Event Management):** A centralized security management platform that aggregates, normalizes, correlates, and analyzes log and event data from across an enterprise IT infrastructure to provide real-time threat detection, incident triage, and regulatory compliance reporting.
- **Log Source:** Any hardware device, operating system, application, or cloud service that generates timestamped audit records documenting operational events (e.g. firewalls, domain controllers, web servers, EDR agents).
- **Log Normalization:** The process of converting disparate, vendor-specific log formats (JSON, CEF, LEEF, Syslog, XML) into a unified, standardized schema (e.g., Elastic Common Schema - ECS, or CIM in Splunk) with standardized field names (`src_ip`, `dest_ip`, `user`, `action`).
- **Correlation Rule:** A logical detection statement that analyzes normalized log streams across multiple independent sources over time windows to identify suspicious patterns or multi-stage attacks.

---

## 2. SIEM Architectural Pipeline

```text
[ LOG GENERATION: Disparate Sources ]
• Firewalls / Routers (Syslog 514)
• Windows Domain Controllers (WinEventLog)
• Cloud Infrastructure (AWS CloudTrail, Azure Activity)
• Endpoint Security (EDR Agent Telemetry)
• Web Applications (Nginx / Apache / WAF)
                     │
                     ▼ 1. INGESTION & TRANSPORT
[ Log Shippers / Forwarders (Splunk Universal Forwarder, Logstash, Fluentd) ]
                     │
                     ▼ 2. PARSING & NORMALIZATION
[ Processing Layer ] ── Converts raw logs into standard schema:
                       Raw: "Src=10.0.0.1 DstPort=443 Action=Block"
                       Normalized: { src_ip: "10.0.0.1", dest_port: 443, event_outcome: "denied" }
                     │
                     ▼ 3. INDEXING & RETENTION
[ High-Performance Storage Cluster (Hot / Warm / Cold / Frozen Tiers) ]
                     │
                     ▼ 4. CORRELATION & DETECTION ENGINE
[ Real-Time Rule Matching & Machine Learning Anomaly Detection ]
                     │
                     ▼ [ TRIGGER: Threat Pattern Confirmed! ]
[ SOC ALERT / INCIDENT CREATED IN SOAR (Splunk SOAR, Cortex XSOAR) ]
```

---

## 3. High-Value Enterprise Log Sources

| Log Category | Primary Source Systems | High-Value Telemetry & Event Types | Security Threat Detected |
| :--- | :--- | :--- | :--- |
| **Authentication & Identity** | Windows Active Directory Domain Controllers, Okta, Azure AD | - Event ID 4624 (Successful Login)<br>- Event ID 4625 (Failed Login)<br>- Event ID 4720 (User Created)<br>- Event ID 4728 (Added to Admin Group) | Brute force, Password spraying, Unauthorized account creation, Privilege escalation. |
| **Network & Perimeter** | Next-Gen Firewalls (Palo Alto, Fortinet), VPN Concentrators | Inbound/Outbound connection logs, dropped packets, VPN logins, source/dest IPs & ports. | C2 beaconing, port scanning, unauthorized external data egress, VPN anomalies. |
| **Host / Endpoint Activity**| Windows Event Logs, Sysmon, EDR agents, Linux `auditd` | - Event ID 4688 (Process Creation with Command Line)<br>- Event ID 7045 (Service Installed)<br>- Sysmon Event ID 1 (Process Create)<br>- Sysmon Event ID 3 (Network Connect) | Process injection, LOLBin execution (PowerShell/Certutil), persistence mechanisms. |
| **DNS Infrastructure** | Internal DNS Resolvers (BIND, Windows DNS) | DNS lookup queries, resolved IPs, failed query codes (NXDOMAIN). | DNS tunneling, C2 domain resolution, malware domain queries. |
| **Web & Application** | WAF (Cloudflare, AWS WAF), Reverse Proxies (Nginx) | HTTP URI paths, user-agents, response status codes, raw request headers. | SQL injection, XSS attempts, web directory brute force, credential stuffing. |
| **Cloud Audit** | AWS CloudTrail, Google Cloud Audit Logs | API calls, IAM role assumptions (`AssumeRole`), S3 bucket permission changes. | Cloud persistence, unauthorized infrastructure provisioning, S3 data leaks. |

---

## 4. How Correlation Rules Work (Real-World Examples)

A correlation rule combines multiple distinct event streams across a **Time Window** to detect threats that individual logs cannot uncover in isolation:

### Rule Example 1: Detecting Password Spraying
```text
CORRELATION LOGIC:
IF (COUNT of DISTINCT target_user in Event ID 4625 [Failed Login]) >= 20
AND (event.src_ip is identical)
AND (time_window <= 10 minutes)
AND (COUNT of DISTINCT target_user in Event ID 4624 [Successful Login]) >= 1
THEN:
GENERATE HIGH SEVERITY ALERT: "Successful Password Spray Attack Detected from IP <src_ip>!"
```

### Rule Example 2: Detecting Ransomware Ingestion
```text
CORRELATION LOGIC:
IF (Sysmon Event ID 1: Process 'vssadmin.exe' spawned with argument 'delete shadows /all /quiet')
AND (Followed within 60 seconds by File System Event: > 500 file modification events
     renaming extensions to '.locked' or '.enc')
THEN:
GENERATE CRITICAL SEVERITY ALERT: "Active Ransomware Encryption Behavior on Host <hostname>!"
EXECUTE SOAR PLAYBOOK: Instantly isolate host from the network!
```

---

## 5. Major SIEM Platforms

- **Splunk Enterprise Security:** The traditional market leader; highly flexible Splunk Processing Language (SPL), massive ecosystem of apps.
- **Elastic Security (ELK Stack):** Open-source foundation (Elasticsearch, Logstash, Kibana), native Elastic Common Schema (ECS), scalable distributed indexing.
- **Microsoft Sentinel:** Cloud-native SIEM/SOAR deeply integrated with Azure AD, Microsoft Defender XDR, and Office 365 telemetry, billed on consumption.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"A SIEM provides centralized visibility and threat detection across an enterprise by ingesting, parsing, normalizing, and indexing telemetry from diverse log sources—including Active Directory authentication events, firewall network sessions, DNS queries, endpoint Sysmon process tracking, and cloud audit logs like CloudTrail. The power of a SIEM lies in its correlation engine, which evaluates multi-source events across defined time windows to identify complex attack sequences—such as correlating external port scans with internal failed logins followed by anomalous outbound data transfers. In modern security operations, SIEMs integrate with SOAR platforms to automate repetitive tier-1 triage and execute automated containment playbooks."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing SIEM with Log Storage.  
  *Correction:* A raw log server (like a basic rsyslog box or S3 bucket) merely stores files. A **SIEM** normalizes, indexes, enriches with threat intelligence, and executes real-time correlation queries to trigger automated alerts.
- **Trap:** Forgetting Sysmon when discussing Windows logs.  
  *Correction:* Default Windows Event Logs miss critical details. Mentioning **Sysmon (System Monitor)**—which logs full command-line arguments (Event 1), network connections by process (Event 3), and DLL loading (Event 7)—demonstrates deep practical SOC engineering expertise.
