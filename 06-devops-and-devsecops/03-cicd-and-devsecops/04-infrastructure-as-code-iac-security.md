# Infrastructure as Code (IaC) Security and Policy-as-Code (PaC)

## 1. Topic & Definition
**Infrastructure as Code (IaC)** is the architectural practice of provisioning, managing, and configuring cloud infrastructure (virtual machines, networks, databases, IAM policies) using machine-readable definition files (e.g., HashiCorp Terraform HCL, AWS CloudFormation, Pulumi, Kubernetes manifests) rather than manual interactive configuration ("ClickOps").

While IaC enables rapid scalability and immutable infrastructure, it introduces significant risk: **a single misconfigured line of code committed to an IaC repository can provision hundreds of publicly exposed cloud assets in seconds across multiple regions**.

**Policy-as-Code (PaC)** solves this by codifying organizational security, compliance, and governance rules into declarative policies that automatically validate IaC templates before infrastructure is ever provisioned.

---

## 2. How It Works: Pre-Deployment Shift-Left IaC Validation

```
+---------------------------------------------------------------------------------------------------+
|                            Shift-Left IaC Validation Lifecycle                                    |
+---------------------------------------------------------------------------------------------------+
  [ Terraform Code (.tf) ] ---> [ `terraform plan` Output ] ---> [ Plan JSON Export ]
                                                                        |
                                                                        v
  [ Pass: Terraform Apply ] <--- [ OPA Rego / Checkov Policy Engine ] <--- [ Corporate Security Policies]
                                        | (CIS AWS Benchmarks)
                                        v
                                 [ Violations Detected: FAIL BUILD! ]
                                 (e.g., S3 Bucket encryption = false)
```

### The Three Phases of IaC Security
1. **Static Pre-Commit Scanning:** Scanning raw `.tf` files in developer IDEs using linters and scanners (Checkov, tfsec).
2. **Pre-Deployment Plan Scanning:** Converting the compiled execution plan (`terraform show -json tfplan > plan.json`) into structured JSON and evaluating it against Policy-as-Code engines (Open Policy Agent) in CI/CD.
3. **Runtime Drift Detection:** Scheduled cloud audits comparing the running infrastructure state against the Git IaC repository to detect out-of-band manual modifications.

---

## 3. Policy-as-Code with Open Policy Agent (OPA) and Rego

**Open Policy Agent (OPA)** is an open-source, general-purpose policy engine that decouples policy decisions from application code. Policies are written in **Rego**, a declarative, query-based logic language.

### Example Enterprise Rego Security Policy: Block Public S3 Buckets
```rego
package terraform.security

# Deny deployment if any AWS S3 bucket has public-read ACL
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket"
    resource.change.after.acl == "public-read"
    msg := sprintf("SECURITY VIOLATION: S3 Bucket '%v' configured with public-read ACL!", [resource.name])
}

# Deny deployment if S3 server-side encryption is disabled
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket_server_side_encryption_configuration"
    count(resource.change.after.rule) == 0
    msg := sprintf("SECURITY VIOLATION: S3 Bucket '%v' lacks mandatory server-side encryption!", [resource.name])
}
```

---

## 4. Practical Common IaC Vulnerabilities and Remediation

### 1. Insecure Security Group: Open SSH / RDP
- **Insecure Terraform:**
  ```hcl
  resource "aws_security_group_rule" "ingress_ssh" {
    type        = "ingress"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # CRITICAL RISK: Open to entire internet!
  }
  ```
- **Remediation:** Restrict to corporate bastion VPN CIDR: `cidr_blocks = ["198.51.100.12/32"]`.

### 2. Over-Privileged IAM Role Policies
- **Insecure Terraform:**
  ```hcl
  resource "aws_iam_policy" "wildcard_admin" {
    policy = jsonencode({
      Version = "2012-10-17"
      Statement = [{
        Action   = "*"       # CRITICAL RISK: Full administrative control!
        Effect   = "Allow"
        Resource = "*"
      }]
    })
  }
  ```
- **Remediation:** Enforce Principle of Least Privilege (PoLP); restrict to specific actions and explicit resource ARNs.

---

## 5. Key Differences Matrix: IaC Security Tools

