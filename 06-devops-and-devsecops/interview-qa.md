# DevOps, Cloud Security & DevSecOps — Technical Interview Q&A Compendium

## Git, Scripting & Pipeline Security

### Q1: Explain Git internals. What are the four core object types, and why is Git tamper-evident?
> **Model Answer:**
> - Git is an immutable, content-addressable key-value store modeling repository history as a Directed Acyclic Graph (DAG) of cryptographic snapshots.
> - The four core object types are:
>   1. **Blobs:** Stores raw file data without metadata or filenames.
>   2. **Trees:** Stores directory structures, associating filenames and file modes to blob and sub-tree hashes.
>   3. **Commits:** Links a root tree hash, parent commit hashes, author/committer timestamps, and commit messages.
>   4. **Annotated Tags:** Permanent pointers to commits with independent messages and GPG signatures.
> - Git is tamper-evident because each object's 40-character hex identity is calculated as $\text{SHA-1}(\text{"<type> <size>\0"} + \text{content})$. Modifying a single byte in any historical commit alters its SHA-1 hash, cascading changes through all descendant parent pointers in the DAG.

---

### Q2: Why is deleting a leaked secret file and committing again (`git rm && git commit`) insufficient? How do you remediate it?
> **Model Answer:**
> - Git never forgets; `git rm` merely removes the file from the working directory and the latest commit snapshot. The secret remains permanently stored inside historical commit blobs in the `.git/` object database.
> - **Mandatory 3-Step Remediation:**
>   1. **Revoke and Rotate Immediately:** Assume the credential was scraped within seconds of the push. Invalidate the key in AWS, GitHub, or the database first.
>   2. **Purge from Git Object History:** Use `git-filter-repo` (`git filter-repo --invert-paths --path secrets.env`) to rewrite the commit graph and completely remove the blob from all commits.
>   3. **Force-Push & Clear Reflogs:** Run `git push origin --force --all` and contact GitHub/GitLab support to purge cached pull request views and dangling reflogs.

---

