# Database Normalization: 1NF, 2NF, 3NF, BCNF & Anomalies

> **Domain:** Core Computer Science  
> **Sub-Domain:** DBMS & SQL  
> **Interview Importance:** Very High / Standard Technical Round Design Question  

---

## 1. Topic & Definitions

- **Database Normalization:** A systematic schema design technique for organizing database tables to minimize **data redundancy** (duplication) and eliminate **modification anomalies** (insertion, deletion, and update anomalies).
- **Functional Dependency ($X \rightarrow Y$):** A relationship between attributes where the value of attribute set $X$ uniquely determines the value of attribute set $Y$. (e.g., `SSN -> FullName`).
- **Prime Attribute:** An attribute that is a member of any candidate key of the relation.
- **Non-Prime Attribute:** An attribute that does not belong to any candidate key.

---

## 2. The Three Database Anomalies (Why Normalization Matters)

Consider a flawed, unnormalized table combining Students, Courses, and Instructors:

```text
Table: FlawedEnrollment (StudentID, StudentName, CourseCode, CourseName, Instructor, OfficeRoom)
```

1. **Insertion Anomaly:** You cannot add a new course offering (e.g. `CS401: Advanced Cryptography`) to the database unless at least one student enrolls in it, because `StudentID` is part of the primary key and cannot be `NULL`.
2. **Deletion Anomaly:** If the only student enrolled in `CS401` drops the course and their record is deleted, the entire course information, course name, instructor, and office room are permanently lost from the database.
3. **Update / Modification Anomaly:** If instructor `Dr. Alice` changes her office room, you must update hundreds of individual student enrollment rows. If one row is missed, the database enters an inconsistent state.

---

## 3. Step-by-Step Walkthrough: Normal Forms

```text
+───────────────────────────────────────────────────────────────────────────+
| UNNORMALIZED FORM (UNF)                                                   |
|   ▼ (Eliminate repeating groups & multi-valued attributes)                |
| FIRST NORMAL FORM (1NF)                                                   |
|   ▼ (Eliminate Partial Dependencies: non-prime depends on partial PK)     |
| SECOND NORMAL FORM (2NF)                                                  |
|   ▼ (Eliminate Transitive Dependencies: non-prime depends on non-prime)   |
| THIRD NORMAL FORM (3NF)                                                   |
|   ▼ (Eliminate remaining anomalies: in every X -> Y, X must be Super Key) |
| BOYCE-CODD NORMAL FORM (BCNF)                                             |
+───────────────────────────────────────────────────────────────────────────+
```

### 1. First Normal Form (1NF)
- **Rules:**
  1. Each column must contain strictly **Atomic (indivisible) values** (no comma-separated lists, arrays, or repeating groups).
  2. Each column must contain values of the same data type.
  3. All columns must have unique names.
  4. Each record must be uniquely identifiable via a Primary Key.

```text
❌ Violation of 1NF:
StudentID | Name  | PhoneNumbers
----------+-------+--------------------------
101       | Alice | 555-0100, 555-0101 (Not atomic!)

✅ Normalized to 1NF:
StudentID | Name  | PhoneNumber
----------+-------+-------------
101       | Alice | 555-0100
101       | Alice | 555-0101
```

---

### 2. Second Normal Form (2NF)
- **Rules:**
  1. Table must be in **1NF**.
  2. **No Partial Dependencies:** Every non-prime attribute must be **fully functionally dependent** on the *entire* primary key, not a subset/part of it.
  > ⚠️ *Note: 2NF only applies to tables with Composite Primary Keys! If a table has a single-attribute primary key and is in 1NF, it is automatically in 2NF.*

```text
❌ Violation of 2NF:
Composite Primary Key: { StudentID, CourseID }
Columns: StudentID, CourseID, CourseFee, StudentGrade

• StudentGrade depends on { StudentID, CourseID } (Full Dependency - OK)
• CourseFee depends ONLY on CourseID (Partial Dependency - VIOLATION!)

✅ Normalized to 2NF (Split into two tables):
Table 1: Course (CourseID [PK], CourseFee)
Table 2: Enrollment (StudentID [FK], CourseID [FK], StudentGrade)
```