| Feature | Checkov | tfsec (Trivy) | Open Policy Agent (OPA) / Rego | HashiCorp Sentinel |
| :--- | :--- | :--- | :--- | :--- |
| **Primary Domain** | Static & Plan IaC Scanning | Fast CLI IaC Scanner | Universal Policy Engine (Kubernetes & Cloud) | Terraform Enterprise / Cloud Engine |
| **Rule Language** | Python / YAML custom policies | Go / Rego | **Rego** (Declarative Logic Language) | Sentinel (Proprietary language) |
| **Out-of-the-Box Rules**| 1000+ CIS Benchmark checks | Comprehensive CIS rules | Blank by default; requires policy repos | Pre-built policy sets |
| **License** | Open Source (Palo Alto) | Open Source (Aqua Security) | Open Source (CNCF Graduated) | Commercial (HashiCorp Enterprise) |
| **Pipeline Stage** | Pre-commit & CI Quality Gate | Local IDE & Early CI | CI Gate & Kubernetes Admission Control | Terraform Cloud Deployment Gate |

---

## 6. Cybersecurity Relevance, Threats & Attack Vectors

### 1. The Terraform State File Credential Minefield (`terraform.tfstate`)
- **Critical Risk:** Terraform stores metadata and resource properties in a state file. **Terraform state files store database passwords, TLS private keys, and cloud credentials in PLAINTEXT JSON by default**.
- **Exploitation:** An engineer commits `terraform.tfstate` to Git, or stores it in an unencrypted, public S3 bucket without access logging. An attacker reads the state file and extracts full root credentials for all newly provisioned databases.
- **Defense:**
  1. Add `*.tfstate` to `.gitignore`.
  2. Store state in encrypted remote backends (AWS S3 with KMS encryption and strict IAM bucket policies).
  3. Enforce S3 bucket versioning and DynamoDB state locking to prevent race conditions.

### 2. Infrastructure Drift & Out-of-Band Shadow IT
- **Problem:** A developer creates infrastructure via Terraform, but later uses the AWS Management Console to manually add an open `0.0.0.0/0` firewall rule during an emergency debugging session, forgetting to remove it.
- **Consequence:** The Git repository displays secure configuration, but production infrastructure is actively vulnerable (**Drift**).
- **Defense:** Automated scheduled drift detection pipelines running `terraform plan --detailed-exitcode` daily; automated SOAR playbooks that revert unmanaged manual changes.

---

## 7. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Infrastructure as Code (IaC) eliminates manual error and configuration drift by defining cloud resources through declarative templates like Terraform and CloudFormation. However, IaC also introduces the risk of scaling severe misconfigurations—such as public S3 buckets, open SSH security groups, and wildcard IAM policies—across thousands of cloud assets instantly. Securing IaC requires shifting security left: executing static analysis using tools like Checkov and enforcing declarative Policy-as-Code using Open Policy Agent (OPA) Rego against compiled Terraform execution plans. Crucially, organizations must secure the Terraform state file (`terraform.tfstate`), which contains sensitive credentials in cleartext JSON, by mandating KMS-encrypted remote backends with restricted IAM access policies."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Committing `terraform.tfstate` to Git. *Correction:* State files contain cleartext passwords and secrets; they must strictly be excluded via `.gitignore` and hosted in an encrypted S3/GCS remote backend.
- **Trap 2:** Scanning only static `.tf` files without scanning `terraform plan`. *Correction:* Static scans miss dynamic variables, computed modules, and runtime overrides; scanning the compiled JSON plan captures the exact final infrastructure state.
- **Trap 3:** Confusing IaC with Configuration Management. *Correction:* IaC (Terraform) focuses on provisioning cloud infrastructure resources; Configuration Management (Ansible, Puppet) focuses on configuring software and OS state inside provisioned virtual machines.

### Expected Follow-Up Questions
1. *What is "Drift" in IaC, and how is it detected?*
   - Drift occurs when infrastructure is modified out-of-band (e.g., via cloud console) so that actual cloud state diverges from the IaC code. It is detected by running `terraform plan`, which refreshes state against real cloud APIs and highlights discrepancies.
2. *Why is Open Policy Agent (OPA) preferred over hardcoded bash script checks?*
   - OPA uses a dedicated declarative logic language (Rego) that separates policy rules from implementation logic, allowing unified security policies to be evaluated across Terraform plans, Kubernetes admission controllers, and microservice APIs.
