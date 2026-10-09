# Multi-Cloud Security Cross-Reference Matrix: AWS vs. Azure vs. GCP

## 1. Topic & Definition
Enterprise architectures increasingly deploy across multiple public cloud providers (Amazon Web Services, Microsoft Azure, Google Cloud Platform). While underlying security concepts (least privilege, network segmentation, envelope encryption, threat detection) remain consistent, each provider uses distinct terminology, identity models, network constructs, and governance abstractions.

This master matrix provides a direct, comprehensive cross-reference to translate security controls, configurations, and incident response terminology seamlessly across the "Big Three" cloud ecosystems.

---

## 2. Master Multi-Cloud Security Cross-Reference Matrix

```
+------------------------------------+--------------------------+--------------------------+--------------------------+
| Security Domain                    | Amazon Web Services (AWS)| Microsoft Azure          | Google Cloud Platform    |
+------------------------------------+--------------------------+--------------------------+--------------------------+
| **Root Identity / Directory**      | AWS IAM / AWS Identity Ctr| Microsoft Entra ID       | Cloud Identity / G Suite |
| **Identity Entity**                | IAM User, Group, Role    | User, Group, App Reg     | Google Account, Group, SA|
| **Machine / Workload Identity**    | IAM Role (STS AssumeRole)| Managed Identity         | Service Account (OAuth)  |
| **Temporary Credential Service**   | AWS STS                  | Entra ID OAuth2 Token End| STS / OAuth2 Endpoints   |
| **Multi-Account Governance**       | AWS Organizations + SCPs | Management Groups + Azure Policy | Resource Hierarchy + Org Policies|
| **Maximum Privilege Ceiling**      | Permissions Boundaries   | Azure RBAC Deny Assign   | IAM Deny Policies        |
|                                    |                          |                          |                          |
| **Virtual Network Perimeter**      | Virtual Private Cloud    | Virtual Network (VNet)   | VPC Network (Global)     |
| **Stateful Virtual Firewall**      | Security Groups (SGs)    | Network Security Groups  | Cloud Firewall Rules     |
| **Stateless Subnet Firewall**      | Network ACLs (NACLs)     | Network Security Groups  | N/A (Stateless rules via |
|                                    |                          | (Subnet-level association| tags / priority)         |
| **Private Backbone Connectivity**  | AWS PrivateLink (VPCE)   | Azure Private Endpoint   | Private Service Connect  |
| **Hub-and-Spoke Interconnect**     | AWS Transit Gateway      | Azure Virtual WAN        | Network Connectivity Ctr |
|                                    |                          |                          |                          |
| **Object Storage Service**         | Amazon S3                | Azure Blob Storage       | Google Cloud Storage(GCS)|
| **Storage Immutability (WORM)**    | S3 Object Lock           | Immutable Blob Storage   | GCS Bucket Lock (Retention)|
| **Public Access Kill-Switch**      | S3 Block Public Access   | Storage Account Public Deny| GCS Public Access Prevention|
|                                    |                          |                          |                          |
| **Key Management Service**         | AWS KMS                  | Azure Key Vault          | Google Cloud KMS         |
| **Dedicated Hardware HSM**         | AWS CloudHSM             | Azure Managed HSM        | Cloud HSM                |
| **Hardware Enclave Compute**       | AWS Nitro Enclaves       | Azure Confidential Computing| Confidential VMs (SEV)|
|                                    |                          |                          |                          |
| **Control Plane API Audit Log**    | AWS CloudTrail           | Azure Activity Log       | Cloud Audit Logs         |
| **Intelligent Threat Detection**   | AWS GuardDuty (Agentless)| Microsoft Defender for Cloud| Security Command Center  |
| **Security Posture & Compliance**  | AWS Security Hub         | Defender for Cloud (CSPM)| Security Command Center  |
|                                    |                          |                          |                          |
| **Web Application Firewall (WAF)** | AWS WAF                  | Azure WAF (Front Door/AppGW)| Google Cloud Armor    |
| **DDoS Protection Service**        | AWS Shield (Standard/Adv)| Azure DDoS Protection     | Cloud Armor DDoS Defense |
+------------------------------------+--------------------------+--------------------------+--------------------------+
```

---

## 3. Deep-Dive Domain Comparisons

### A. Identity & Governance Models

```
AWS RESOURCE HIERARCHY:
[ Management / Payer Account ] ---> [ Organizational Units (OUs) ] ---> [ Member Accounts ] ---> [ Resources ]
(Governed via Service Control Policies - SCPs)

AZURE RESOURCE HIERARCHY:
[ Root Management Group ] ---> [ Management Groups ] ---> [ Subscriptions ] ---> [ Resource Groups ] ---> [ Resources ]
(Governed via Azure Policy & Entra ID Tenant)

GCP RESOURCE HIERARCHY:
[ Organization Node ] ---> [ Folders ] ---> [ Projects ] ---> [ Resources ]
(Governed via Organization Policies; Projects are the fundamental billing & IAM boundary!)
```

#### Key Identity Differences:
- **AWS:** Multi-account architecture is the standard isolation model. IAM is account-scoped unless integrated with AWS Identity Center. Roles issue short-lived STS tokens.
- **Azure:** Centered around a single centralized **Microsoft Entra ID (Azure AD)** tenant. Access to Azure subscriptions is delegated via Azure RBAC; applications use **Managed Identities** eliminating secret management.
- **GCP:** All IAM permissions are bound to the hierarchical tree (Organization $\to$ Folder $\to$ Project $\to$ Resource). Machine identities are **Service Accounts** that generate short-lived OAuth 2.0 access tokens.

