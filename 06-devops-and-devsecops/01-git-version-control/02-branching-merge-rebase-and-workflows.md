# Branching Strategies, Merge vs. Rebase, and Git Workflows

## 1. Topic & Definition
In Git, a **Branch** is not a heavy container or physical copy of the codebase; it is simply a lightweight, mutable 41-byte pointer (a text file in `.git/refs/heads/<branch-name>`) containing the 40-character hexadecimal SHA-1 commit hash of the latest commit on that line of development.

Integrating disparate lines of development requires either **Merging** (combining histories via explicit merge commits) or **Rebasing** (rewriting commit ancestry to achieve a linear history). The choice between them directly affects code review clarity, auditability, and SOC incident forensics.

---

## 2. How It Works: Merge vs. Rebase Mechanics

### A. Merge Mechanics: Fast-Forward vs Three-Way Merge

```
Scenario: Main has commit A-B. Feature has B-C-D.
FAST-FORWARD MERGE (No new commits on Main):
Main:    A --- B
                \
Feature:         C --- D  ===> git merge feature ===> Main: A --- B --- C --- D (HEAD moves forward)

Scenario: Main has diverged (A-B-E). Feature has (B-C-D). Common Ancestor = B.
THREE-WAY MERGE (Creates new Merge Commit M with 2 parents):
Main:    A --- B ------- E ---------- M (HEAD)
                \                   /
Feature:         C --------- D ----/ (Parents of M: E and D)
```

1. **Fast-Forward Merge:** If the target branch has not diverged since the feature branch was created, Git simply slides the target branch pointer forward to match the feature branch. No new commit object is created.
2. **Three-Way Merge (`git merge --no-ff`):** When branches have diverged, Git locates the **Best Common Ancestor (BCA)** commit, analyzes the diffs between the BCA and both branch tips, and creates a brand-new **Merge Commit** containing **two parent hashes** (`Parent 1: E`, `Parent 2: D`).

---

### B. Rebase Mechanics: Rewriting Ancestry

```
DIVERGED STATE:
Main:    A --- B --- E
                \
Feature:         C --- D

EXECUTE: git checkout feature && git rebase main
Step 1: Stores commits C and D into temporary patch files.
Step 2: Resets feature branch pointer to commit E (tip of main).
Step 3: Re-applies patch C on top of E -> creates C' (brand-new SHA!).
Step 4: Re-applies patch D on top of C' -> creates D' (brand-new SHA!).

REBASED RESULT (Linear History):
Main:    A --- B --- E
                      \
Feature:               C' --- D' (Completely new cryptographic commit hashes!)
```

> **The Golden Rule of Rebasing:** **NEVER rebase commits that have been pushed to a public, shared repository.** Because rebasing generates brand-new commit hashes, colleagues basing work on original commits `C` and `D` will face corrupted git trees and painful reconciliation loops.

---

## 3. Practical Branching Workflows

```
+------------------------------------+---------------------------------------------------------------+
| Workflow                           | Architectural Characteristics & Security Suitability          |
+------------------------------------+---------------------------------------------------------------+
| **Trunk-Based Development**        | All developers commit short-lived branches directly into main.|
| (Modern Cloud / DevSecOps)         | Employs feature flags; pairs with automated CI/CD scans.     |
+------------------------------------+---------------------------------------------------------------+
| **GitHub Flow**                    | Simple: Main branch is always deployable; feature branches    |
| (Agile SaaS standard)              | merged strictly via Peer-Reviewed Pull Requests.              |
+------------------------------------+---------------------------------------------------------------+
| **GitFlow**                        | Complex: Long-lived `main`, `develop`, `release/*`, `hotfix/*`|
| (Legacy Enterprise / Regulated)    | Strict release gates; heavy branching overhead.               |
+------------------------------------+---------------------------------------------------------------+
```

---

## 4. Key Differences Matrix: Git Merge vs Git Rebase

| Dimension | `git merge` | `git rebase` |
| :--- | :--- | :--- |
| **History Structure** | Non-linear DAG with branching forks and join commits | **Strictly linear, clean chronological line** |
| **Commit Integrity** | Original commit hashes are 100% preserved | **Rewrites history** (creates brand-new commit SHAs) |
| **Traceability** | Preserves exact chronological context and merge events | Hides historical merge timing; flattens timeline |
| **Conflict Resolution**| Resolved once inside the single merge commit | May require resolving conflicts step-by-step per commit |
| **SOC Forensics** | **Superior for audit trails** (identifies merge approvals)| Cleaner to read; harder to verify original commit timestamps |

---

## 5. Cybersecurity Relevance, Threats & Audit Compliance

### 1. The Audit Compliance Dilemma: History Rewriting
- **Regulatory Challenge:** Under compliance frameworks (SOC 2 Type II, ISO 27001, PCI-DSS), companies must prove that deployed production code matches peer-reviewed, authorized pull requests.
- **Risk of Unchecked Rebasing:** An engineer who force-pushes (`git push --force`) a rebased branch overwrites historical commit hashes, destroying the audit link between code reviews and deployed binaries.
- **Enterprise Control:** Enforce **Branch Protection Rules** on production branches (`main`, `release/*`):
  - Prohibit force-pushes (`--force` / `--force-with-lease`).
  - Require minimum 2 approving peer reviews.
  - Require linear history via squash-and-merge or rebase-and-merge controlled exclusively by the GitHub/GitLab server.

### 2. Malicious Code Injection via Unprotected Fast-Forward
- **Mechanism:** If branch protection allows fast-forward merges without requiring Pull Request reviews, an attacker with repository write access pushes backdoor commits directly to the remote repository without triggering peer review workflows.
- **Defense:** Enforce `require pull request reviews before merging` and mandatory status checks (passing CI SAST tests) before merge completion.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Git branching models are governed by pointers to commit nodes in the Directed Acyclic Graph. When integrating changes, `git merge` preserves the historical truth by joining divergent lines with a multi-parent merge commit, maintaining full auditability of when and how code was integrated. In contrast, `git rebase` replays commits onto a new base commit, creating brand-new commit hashes to achieve a clean, linear history. Modern DevSecOps teams favor Trunk-Based Development with GitHub Flow, utilizing protected branches where developers open short-lived feature branches, pass automated CI security quality gates, and require mandatory peer reviews before merging into `main`."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Rebasing a shared, public branch. *Correction:* Never rebase commits that others are building on; rebasing creates new commit hashes, corrupting your team's Git trees.
- **Trap 2:** Assuming Git branches are folders or copies of files. *Correction:* A branch is merely a 41-byte text file in `.git/refs/heads/` storing a 40-character commit SHA.
- **Trap 3:** Using `git push --force` blindly. *Correction:* Use `--force-with-lease` instead, which halts the push if someone else has updated the remote branch since your last fetch, preventing accidental code destruction.

### Expected Follow-Up Questions
1. *What is `git push --force-with-lease` and why is it safer than `git push --force`?*
   - `--force` blindly overwrites the remote branch regardless of its state. `--force-with-lease` checks if the remote branch reference matches your local tracking reference; if another developer pushed commits in the interim, the push is safely rejected.
2. *What is an interactive rebase (`git rebase -i`)?*
   - A tool allowing developers to clean up local commit history before creating a pull request: squashing messy "WIP" commits into logical units, rewording commit messages, reordering commits, or deleting unwanted commits.
