# Cloud Monitoring, Auditing, CSPM, and Cloud Incident Response

## 1. Topic & Definition
Cloud environments generate high-velocity telemetry across control plane API invocations, network flows, and workload runtimes. 

Maintaining effective cloud defense requires:
1. **Auditing & Forensic Visibility:** Immutable recording of every API call via **AWS CloudTrail** (or Azure Activity Logs / GCP Cloud Audit Logs).
2. **Intelligent Threat Detection:** Continuous automated threat analysis via **AWS GuardDuty** analyzing DNS, network flows, and API anomalies.
3. **Posture & Governance Automation:** **Cloud Security Posture Management (CSPM)** identifying compliance deviations and toxic risk combinations across multi-cloud environments.
4. **Cloud Incident Response:** Executing isolation and forensic acquisition procedures without corrupting digital evidence.

---

## 2. How It Works: Cloud Auditing and Threat Telemetry Pipeline

```
+---------------------------------------------------------------------------------------------------+
|                            Cloud Threat Detection and Ingestion Pipeline                          |
+---------------------------------------------------------------------------------------------------+
  TELEMETRY SOURCES:
  [ CloudTrail Mgmt Events ] (Who called what API)
  [ CloudTrail S3 Data Ops ] (Who read which bucket object)  ---> [ Amazon EventBridge / Kinesis ]
  [ VPC Flow Logs (NetFlow)] (Network IP/Port metadata)                   |
  [ Route 53 DNS Queries   ] (Malware DGA / C2 lookups)                   v
                                                             [ AWS GuardDuty ML Engine ]
                                                             - Anomaly Detection
                                                             - Threat Intel Matching (Tor/C2)
                                                                          |
                                                                          v
  [ SIEM: Splunk / Sentinel ] <--- [ AWS Security Hub ] <--- [ High-Severity Finding Triggered ]
  (Central SOC Investigation)      (Aggregated Posture &      (e.g., "EC2 Bitcoin Miner Detected")
                                    CIS Benchmark Scans)
```

### A. AWS CloudTrail: The Immutable Audit Ledger
CloudTrail records every API call made across the AWS account:
- **Management Events (Control Plane):** Captures management actions on resources (`iam:CreateUser`, `ec2:RunInstances`, `s3:CreateBucket`). Enabled on all accounts by default.
- **Data Events (Data Plane):** Captures high-volume operational requests on or within resources (`s3:GetObject`, `s3:PutObject`, `lambda:Invoke`). Disabled by default due to high event volumes.
- **Log File Integrity Validation:** Uses cryptographic hashing (SHA-256) and public key digital signatures to create **Digest Files**. This allows security teams to mathematically prove in court that CloudTrail logs were not tampered with, modified, or deleted after creation.

### B. AWS GuardDuty: ML-Driven Anomaly Detection
GuardDuty runs completely out-of-band without deploying host agents:
- Directly consumes foundational AWS data streams (CloudTrail management logs, VPC Flow Logs, Route 53 DNS queries, EKS audit logs).
- Uses machine learning and threat intelligence to identify compromises:
  - *Cryptocurrency mining:* EC2 instances connecting to known mining pools.
  - *Compromised IAM credentials:* API calls from anomalous geographic regions or Tor exit nodes.
  - *DNS Data Exfiltration:* Excessive high-entropy DNS queries indicating DNS tunneling.

---

## 3. The Modern Cloud Security Acronym Spectrum

```
+------------------------------------+---------------------------------------------------------------+
| Security Technology Category       | Domain Focus & Responsibility                                 |
+------------------------------------+---------------------------------------------------------------+
| **CSPM** (Cloud Security Posture   | Scans cloud control plane APIs for misconfigurations          |
| Management)                        | (e.g., open S3 buckets, unencrypted disks, missing MFA).      |
| **CWPP** (Cloud Workload Protection| Protects running compute workloads (agents on EC2/containers  |
| Platform)                          | detecting malware, unauthorized processes, file integrity).  |
| **CIEM** (Cloud Infrastructure     | Analyzes IAM entitlements; identifies unused excessive       |
| Entitlement Management)            | permissions and unused admin roles across cloud identities.   |
| **CNAPP** (Cloud-Native Application| **Unified Platform:** Combines CSPM + CWPP + CIEM + Container |
| Protection Platform)               | Scanning into a single holistic security graph (e.g., Wiz).   |
+------------------------------------+---------------------------------------------------------------+
```

