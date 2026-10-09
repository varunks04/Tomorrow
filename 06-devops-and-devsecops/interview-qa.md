# DevOps & Cloud Security — Technical Interview Q&A

### Q1: What is the Cloud Shared Responsibility Model?
> **Model Answer:**
> Security duties are split between the Cloud Service Provider (CSP) and the customer:
> - **Security OF the Cloud (Provider):** Physical data centers, host hardware, hypervisor, physical network facilities.
> - **Security IN the Cloud (Customer):** Customer data, IAM roles, network access configurations (Security Groups/firewalls), operating system patches (on IaaS), and application code.
> *In IaaS, customer controls from OS upward; in PaaS, customer controls code and data; in SaaS, customer controls only identity, access, and configuration.*

---

### Q2: How do Security Groups differ from Network ACLs (NACLs) in AWS?
> **Model Answer:**
> - **Security Groups:** Operate at the **instance/ENI level**. They are **stateful** (if inbound traffic is allowed, corresponding outbound response is automatically permitted). Rules are allow-only (cannot create explicit deny).
> - **Network ACLs (NACLs):** Operate at the **subnet boundary**. They are **stateless** (inbound and outbound rules must be explicitly configured separately). Rules are evaluated in numerical order and support explicit allow and deny rules.

---

### Q3: What is the risk of running Docker containers as `root`?
> **Model Answer:**
> By default, `root` (UID 0) inside a container maps directly to `root` (UID 0) on the host kernel unless user namespaces (`userns-remap`) are enabled.
> If an attacker achieves a container breakout (e.g., via kernel exploit or mounting the host Docker socket `/var/run/docker.sock`), they gain instant root access over the host operating system.  
> *Mitigation:* Specify `USER <non-root-uid>` in the Dockerfile, drop capabilities (`--cap-drop=ALL`), and enforce read-only root filesystems.
