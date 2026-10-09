# Cloud Service Models (IaaS, PaaS, SaaS, FaaS) and the Shared Responsibility Model

## 1. Topic & Definition
Cloud computing delivers on-demand computing resources over the internet on a pay-as-you-go basis.

The cornerstone of all cloud security governance is the **Shared Responsibility Model**. A cloud customer cannot outsource accountability: **the Cloud Service Provider (CSP) is responsible for the security *of* the cloud, while the Customer is responsible for security *in* the cloud**.

---

## 2. The Cloud Service Models Spectrum

```
+---------------------------------------------------------------------------------------------------+
|                               The Cloud Service Models Continuum                                  |
+---------------------------------------------------------------------------------------------------+
  IaaS (Infrastructure as a Service)  --> AWS EC2, Azure VMs, GCP Compute Engine
  PaaS (Platform as a Service)        --> AWS Elastic Beanstalk, Azure App Service, Google App Engine
  FaaS (Function as a Service / Serverless) --> AWS Lambda, Azure Functions, Google Cloud Functions
  SaaS (Software as a Service)        --> Microsoft 365, Salesforce, Google Workspace, ServiceNow
```

---

## 3. The Shared Responsibility Breakdown Matrix

The division of responsibility shifts dramatically based on the chosen cloud service abstraction layer:

```
+------------------------------------+-----------+-----------+-----------+-----------+-----------+
| Layer / Architectural Component    | On-Prem   | IaaS      | PaaS      | FaaS      | SaaS      |
+------------------------------------+-----------+-----------+-----------+-----------+-----------+
| **Data & Information Governance**  | CUSTOMER  | CUSTOMER  | CUSTOMER  | CUSTOMER  | CUSTOMER  |
| **Identity & Access Mgmt (IAM)**   | CUSTOMER  | CUSTOMER  | CUSTOMER  | CUSTOMER  | CUSTOMER  |
| **Application Logic & Code**       | CUSTOMER  | CUSTOMER  | CUSTOMER  | CUSTOMER  | CSP       |
| **Runtime Environment**            | CUSTOMER  | CUSTOMER  | CSP       | CSP       | CSP       |
| **Operating System & Patching**    | CUSTOMER  | CUSTOMER  | CSP       | CSP       | CSP       |
| **Network Controls / Firewall**    | CUSTOMER  | CUSTOMER  | Shared    | CSP       | CSP       |
| **Virtualization / Hypervisor**    | CUSTOMER  | CSP       | CSP       | CSP       | CSP       |
| **Physical Hardware & Servers**    | CUSTOMER  | CSP       | CSP       | CSP       | CSP       |
| **Physical Datacenter Security**   | CUSTOMER  | CSP       | CSP       | CSP       | CSP       |
+------------------------------------+-----------+-----------+-----------+-----------+-----------+
```

### The Non-Negotiable Axiom: The Customer Always Owns Data & IAM
Regardless of the model chosen—even in full SaaS solutions like Microsoft 365 or Salesforce:
- **The Customer is ALWAYS 100% responsible for:**
  1. **Customer Data:** Classifying, encrypting, and governing access to confidential corporate files.
  2. **Identity & Access Management (IAM):** Enforcing strong passwords, requiring MFA, and revoking accounts for departing employees.
  3. **Endpoint Device Hygiene:** Ensuring malware-free laptops connect to cloud portals.

---

## 4. Operational Boundaries: IaaS vs PaaS vs SaaS vs FaaS

### 1. Infrastructure as a Service (IaaS)
- **CSP Role:** Manages physical facilities, physical power/cooling, host hardware, and virtualization hypervisors (e.g., AWS Nitro).
- **Customer Role:** Full responsibility from the Operating System upward. You must configure OS security baselines, install Linux kernel security patches, configure host firewalls (iptables), patch libraries, and harden the network perimeter.

### 2. Platform as a Service (PaaS)
- **CSP Role:** Manages the OS, runtime execution engines, automatic OS patching, and hardware scaling.
- **Customer Role:** Focuses purely on application code, database configuration, and API access boundaries.

### 3. Function as a Service (FaaS / Serverless)
- **CSP Role:** Manages ephemeral container lifecycles, runtime bootstrap, OS patching, and hardware execution.
- **Customer Role:** Securing application code, third-party libraries bundled into the zip package, and restricting the execution IAM role attached to the function.

### 4. Software as a Service (SaaS)
- **CSP Role:** Manages the entire technology stack (hardware, OS, database, application code, availability, disaster recovery).
- **Customer Role:** Managing user accounts, role assignments, tenant configurations, data sharing settings, and enabling MFA.

---

## 5. Cybersecurity Relevance, Threats & Compliance Pitfalls

### 1. The "Cloud Means Secure" Fallacy
- **The Critical Mistake:** Organizations migrate from on-premises to AWS IaaS (EC2) and assume AWS will automatically patch their Windows and Linux servers.
- **The Real-World Consequence:** Capital One (2019 breach) was compromised because an unpatched SSRF vulnerability in an open-source WAF running on an EC2 instance allowed attackers to query the AWS Instance Metadata Service (IMDS), exfiltrating IAM credentials that compromised 100 million customer records.
- **The Reality:** AWS provides a secure hypervisor; securing the OS, software, and IAM configurations running inside the EC2 instance is **100% the customer's responsibility**.

### 2. SaaS Tenant Misconfiguration & Blind Sharing
- **Attack Vector:** An employee creates a public sharing link (`"Anyone with the link can view"`) to a sensitive corporate OneDrive folder containing financial forecasts.
- **The Responsibility Reality:** Microsoft's infrastructure was not breached; the customer failed to configure SaaS tenant sharing policies to restrict external link creation.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"The Shared Responsibility Model is the foundational governance framework of cloud security, dividing obligations between the Cloud Service Provider and the Customer. The CSP is responsible for security 'of' the cloud—physical datacenters, server hardware, power, and virtualization hypervisors. The Customer is responsible for security 'in' the cloud—their data, access control, and configurations. As organizations transition from IaaS to PaaS, FaaS, and SaaS, the CSP absorbs more operational responsibilities like OS patching and runtime management. However, regardless of the service model, the customer always retains non-negotiable responsibility for their data classification, encryption, and Identity and Access Management (IAM)."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Believing the CSP patches operating systems in IaaS (e.g., EC2 instances). *Correction:* In IaaS, OS patching is entirely the customer's responsibility; CSPs only patch operating systems in PaaS and SaaS.
- **Trap 2:** Assuming SaaS requires zero security engineering from the customer. *Correction:* Customers must still enforce MFA, configure tenant sharing policies, audit user permissions, and monitor for credential compromises in SaaS.
- **Trap 3:** Confusing PaaS with Serverless (FaaS). *Correction:* PaaS provides persistent running application platforms (e.g., App Service); FaaS runs short-lived, event-driven ephemeral code executions billed per millisecond.

### Expected Follow-Up Questions
1. *What was the root cause of the Capital One breach in relation to the Shared Responsibility Model?*
   - An SSRF vulnerability in an application running on customer-managed IaaS (EC2) allowed an attacker to query the metadata service and steal IAM role credentials. AWS secured the underlying cloud; Capital One was responsible for the application vulnerability and overly permissive IAM S3 bucket role.
2. *What is AWS Nitro and how does it affect hypervisor security?*
   - AWS Nitro offloads virtualization, networking, and storage processing from the host CPU onto dedicated hardware ASICs, minimizing the attack surface and mathematically preventing any AWS operator from accessing customer EC2 memory.
