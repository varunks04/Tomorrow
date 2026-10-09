# SQL Queries, Execution Order, Aggregates & Joins

> **Domain:** Core Computer Science  
> **Sub-Domain:** DBMS & SQL  
> **Interview Importance:** Very High / Mandatory Technical Assessment Area  

---

## 1. Topic & Definitions

- **Structured Query Language (SQL):** The domain-specific declarative language used to define, query, manipulate, and control relational databases.
- **Classification of SQL Commands:**
  - **DDL (Data Definition Language):** Defines or modifies database structure (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME`). Auto-commits in most engines.
  - **DML (Data Manipulation Language):** Modifies data records (`INSERT`, `UPDATE`, `DELETE`).
  - **DQL (Data Query Language):** Retrieves data from tables (`SELECT`).
  - **DCL (Data Control Language):** Manages access privileges and security permissions (`GRANT`, `REVOKE`).
  - **TCL (Transaction Control Language):** Manages transactions within the database (`COMMIT`, `ROLLBACK`, `SAVEPOINT`).
- **Logical Query Processing Order:** SQL queries are not executed in the written syntactic order. The engine evaluates clauses in a strict logical sequence.

---

## 2. Logical Query Execution Order

When you write a SQL query, the database optimizer executes clauses in the following strict order:

```text
Syntactic Order (How you write it)       Logical Execution Order (How the engine runs it)
──────────────────────────────────       ────────────────────────────────────────────────
1. SELECT                                1. FROM & JOINs (Identify target tables)
2. FROM & JOIN                           2. WHERE (Filter individual rows)
3. WHERE                                 3. GROUP BY (Aggregate rows into buckets)
4. GROUP BY                              4. HAVING (Filter aggregated group buckets)
5. HAVING                                5. SELECT (Evaluate expressions & projections)
6. ORDER BY                              6. DISTINCT (Eliminate duplicates)
7. LIMIT / OFFSET                        7. ORDER BY (Sort final result set)
                                         8. LIMIT / OFFSET (Slice final output window)
```

> 💡 **Key Takeaway:** You **cannot** reference a column alias created in `SELECT` inside a `WHERE` clause because `WHERE` executes *before* `SELECT`! However, you *can* use it in `ORDER BY` because `ORDER BY` executes *after* `SELECT`.

---

## 3. SQL Joins Illustrated with Worked Tables

Consider two sample tables:

```text
Table: Users                            Table: Roles
+----+----------+--------+              +--------+---------------+
| id | username | role_id|              | role_id| role_name     |
+----+----------+--------+              +--------+---------------+
| 1  | Alice    | 10     |              | 10     | Administrator |
| 2  | Bob      | 20     |              | 20     | Analyst       |
| 3  | Charlie  | NULL   |              | 30     | Auditor       |
+----+----------+--------+              +--------+---------------+
```

### 1. INNER JOIN
Returns only rows where there is a matching key in **both** tables.
```sql
SELECT u.username, r.role_name
FROM Users u
INNER JOIN Roles r ON u.role_id = r.role_id;
```
**Result:**
| username | role_name |
| :--- | :--- |
| Alice | Administrator |
| Bob | Analyst |
*(Charlie omitted because `role_id` is NULL; Auditor omitted because no user has `role_id` 30).*

---

### 2. LEFT (OUTER) JOIN
Returns **all** rows from the left table (`Users`), plus matched rows from the right table (`Roles`). If no match, right columns contain `NULL`.
```sql
SELECT u.username, r.role_name
FROM Users u
LEFT JOIN Roles r ON u.role_id = r.role_id;
```
**Result:**
| username | role_name |
| :--- | :--- |
| Alice | Administrator |
| Bob | Analyst |
| Charlie | *NULL* |

---

### 3. RIGHT (OUTER) JOIN
Returns **all** rows from the right table (`Roles`), plus matched rows from the left table (`Users`). If no match, left columns contain `NULL`.
```sql
SELECT u.username, r.role_name
FROM Users u
RIGHT JOIN Roles r ON u.role_id = r.role_id;
```
**Result:**
| username | role_name |
| :--- | :--- |
| Alice | Administrator |
| Bob | Analyst |
| *NULL* | Auditor |

---

### 4. FULL OUTER JOIN
Returns all rows when there is a match in either left or right table. Unmatched columns contain `NULL`.
```sql
SELECT u.username, r.role_name
FROM Users u
FULL OUTER JOIN Roles r ON u.role_id = r.role_id;
```
**Result:**
| username | role_name |
| :--- | :--- |
| Alice | Administrator |
| Bob | Analyst |
| Charlie | *NULL* |
| *NULL* | Auditor |

---

### 5. CROSS JOIN (Cartesian Product)
Returns every combination of rows from the first table with every row from the second table ($N \times M$ rows).
```sql
SELECT u.username, r.role_name FROM Users u CROSS JOIN Roles r;
-- Produces 3 x 3 = 9 rows total!
```

---

### 6. SELF JOIN
A table joined with itself to model hierarchical relationships (e.g., employee-to-manager).
```sql
SELECT e.name AS Employee, m.name AS Manager
FROM Employees e
LEFT JOIN Employees m ON e.manager_id = m.id;
```

---

## 4. Aggregations: `WHERE` vs. `HAVING`

| Feature | `WHERE` Clause | `HAVING` Clause |
| :--- | :--- | :--- |
| **Execution Point** | Before `GROUP BY`. | After `GROUP BY`. |
| **Filter Target** | Individual rows prior to grouping. | Aggregate groups created by `GROUP BY`. |
| **Aggregate Functions**| **Cannot** use aggregate functions (`SUM`, `COUNT`, `AVG`). | **Can** use aggregate functions. |
| **Example Query** | `WHERE salary > 50000` | `HAVING COUNT(user_id) > 5` |

```sql
-- Practical Example: Find departments with more than 3 high-earning security engineers
SELECT department, COUNT(id) AS high_earners
FROM Employees
WHERE salary > 100000            -- Row filter: only count salaries > 100k
GROUP BY department              -- Group remaining rows by department
HAVING COUNT(id) >= 3            -- Aggregate filter: only keep departments with >= 3 engineers
ORDER BY high_earners DESC;
```

---

## 5. Key Differences: `DELETE` vs. `TRUNCATE` vs. `DROP`

| Feature | `DELETE` | `TRUNCATE` | `DROP` |
| :--- | :--- | :--- | :--- |
| **Category** | DML | DDL | DDL |
| **Operation** | Deletes specific rows matching `WHERE`. | Deletes all rows in table at once. | Deletes entire table schema and data. |
| **Speed** | Slower (logs row-by-row deletions). | Extremely fast (deallocates pages).| Instantaneous. |
| **Rollback Capability**| Fully rollback-able inside transaction. | Cannot rollback in several engines. | Cannot rollback in several engines. |
| **Triggers** | Activates `ON DELETE` triggers. | Does **NOT** activate delete triggers. | Does not fire triggers. |

---

## 6. Cybersecurity Relevance & Threat Vectors

### 1. Forensics: `DELETE` vs. `TRUNCATE` Log Evaporation
- Incident responders rely on database Transaction Logs (WAL / Redo Logs) to reconstruct compromised data.
- When an attacker runs `DELETE FROM audit_logs`, the operation writes individual delete records to the transaction log, allowing forensic tools to recover deleted rows.
- If an attacker runs `TRUNCATE TABLE audit_logs`, the database simply deallocates data pages without logging individual row records, frustrating forensic reconstruction.

### 2. Blind SQL Injection using Correlated Subqueries
- Attackers exploit subqueries to extract data bit-by-bit when no database output is displayed on screen:
  ```sql
  -- Attacker injects a subquery to extract admin password hash
  SELECT * FROM products WHERE id = 1 AND (SELECT SUBSTRING(password, 1, 1) FROM users WHERE username='admin') = 'a';
  ```
- If the application returns "Product Found", the attacker knows the first character is 'a'.

---

## 7. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"SQL is categorized into DDL for schema definitions, DML for data manipulation, DQL for querying, DCL for access control permissions, and TCL for transactions. The engine processes queries logically starting with `FROM` and `JOIN`, then filtering rows with `WHERE`, grouping via `GROUP BY`, filtering groups with `HAVING`, projecting columns in `SELECT`, and finally sorting with `ORDER BY`. Joins combine tables: `INNER JOIN` returns mutual matches, `LEFT JOIN` retains all left rows with NULL right matches, and `CROSS JOIN` computes Cartesian products. In security operations, understanding DCL least privilege (`GRANT/REVOKE`) limits attacker blast radius, and knowing that `DELETE` logs rows while `TRUNCATE` deallocates pages is crucial for digital forensics."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Using `WHERE` to filter aggregate results like `WHERE COUNT(*) > 10`.  
  *Correction:* The compiler will reject this with a syntax error. Aggregate calculations must be filtered in the `HAVING` clause.
- **Trap:** Forgetting that `TRUNCATE` resets identity auto-increment counters, while `DELETE` preserves the counter.