---

### B. Cloud Networking & Firewall Topology

```
+------------------------------------+--------------------------+--------------------------+--------------------------+
| Feature                            | AWS VPC                  | Azure VNet               | GCP VPC                  |
+------------------------------------+--------------------------+--------------------------+--------------------------+
| **Network Scope**                  | Regional                 | Regional                 | **Global (Spans regions!)**|
| **Subnet Scope**                   | **Single AZ**            | Regional (Spans AZs)     | Regional (Spans AZs)     |
| **Default Egress Rule**            | Blocked if no IGW/NAT    | **Permitted by default** | Permitted if Cloud NAT   |
| **Default Inter-Subnet Routing**   | Open within VPC          | Open within VNet         | Open within VPC          |
| **Firewall Statefulness**          | SGs = Stateful, NACLs = Stateless | NSGs = Stateful         | Cloud Firewall = Stateful|
+------------------------------------+--------------------------+--------------------------+--------------------------+
```

> **Critical Azure Gotcha:** By default, Azure virtual machines possess default outbound internet connectivity through a system-assigned public IP unless explicitly blocked via an Outbound Network Security Group (NSG) rule or NAT Gateway!

---

## 4. Key Differences Matrix: Threat Detection & Incident Response Telemetry

| Action / Investigation Step | AWS Equivalent | Azure Equivalent | GCP Equivalent |
| :--- | :--- | :--- | :--- |
| **Track who created an admin** | CloudTrail: `CreateUser` | Activity Log: `Write User` | Cloud Audit Logs: `admin.create` |
| **Track who read sensitive object**| CloudTrail S3 Data Events | Storage Analytics / Blob Logs | Cloud Audit: `Data Access` |
| **Identify crypto-mining host** | AWS GuardDuty finding | Microsoft Defender for Cloud | Security Command Center |
| **Isolate compromised VM** | Attach empty Security Group | Attach NSG with Deny All | Tag instance with quarantine network tag |
| **Acquire disk snapshot** | `ec2:CreateSnapshot` | `az snapshot create` | `gcloud compute disks snapshot` |

---

## 5. Cybersecurity Relevance, Threats & Multi-Cloud Pitfalls

### 1. Azure NSG Directional Rule Confusion
- **Risk:** In AWS Security Groups, return traffic for an allowed inbound connection is automatically tracked and allowed (stateful). In Azure, Network Security Groups (NSGs) are also stateful, but they feature separate Inbound and Outbound rule lists. Engineers sometimes mistakenly believe an Outbound rule is required to allow return traffic, leading to overly broad `0.0.0.0/0` outbound allow configurations that permit malware data exfiltration.

### 2. GCP Global VPC Blast Radius
- **Risk:** In AWS and Azure, a VPC/VNet is strictly scoped to a single geographic region. In Google Cloud Platform, a **VPC is global**; subnets reside in distinct regions, but they route to each other across Google's private global fiber network automatically.
- **Exploitation:** Compromising a VM in a development subnet in `us-central1` allows direct, un-routed network pivoting to production database subnets in `europe-west1` unless explicit Cloud Firewall rules prevent inter-subnet traffic.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"In enterprise multi-cloud environments, security principles translate directly across cloud providers even as terminology shifts. Identity governance organizes hierarchically: AWS uses Organizations with Service Control Policies, Azure uses Management Groups with Azure Policy, and GCP organizes projects under Folders and Organization Nodes. Machine identity relies on temporary credentials across all three: AWS IAM Roles via STS, Azure Managed Identities via Entra ID, and GCP Service Accounts via OAuth 2.0. In cloud networking, while AWS and Azure scope virtual networks regionally, GCP VPCs are uniquely global, requiring strict firewall tags to prevent inter-regional lateral movement. For posture governance and detection, enterprises unify AWS GuardDuty, Microsoft Defender for Cloud, and GCP Security Command Center into centralized Cloud-Native Application Protection Platforms (CNAPP) to maintain consistent security baselines."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Assuming subnets in AWS and GCP behave identically. *Correction:* An AWS subnet is physically bound to a single Availability Zone; a GCP or Azure subnet spans multiple AZs across an entire region.
- **Trap 2:** Confusing Azure App Registrations with Enterprise Applications. *Correction:* App Registration is the global definition/blueprint of an app; Enterprise Application (Service Principal) is the local instance and security identity within a specific Entra ID tenant.
- **Trap 3:** Expecting S3 data event logging to be free in AWS. *Correction:* S3 data events in CloudTrail are disabled by default and incur significant log ingestion fees; they should be enabled selectively on sensitive compliance buckets.

### Expected Follow-Up Questions
1. *How does Workload Identity Federation work across cloud providers (e.g., Azure VM accessing AWS S3)?*
   - By configuring OIDC trust between cloud providers. The Azure VM requests an Entra ID token, which it presents directly to AWS STS via an IAM Role trust policy (`sts:AssumeRoleWithWebIdentity`), obtaining temporary AWS credentials without storing any static AWS keys in Azure.
2. *What is the difference between AWS SCPs and Azure Policy?*
   - AWS SCPs only define guardrails that deny access (they cannot deploy resources or audit configurations); Azure Policy can enforce deny actions, audit compliance, and automatically remediate or deploy compliant resources (`DeployIfNotExists`).
