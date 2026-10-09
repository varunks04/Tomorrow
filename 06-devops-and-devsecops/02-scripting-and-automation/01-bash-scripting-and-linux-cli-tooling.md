# Bash Scripting, Linux Pipelines, and Text Processing in Security

## 1. Topic & Definition
**Bourne Again Shell (Bash)** is the primary command-line interpreter across Linux and Unix environments. In cybersecurity operations, Bash scripting and pipeline construction provide the bedrock for rapid Incident Response (IR) triage, log file extraction, automated forensics, and system hardening.

The Unix Philosophy—*"Write programs that do one thing and do it well, and write programs to work together"*—manifests via **Standard Streams and Pipes**, enabling security engineers to chain simple CLI primitives into high-throughput stream processing engines.

---

## 2. How It Works: Streams, File Descriptors, and Pipelines

```
+---------------------------------------------------------------------------------------------------+
|                              Linux Standard Streams and Pipelines                                 |
+---------------------------------------------------------------------------------------------------+
  File Descriptor 0: STDIN  (Standard Input  - Keyboard / Pipe In)
  File Descriptor 1: STDOUT (Standard Output - Terminal Screen / Redirect Out)
  File Descriptor 2: STDERR (Standard Error  - Terminal Screen / Diagnostic Out)

  PIPELINE FLOW:
  [ command1 ] --( stdout: FD 1 )--> | PIPE (|) | --( stdin: FD 0 )--> [ command2 ]
        |                                                                     |
        +--( stderr: FD 2 )--> /dev/null                                      +--( stdout: FD 1 )--> alert.log
```

### Stream Redirection Operators
- `>` : Overwrites stdout to a file (`echo "bad_ip" > blocklist.txt`).
- `>>` : Appends stdout to a file (`echo "bad_ip" >> blocklist.txt`).
- `2>` : Redirects stderr to a file (`command 2> error.log`).
- `2>&1` : Merges stderr into stdout stream (`command > output.log 2>&1`).
- `&>` : Shorthand for redirecting both stdout and stderr (`command &> all.log`).
- `<` : Reads stdin from a file (`mysql < dump.sql`).
- `|` : Chains stdout of the left command to stdin of the right command.

---

## 3. The Core Security Text Processing Toolkit

### 1. `grep` / `ripgrep` (Regular Expression Searching)
```bash
# Find all failed SSH login attempts in auth.log
grep "Failed password for" /var/log/auth.log

# Extract all IPv4 addresses using extended regex (-E)
grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" /var/log/nginx/access.log
```

### 2. `awk` (Field-Based Stream Processing)
`awk` processes structured log lines by splitting records into delimited fields (`$1, $2, ... $NF`):
```bash
# Print client IP ($1) and HTTP Status Code ($9) from Nginx web logs
awk '{print $1, $9}' /var/log/nginx/access.log

# Filter and alert on HTTP 500 Internal Server Errors
awk '$9 == 500 {print "CRITICAL: Server Error from IP: " $1}' /var/log/nginx/access.log
```

### 3. `sed` (Stream Editor for Substitution and Transformation)
```bash
# Redact sensitive IP subnets in logs before external sharing
sed -E 's/10\.[0-9]+\.[0-9]+\.[0-9]+/10.REDACTED/g' syslog.log

# Extract lines between specific timestamps
sed -n '/Oct  9 12:00:00/,/Oct  9 13:00:00/p' /var/log/syslog
```

### 4. `sort`, `uniq`, and `cut` (Aggregation Pipelines)
The classic **Top-10 Attacker IP Extraction Pipeline**:
```bash
# Extract source IPs, count occurrences, and sort descending
cat /var/log/nginx/access.log | cut -d ' ' -f 1 | sort | uniq -c | sort -nr | head -n 10
```

### 5. `xargs` (Parallel Argument Construction)
```bash
# Take suspicious IP list from stdin and query WHOIS in parallel (4 jobs)
cat bad_ips.txt | xargs -n 1 -P 4 whois
```

---

## 4. Defensive Bash: Hardening Enterprise Scripts

By default, Bash continues execution when commands fail, treats unset variables as empty strings, and masks pipeline failure codes. Production security automation scripts must enforce strict defensive execution headers:

```bash
#!/usr/bin/env bash
# DEFENSIVE BASH EXECUTION HEADER
set -euo pipefail
IFS=$'\n\t'
```