### Q3: What is the security danger of using `subprocess.run(..., shell=True)` or `os.system()` in Python automation?
> **Model Answer:**
> - `shell=True` spawns an intermediate system shell interpreter (`/bin/sh -c`), passing the command string through the shell's command parser.
> - If any portion of the command includes untrusted user input, an attacker can inject shell metacharacters (`;`, `|`, `&&`, `` ` ``), executing arbitrary system commands (**Command Injection**).
> - **Secure Pattern:** Always use `shell=False` and pass arguments as an explicit array of strings: `subprocess.run(["ping", "-c", "1", host], shell=False)`. This directly invokes the Linux kernel `execve()` system call, which treats the input strictly as an opaque argument vector without interpreting command separators.

---

### Q4: What is Poisoned Pipeline Execution (PPE) in CI/CD?
> **Model Answer:**
> - Poisoned Pipeline Execution occurs when an untrusted contributor alters pipeline configuration files (e.g., `.github/workflows/ci.yml`) inside a Pull Request to execute malicious scripts during automated CI runner execution.
> - If the pipeline exposes repository secrets to workflows triggered by pull requests from forks, the attacker's script can extract and exfiltrate production cloud credentials or signing keys.
> - **Defenses:** Never grant access to production secrets on `pull_request` triggers; mandate approval for outside collaborator workflow runs; and run pull request workflows on ephemeral, unprivileged runners.

---

## Containers, Kubernetes & Workload Hardening

### Q5: How do Linux Namespaces and Cgroups create container isolation?
> **Model Answer:**
> - A container is a standard Linux process running directly on the host kernel, isolated by kernel primitives:
>   - **Namespaces restrict what a process can see (Visibility):** PID (Process IDs), NET (Network devices/ports), MNT (Mount points/filesystem), IPC (Inter-Process Communication), UTS (Hostnames), and USER (UID/GID mappings).
>   - **Cgroups restrict how much a process can use (Resource Limits):** Enforces hard limits on CPU bandwidth, memory consumption (triggering OOM killer on overflow), and maximum PIDs (`pids.max` to prevent fork bombs).
>   - **OverlayFS:** Stacks immutable read-only image layers (`lowerdir`) with an ephemeral read-write container layer (`upperdir`) via Copy-on-Write (CoW).

---

### Q6: What is Google Distroless, and why does it improve container security?
> **Model Answer:**
> - Distroless images contain *only* the application runtime (e.g., Python, Node.js, Go) and minimal dependencies; they contain **no operating system package manager (`apt`/`apk`), no shell (`/bin/sh`/`/bin/bash`), and no terminal utilities (`curl`, `nc`, `wget`)**.
> - **Security Value:** If an attacker discovers a Remote Code Execution (RCE) flaw in the application, they cannot spawn a reverse shell (`/bin/sh`), download secondary malware via `curl`, or install attack packages, drastically curtailing the attacker's post-exploitation capabilities.

---

### Q7: Explain Kubernetes Pod Security Standards (PSS) and the "Restricted" profile.
> **Model Answer:**
> - PSS defines three security levels enforced by the built-in Pod Security Admission (PSA) controller: Privileged, Baseline, and Restricted.
> - The **Restricted** standard represents the hardened production gold standard:
>   1. Mandates non-root execution (`USER <non-zero-uid>`).
>   2. Drops all Linux capabilities (`ALL`) with only minimal exceptions like `NET_BIND_SERVICE`.
>   3. Enforces an immutable read-only root filesystem (`readOnlyRootFilesystem: true`).
>   4. Prohibits privilege escalation (`allowPrivilegeEscalation: false`).
>   5. Restricts volume types to safe configMaps, secrets, and PVCs, blocking hostPath mounts.

---

### Q8: Why are Kubernetes Network Policies critical, and what is the "Default Deny All" pattern?
> **Model Answer:**
> - By default, the Kubernetes network is completely flat: every pod can communicate with every other pod across any node or namespace without restriction.
> - A **Default Deny All** Network Policy matches all pods in a namespace and declares empty ingress/egress lists, instantly isolating all pods from receiving or initiating network traffic.
> - Responders and architects then create explicit microservice allow rules (e.g., allow `frontend` pods to reach `backend-api` on port 8080). This stops lateral movement and blocks compromised pods from establishing outbound reverse shells.

---

## Cloud Security, IAM & Cryptography

### Q9: Explain the Cloud Shared Responsibility Model across IaaS, PaaS, and SaaS.
> **Model Answer:**
> - **Security OF the Cloud (Provider Responsibility):** Physical facilities, server hardware, power, hypervisor virtualization, and physical network infrastructure.
> - **Security IN the Cloud (Customer Responsibility):** Customer data, Identity and Access Management (IAM), access controls, and device hygiene.
> - **Service Model Variations:**
>   - **IaaS (EC2):** Customer manages everything from the Operating System upward (OS patches, host firewalls, apps).
>   - **PaaS (Elastic Beanstalk):** CSP patches the OS and runtime; customer manages application code and data.
>   - **SaaS (Microsoft 365):** CSP manages the entire stack; customer is responsible for user accounts, MFA, and sharing policies.
>   *Rule:* The customer ALWAYS retains 100% responsibility for their Data and IAM across every model!

---

### Q10: How does Envelope Encryption work, and why is it used in AWS KMS?
> **Model Answer:**
> - Envelope encryption uses a two-tiered key hierarchy:
>   1. The application calls KMS: `kms:GenerateDataKey(KeyId="Master-CMK")`.
>   2. KMS returns a **Plaintext Data Encryption Key (DEK)** and an **Encrypted DEK** (encrypted by the master key).
>   3. The application encrypts the large data file locally with the plaintext DEK using AES-256-GCM.
>   4. The application **immediately purges and zeroes the plaintext DEK from RAM**.
>   5. The application stores the ciphertext file and the encrypted DEK side-by-side in S3.
> - **Why it is used:** The Master Key never leaves the FIPS 140-2 Level 3 Hardware Security Module (HSM), while multi-gigabyte data files are encrypted locally at wire speed without sending large payloads across the KMS network API.

---

### Q11: What is the difference between AWS Security Groups and Network ACLs (NACLs)?
> **Model Answer:**
> - **Security Groups (SGs):** Operate at the **instance/virtual network interface (ENI)** level. They are **stateful** (inbound traffic allowed automatically permits outbound reply traffic). Rules are allow-only (implicit deny by default).
> - **Network ACLs (NACLs):** Operate at the **subnet boundary**. They are **stateless** (inbound and outbound rules must be explicitly created independently, including ephemeral return ports $1024–65535$). Rules are evaluated in strict numerical order and support both explicit Allow and Deny rules.

---

### Q12: How do you defend Amazon S3 buckets against Ransomware?
> **Model Answer:**
> - Deploy a multi-layered storage defense:
>   1. Enable **S3 Block Public Access** at the AWS Account level.
>   2. Enforce TLS 1.3 via `aws:SecureTransport: false` Deny condition in the bucket policy.
>   3. Enforce Customer Managed KMS key encryption (SSE-KMS).
>   4. Enable **S3 Object Versioning** to retain historical copies upon deletion.
>   5. Enable **S3 Object Lock in Compliance Mode (WORM)**: guarantees that no identity—including the AWS Account Root User—can delete or overwrite objects until the retention period expires, completely neutralizing cloud ransomware.
