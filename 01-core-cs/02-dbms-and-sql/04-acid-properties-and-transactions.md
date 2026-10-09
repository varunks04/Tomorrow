# ACID Properties, Transactions, Isolation Levels & Concurrency Anomalies

> **Domain:** Core Computer Science  
> **Sub-Domain:** DBMS & SQL  
> **Interview Importance:** Extremely High / Core Systems & Security Architecture Concept  

---

## 1. Topic & Definitions

- **Database Transaction:** A logical unit of work consisting of one or more SQL operations that are executed atomically against a database. Either all operations succeed and become permanent, or the entire transaction is rolled back with zero effect.
- **Transaction Lifecycle States:**
  ```text
               ┌───────────► PARTIALLY COMMITTED ───► COMMITTED
               │                     │
    ACTIVE ────┤                     ▼
               │                  FAILED ────────────► ABORTED (Rolled back)
               └─────────────────────┘
  ```

---

## 2. The ACID Properties Explained

```text
+─────────────────────────────────────────────────────────────────────────────+
|                               ACID PROPERTIES                               |
+─────────────────────────────────────────────────────────────────────────────+
|  A - ATOMICITY     | "All or Nothing." If any step fails, the entire        |
|                    | transaction rolls back. Guaranteed via Undo Logs/WAL.  |
|────────────────────┼────────────────────────────────────────────────────────|
|  C - CONSISTENCY   | "Invariants Preserved." Database transitions from one  |
|                    | valid state to another, respecting all constraints.    |
|────────────────────┼────────────────────────────────────────────────────────|
|  I - ISOLATION     | "Concurrency Independence." Concurrent transactions   |
|                    | cannot interfere or observe intermediate states.       |
|────────────────────┼────────────────────────────────────────────────────────|
|  D - DURABILITY    | "Permanent Persistence." Once committed, updates       |
|                    | survive power outages or crashes. Enforced via Redo.   |
+─────────────────────────────────────────────────────────────────────────────+
```

### How ACID is Enforced Internally
1. **Write-Ahead Logging (WAL):** Before any data page is modified in RAM or written to disk, the change is first written to an append-only log file on non-volatile storage. If the power cuts off mid-transaction, during recovery:
   - **Undo Log:** Rolls back changes from uncommitted transactions (**Atomicity**).
   - **Redo Log:** Replays changes from committed transactions that were not yet flushed from RAM to disk (**Durability**).
2. **MVCC (Multi-Version Concurrency Control):** Modern databases (PostgreSQL, MySQL InnoDB) implement MVCC rather than blocking table locks. Readers don't block writers, and writers don't block readers; each transaction sees a consistent point-in-time snapshot of the database.

---

## 3. Concurrency Anomalies (Read Phenomena)

When multiple transactions execute concurrently without strict isolation, three classic anomalies emerge:

### 1. Dirty Read
- **Anomaly:** Transaction $T_1$ modifies a row without committing. Transaction $T_2$ reads this uncommitted data. Then, $T_1$ rolls back. $T_2$ has now made business decisions based on "dirty" phantom data that never officially existed!
```text
T1: BEGIN; UPDATE accounts SET balance = balance - 100 WHERE id = 1; (Uncommitted)
T2: BEGIN; SELECT balance FROM accounts WHERE id = 1; (Reads deducted balance!)
T1: ROLLBACK; (Balance restored!)
T2: Proceeds with invalid state!
```

### 2. Non-Repeatable Read (Fuzzy Read)
- **Anomaly:** Transaction $T_1$ reads a row. Transaction $T_2$ updates that exact row and commits. Transaction $T_1$ re-reads the same row within the same transaction and gets a **different value**.

### 3. Phantom Read
- **Anomaly:** Transaction $T_1$ executes a range query (e.g., `SELECT * FROM users WHERE age > 30`), finding 5 rows. Transaction $T_2$ inserts a brand-new user with age 35 and commits. Transaction $T_1$ re-runs the range query and suddenly observes 6 rows (a "phantom" row appeared).

---

## 4. ANSI SQL Isolation Levels vs. Anomalies Prevented

Database engines define four isolation levels offering tradeoffs between concurrency performance and strict consistency:

| Isolation Level | Dirty Read Permitted? | Non-Repeatable Read Permitted? | Phantom Read Permitted? | Concurrency Throughput | Mechanism Used |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **Read Uncommitted** | ❌ **YES (Vulnerable)** | ❌ **YES** | ❌ **YES** | Maximum (No locks) | Raw reads |
| **Read Committed** *(Default in Postgres/SQL Server)* | ✅ **NO (Prevented)** | ❌ **YES** | ❌ **YES** | Very High | MVCC (Reads latest committed snapshot per statement) |
| **Repeatable Read** *(Default in MySQL InnoDB)* | ✅ **NO (Prevented)** | ✅ **NO (Prevented)** | ❌ **YES** *(MySQL uses Gap Locks to prevent)* | Moderate | MVCC (Reads snapshot frozen at start of transaction) |
| **Serializable** | ✅ **NO (Prevented)** | ✅ **NO (Prevented)** | ✅ **NO (Prevented)** | Lowest (Transactions queued or aborted on conflict) | Strict Two-Phase Locking (2PL) or SSI (Serializable Snapshot) |

---

## 5. Cybersecurity Relevance & Threat Vectors

### 1. Double-Spending and Balance Exploits (Race Conditions)
- Occurs when an e-commerce or financial application uses `Read Committed` isolation:
  ```text
  Attacker fires two simultaneous requests: "Withdraw $100"
  Thread 1: Checks balance ($100 >= $100) -> OK
  Thread 2: Checks balance ($100 >= $100) -> OK
  Thread 1: Withdraws $100, sets balance to $0
  Thread 2: Withdraws $100, sets balance to -$100 (or $0)
  Result: Attacker withdrew $200 with only $100 in their account!
  ```
- **Defense:** Use pessimistic locking (`SELECT ... FOR UPDATE`), transaction-level `SERIALIZABLE` isolation, or atomic SQL updates: `UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100;`.

### 2. Forensic Log Anti-Forensics via Aborted Transactions
- An attacker testing password credentials against a database-backed authentication system can deliberately invoke `ROLLBACK` at the end of failed attempts if the application's logging logic resides inside the same rolled-back database transaction, wiping out audit trails.
- **Defense:** Write security audit logs using an independent transaction with autonomous commit (`PRAGMA AUTONOMOUS_TRANSACTION` in Oracle or asynchronous SIEM event streaming).

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"ACID guarantees reliability in database transactions: Atomicity ensures all-or-nothing execution via undo logs; Consistency maintains schema constraints and invariants; Isolation prevents concurrent transactions from interfering via MVCC and locks; and Durability ensures committed updates persist through crashes using Write-Ahead Logging (WAL). SQL defines four isolation levels to combat concurrency anomalies: Read Uncommitted permits dirty reads; Read Committed stops dirty reads but permits non-repeatable reads; Repeatable Read prevents non-repeatable reads; and Serializable completely eliminates all anomalies including phantom reads by enforcing strict ordering. In security engineering, failing to use proper transaction isolation enables double-spending race conditions and coupon replay vulnerabilities."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Believing `Read Committed` is safe for financial deductions.  
  *Correction:* Under Read Committed, two concurrent threads can read the same positive balance before either commits a debit, leading to double spending. You must use explicit row-level locking (`SELECT ... FOR UPDATE`) or atomic SQL decrement statements.
- **Trap:** Confusing Non-Repeatable Read and Phantom Read.  
  *Correction:* Non-repeatable read involves **updating/modifying an existing row**. Phantom read involves **inserting new rows into a range query**.
