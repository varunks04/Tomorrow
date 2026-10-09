# Git Architecture, Internals, and the Object Model

## 1. Topic & Definition
**Git** is a distributed, content-addressable version control system designed by Linus Torvalds. Unlike traditional VCS systems (e.g., SVN, CVS) that track file-by-file delta diffs, Git treats data as a **Directed Acyclic Graph (DAG) of immutable cryptographic snapshots**.

At its core, Git is a simple key-value object store where every object is compressed via zlib and referenced by a cryptographic hash of its type, size, and content.

---

## 2. How It Works: The 4 Core Objects and Cryptographic Hashing

```
+---------------------------------------------------------------------------------------------------+
|                                 The Git Object Model Graph                                        |
+---------------------------------------------------------------------------------------------------+
  [ Commit Object ] (Hash: c7a19...)
    - tree: a49f1...
    - parent: 9b2d0...
    - author: Alice <alice@corp.com> 1775836800 +0000
    - committer: Alice <alice@corp.com> 1775836800 +0000
    - message: "Add authentication middleware"
           |
           v
  [ Tree Object: Root Dir ] (Hash: a49f1...)
    - 100644 blob 8a1b2...  README.md
    - 040000 tree b72c3...  src/
                              |
                              v
                   [ Tree Object: src/ ] (Hash: b72c3...)
                     - 100644 blob 3f9d4...  auth.py
                     - 100644 blob e12a5...  server.py
```

### The 4 Git Object Types
1. **Blob (Binary Large Object):** Stores raw file data without metadata (no filename, no permissions, no timestamps). Filenames are stored exclusively in Trees.
2. **Tree:** Represents directory listings. Maps human-readable filenames, file modes (e.g., `100644` standard file, `100755` executable), and sub-directories to the corresponding blob or sub-tree hashes.
3. **Commit:** A snapshot manifest linking to a top-level root Tree object, zero or more parent commit hashes, author/committer identity timestamps, and the commit message string.
4. **Annotated Tag:** A permanent pointer to a specific commit containing its own message, timestamp, and optional GPG digital signature.

### Object Hashing & Content-Addressable Storage
Git constructs object headers deterministically:
$$\text{Header} = \text{"<type> <size>\0"}$$
$$\text{Hash} = \text{SHA-1}(\text{Header} + \text{Payload})$$
- The 40-character hexadecimal string determines the exact file path on disk inside `.git/objects/`:
  - Hash: `e69de29bb2d1d6434b8b29ae775ad8c2e48c5391`
  - Stored at: `.git/objects/e6/9de29bb2d1d6434b8b29ae775ad8c2e48c5391` (zlib compressed).

---

## 3. The Three Areas and Plumbing Commands

```
+-------------------+       git add       +-------------------+      git commit     +-------------------+
| Working Directory | ------------------> |   Staging Area    | ------------------> |  Git Repository   |
| (Local Filesystem)| <------------------ |   (The Index)     | <------------------ |  (Object DB / DAG)|
+-------------------+      git checkout   +-------------------+      git reset      +-------------------+
```

### Diagnostic Plumbing Commands (Inspecting the Matrix)
```bash
# Calculate SHA-1 hash of a string without writing to repository
echo -n "test content" | git hash-object --stdin
# Output: d670460b4b4aece5915caf5c68d12f560a9fe3e4

# Inspect the type of an arbitrary Git object
git cat-file -t e69de29bb2d

# View the raw contents of an object (commit, tree, or blob)
git cat-file -p HEAD

# Inspect the contents of a tree object (shows directory structure and hashes)
git ls-tree HEAD

# Resolve the exact commit SHA that HEAD points to
git rev-parse HEAD
```

---

## 4. Key Differences Matrix: Porcelain vs Plumbing Commands

| Dimension | Porcelain Commands (High-Level) | Plumbing Commands (Low-Level Core) |
| :--- | :--- | :--- |
| **Target Audience** | Human software developers | Git internal scripts & tooling |
| **Command Examples** | `git add`, `git commit`, `git merge`, `git pull` | `git hash-object`, `git cat-file`, `git mktree`, `git commit-tree` |
| **Output Format** | Human-friendly styled output | Raw machine-readable hashes and tab-delimited streams |
| **Error Handling** | Interactive hints and warnings | Standard Unix exit codes and raw stderr |
| **Security Scripting**| Fragile for automation (output changes across versions) | **Standard for security tooling & CI pipelines** |

---

## 5. Cybersecurity Relevance, Threats & Attack Vectors

### 1. SHA-1 Collision Attacks (SHAttered Attack)
- **Mechanism:** In 2017, Google and CWI Amsterdam demonstrated the **SHAttered** attack, creating two distinct PDF documents with identical SHA-1 hashes ($2^{63.1}$ evaluations).
- **Security Impact:** In theory, an attacker could craft two files with identical hashes—one benign and one containing malicious backdoor logic. If accepted by Git, the repository would associate both with the same object hash.
- **Git Defense:** Git implemented hardened SHA-1 (SHA-1DC) detecting collision attacks, and is actively transitioning to **SHA-256** (`git init --object-format=sha256`).

### 2. Malicious Hooks Execution (`.git/hooks/`)
- **Mechanism:** Git repositories can define client-side shell hooks (`pre-commit`, `post-checkout`, `post-merge`) that execute arbitrary commands when developers run Git commands.
- **Attack Vector:** An attacker tricks a developer into cloning a repository containing malicious hooks. (CVE-2024-32002 and CVE-2024-32004 involved malicious submodules and symbolic links writing executable hooks into `.git/hooks/` during clone, leading to Remote Code Execution).
- **Defense:** Never execute hooks from untrusted clones; keep Git updated to modern patched versions.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Git is a content-addressable key-value object store modeling version history as an immutable Directed Acyclic Graph (DAG) of cryptographic snapshots. The object model consists of four core types: Blobs (storing raw file bytes), Trees (storing directory structures, filenames, and permissions), Commits (linking a root tree, parent commits, author metadata, and messages), and Annotated Tags. Work moves through three distinct zones: the Working Directory, the Staging Area (Index), and the Git Object Repository. Because object identities are cryptographically bound to their content and ancestry hashes, Git history is tamper-evident, ensuring that modifying a single byte in a historical commit alters all subsequent descendant commit hashes throughout the entire tree."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Believing blobs store filenames. *Correction:* Blobs contain only raw file content bytes; filenames and permissions reside exclusively in Tree objects.
- **Trap 2:** Confusing high-level (porcelain) commands with low-level (plumbing) commands. *Correction:* Security automation scripts should use plumbing commands (`git rev-parse`, `git cat-file`) to avoid output format parsing breakage.
- **Trap 3:** Assuming `git reset --hard` completely destroys commits immediately. *Correction:* Commits remain in `.git/objects/` as unreachable dangling objects until Garbage Collection (`git gc`) purges them after the `reflog` expires (default: 30–90 days).

### Expected Follow-Up Questions
1. *What is the difference between a lightweight tag and an annotated tag?*
   - A lightweight tag is simply a pointer (a file under `.git/refs/tags/` containing a commit SHA); an annotated tag is a distinct Git object in the database containing its own author, date, message, and cryptographic GPG signature.
2. *What is `HEAD` in Git?*
   - `HEAD` is a reference file (`.git/HEAD`) pointing to the currently active branch reference (e.g., `ref: refs/heads/main`), or pointing directly to a specific commit SHA when in a "detached HEAD" state.