---

### 3. Third Normal Form (3NF)
- **Rules:**
  1. Table must be in **2NF**.
  2. **No Transitive Dependencies:** No non-prime attribute should be transitively dependent on the primary key via another non-prime attribute ($A \rightarrow B \rightarrow C$).
  > *Rule of thumb: "Every attribute must depend on the key, the whole key, and nothing but the key, so help me Codd."*

```text
❌ Violation of 3NF:
Table: Employee (EmpID [PK], EmpName, DepartmentID, DepartmentHead)
• EmpID -> DepartmentID
• DepartmentID -> DepartmentHead
• Therefore: EmpID -> DepartmentHead (Transitive Dependency - VIOLATION!)

✅ Normalized to 3NF (Split into two tables):
Table 1: Employee (EmpID [PK], EmpName, DepartmentID [FK])
Table 2: Department (DepartmentID [PK], DepartmentHead)
```

---

### 4. Boyce-Codd Normal Form (BCNF / 3.5NF)
- **Rule:** A stricter version of 3NF. For **every non-trivial functional dependency $X \rightarrow Y$**, the determinant $X$ **must be a Super Key**.
- Resolves anomalies where a candidate key is composite, multiple candidate keys overlap, and a non-prime attribute determines a prime attribute.

---

## 4. Normalization vs. Denormalization Tradeoff

| Dimension | Normalization (3NF / BCNF) | Denormalization |
| :--- | :--- | :--- |
| **Primary Goal** | Data integrity, eliminate duplication & anomalies. | High-speed read query performance. |
| **Write Performance** | **Fast:** Inserts, updates, and deletes touch only one normalized row. | **Slower:** Modifying a duplicated field requires multi-row updates. |
| **Read Performance** | Slower: Requires multi-table `JOIN` operations. | **Fast:** Pre-joined tables avoid expensive join latency. |
| **Storage Usage** | Minimal (no duplicate columns). | Higher (redundant columns across tables). |
| **Typical Use Case** | OLTP systems (Banking, E-commerce, IAM directory). | OLAP systems, Data Warehousing, Reporting engines. |

---

## 5. Cybersecurity Relevance & Threat Vectors

### 1. Inconsistent Security Access Rules via Denormalization
- If role permissions are denormalized across multiple user tables, revoking a user's admin access in `UserAccounts` while failing to update `UserSessions` or `AppPermissions` creates a **zombie administrative privilege** that an attacker can exploit.

### 2. Audit Trail Tampering Resistance
- Normalized schema structures guarantee that critical audit logs and access logs cannot have partial data wiped out without triggering referential integrity constraints, protecting forensics investigations.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Normalization is the structured process of designing relational schemas to eliminate insertion, deletion, and update anomalies while reducing data redundancy. 1NF enforces atomic, non-divisible column values and unique row identification. 2NF builds on 1NF by eliminating partial dependencies where a non-prime column depends on only part of a composite primary key. 3NF eliminates transitive dependencies where a non-prime column depends on another non-prime column. BCNF is a stricter version of 3NF requiring every determinant in a functional dependency to be a super key. In production systems, we normalize OLTP databases for write integrity and selectively denormalize OLAP analytics data stores for query read throughput."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Thinking every table needs 2NF checking.  
  *Correction:* If a table has a single-column primary key (not composite) and satisfies 1NF, it is **automatically in 2NF** by mathematical definition because partial dependencies cannot exist.
- **Trap:** Believing normalization always improves application speed.  
  *Correction:* Normalization speeds up `INSERT`/`UPDATE` operations and reduces storage, but often slows down complex `SELECT` queries due to heavy `JOIN` overhead.