---

## 4. Key Differences Matrix: CloudTrail vs CloudWatch vs GuardDuty

| Dimension | AWS CloudTrail | Amazon CloudWatch | AWS GuardDuty |
| :--- | :--- | :--- | :--- |
| **Primary Purpose** | **Audit Trail & Governance** | **Operational Monitoring & Metrics**| **Intelligent Threat Detection** |
| **Data Tracked** | "Who called what API from where"| CPU utilization, memory, disk, error logs | Active intrusions, anomalies, breaches |
| **Output Type** | JSON event records | Time-series metrics & log groups | Severity-scored Security Findings (0.1–8.9) |
| **Detection Engine**| None (Pure recording logger) | Alarm thresholds (`CPU > 80%`) | Machine Learning + CTI threat feeds |
| **SOC Usage** | Forensic root-cause investigations | Performance & application debugging | Active incident alert generation |

---

## 5. Cloud Incident Response Playbook: Compromised EC2 Instance

When AWS GuardDuty fires an alert indicating an EC2 instance is communicating with a Cobalt Strike C2 server, responders follow a strict 4-step cloud containment workflow:

```
[ STEP 1: NETWORK ISOLATION VIA SECURITY GROUP ]
Apply an "Isolation Security Group" to the compromised instance's ENI:
- Inbound Rules: NONE (0 rules)
- Outbound Rules: NONE (0 rules)
(Stateful nature drops all existing active TCP C2 connections immediately!)

[ STEP 2: IAM CREDENTIAL REVOCATION ]
- Revoke all active temporary STS credentials issued to the attached EC2 IAM Role.
- Attach an inline explicit Deny policy to the role.

[ STEP 3: VOLATILE MEMORY CAPTURE ]
- Execute an AWS Systems Manager (SSM) Run Command to dump physical RAM to an isolated S3 bucket
  using LiME (Linux Memory Extractor) before shutting down the instance.

[ STEP 4: FORENSIC DISK SNAPSHOT & QUARANTINE ]
- Call `ec2:CreateSnapshot` on all attached EBS volumes.
- Tag snapshots: `Forensic-Evidence=Active-Investigation`.
- Copy snapshot to a dedicated, isolated "Forensics AWS Account" using a Customer Managed KMS key
  to prevent tampering by attackers in the compromised account.
```

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Cloud security operations relies on continuous auditing, intelligent threat detection, and automated posture governance. AWS CloudTrail serves as the non-repudiation audit backbone, capturing every control plane API invocation with SHA-256 digest validation to prove tamper-resistance. AWS GuardDuty provides agentless threat detection, applying machine learning over VPC Flow Logs, DNS queries, and CloudTrail to uncover credential hijacking and malware C2 beaconing. At the governance layer, Cloud-Native Application Protection Platforms (CNAPPs) unite CSPM for misconfiguration detection, CIEM for IAM least-privilege analysis, and CWPP for workload protection. When an incident occurs, responders execute cloud-native containment: applying isolation security groups to sever connections, invalidating temporary STS role credentials, and preserving EBS volume snapshots to an isolated forensics account."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Relying on CloudTrail for real-time alerting. *Correction:* CloudTrail delivers logs to S3 with a standard 5 to 15-minute propagation delay; real-time alerting requires GuardDuty or EventBridge rules triggered directly on API events.
- **Trap 2:** Terminating a compromised EC2 instance immediately. *Correction:* Terminating the instance destroys volatile RAM memory and root disk evidence; quarantine the instance with an isolation security group, dump RAM, and take EBS snapshots first.
- **Trap 3:** Confusing CloudTrail with CloudWatch. *Correction:* CloudTrail records API calls (auditing); CloudWatch monitors operational performance metrics and application logs.

### Expected Follow-Up Questions
1. *What is a CloudTrail Digest File?*
   - A cryptographically signed file generated hourly containing the SHA-256 hashes of all log files delivered during that period, chaining digests together to enable verification that logs have not been modified or deleted.
2. *What is a "Toxic Combination" in CNAPP / CSPM?*
   - An attack path created by combining multiple distinct findings that may each seem moderate in isolation: e.g., an EC2 instance with an unpatched CVE, running in a public subnet, possessing an IAM role with administrative S3 permissions.