### Breakdown of Flags:
1. **`set -e` (Exit Immediately):** Halts script execution immediately if any command exits with a non-zero exit status ($? \ne 0$). Prevents a script from continuing into destructive operations when prerequisite steps fail.
2. **`set -u` (Treat Unset Variables as Errors):** Exits if an undefined variable is referenced. Prevents disastrous typos like `rm -rf "$TARGET_DIR/"` where an unset `$TARGET_DIR` results in executing `rm -rf /`!
3. **`set -o pipefail` (Pipeline Exit Code Preservation):** By default, a pipeline returns the exit status of the *last command*. In `command1 | command2`, if `command1` fails but `command2` succeeds, `$?` is 0. With `pipefail`, the pipeline fails if **any** command in the chain fails.
4. **`IFS=$'\n\t'` (Internal Field Separator Hardening):** Sets whitespace splitting strictly to newlines and tabs (excluding spaces), preventing word-splitting vulnerabilities on filenames with spaces.

---

## 5. Key Differences Matrix: Text Processing Tools

| Feature | `grep` / `egrep` | `sed` | `awk` | `cut` |
| :--- | :--- | :--- | :--- | :--- |
| **Primary Domain** | Pattern matching & line filtering | Stream transformation & regex replace | Full programming language; column ops | Delimiter-based field slicing |
| **Field Manipulation**| Primitive (Regex lookahead) | Difficult (Requires capture groups) | **Native & Powerful (`$1, $2, $NF`)** | Simple (`-d ' ' -f 1`) |
| **Mathematical Ops** | None | None | **Full arithmetic support (`sum += $5`)**| None |
| **Execution Speed** | Extremely Fast | Fast | Fast | Extremely Fast |
| **Cyber Use Case** | Triage: Find all instances of CVE | Sanitization: Redact sensitive tokens | Aggregation: Sum egress bytes per IP | Quick column isolation |

---

## 6. Cybersecurity Relevance, Threats & Attack Vectors

### 1. Command Injection via Unsanitized Variable Expansion
- **Vulnerability:** Passing unsanitized user-supplied input into `eval`, backticks (`` `cmd` ``), or unquoted variables.
- **Exploitation:**
  ```bash
  # INSECURE BASH SCRIPT:
  read -p "Enter hostname to ping: " TARGET_HOST
  ping -c 1 $TARGET_HOST
  ```
  Attacker enters: `127.0.0.1; cat /etc/shadow`
  Bash executes the ping, followed immediately by dumping the shadow password file!
- **Defense:** Never pass variables to `eval`; quote all variables (`"$TARGET_HOST"`); validate input against strict character whitelists using regex:
  ```bash
  if [[ ! "$TARGET_HOST" =~ ^[a-zA-Z0-9.-]+$ ]]; then
      echo "Invalid input!" && exit 1
  fi
  ```

### 2. Wildcard Injection in Unix Commands (Tar / Chmod Arbitrary Code Execution)
- **Mechanism:** When a script executes `tar -czf backup.tar *`, the shell expands `*` to all filenames in the directory.
- **Exploitation:** An attacker creates two filenames: `--checkpoint=1` and `--checkpoint-action=exec=sh shell.sh`. When `tar` expands `*`, it interprets the filenames as command-line argument flags, executing `shell.sh` with root permissions!
- **Defense:** Always use `--` to indicate end of command flags (`tar -czf backup.tar -- *`), or reference absolute paths (`./*`).

---

## 7. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Bash scripting and the Linux CLI provide the foundational toolkit for rapid security triage, forensic extraction, and incident response automation. Through standard streams and pipes, security engineers construct high-throughput processing chains combining `grep` for pattern filtering, `awk` for field extraction and mathematical metric aggregation, and `sort | uniq -c` for threat frequency profiling. To ensure operational safety and prevent catastrophic script errors, enterprise production scripts must enforce `set -euo pipefail` to halt on errors, flag unset variables, and preserve pipeline failure codes, alongside strict quoting and input validation to eliminate command injection."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Writing `cat file.txt | grep "pattern"`. *Correction:* Useless use of `cat`; `grep` takes file arguments directly (`grep "pattern" file.txt`), saving a redundant process invocation and pipe buffer.
- **Trap 2:** Omitting `set -o pipefail`. *Correction:* In `curl http://bad | bash`, if `curl` returns a 404 error page, without `pipefail` the pipeline exits with code 0 because `bash` successfully executed the empty input.
- **Trap 3:** Unquoted variable expansions (`$VAR` instead of `"$VAR"`). *Correction:* Unquoted variables cause word splitting and glob expansion, opening scripts to command injection and argument parsing bugs.

### Expected Follow-Up Questions
1. *What does the exit code 127 mean in Linux?*
   - "Command not found", indicating the requested binary is not in `$PATH` or has a typo in its path. Exit code 126 means "Command found but not executable" (permission denied).
2. *How do you extract the Top 5 IP addresses generating 404 errors from an Apache access log?*
   - `awk '$9 == 404 {print $1}' access.log | sort | uniq -c | sort -nr | head -n 5`
