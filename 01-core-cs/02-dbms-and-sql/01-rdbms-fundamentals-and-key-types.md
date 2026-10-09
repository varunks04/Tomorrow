# RDBMS Fundamentals, Schemas & Relational Keys

> **Domain:** Core Computer Science  
> **Sub-Domain:** DBMS & SQL  
> **Interview Importance:** High / Database Architecture Fundamentals  

---

## 1. Topic & Definitions

- **DBMS (Database Management System):** Software for storing, managing, and retrieving unstructured or flat-file data (e.g., XML databases, key-value stores). Does not enforce relational constraints at the engine level.
- **RDBMS (Relational DBMS):** An advanced DBMS based on Edgar F. Codd's relational model where data is stored in structured two-dimensional **Tables (Relations)** consisting of **Rows (Tuples/Records)** and **Columns (Attributes/Fields)**, linked via mathematical relationships and enforcing ACID transactional properties.
- **Schema:** The formal structural blueprint or architecture of a database defining tables, fields, data types, constraints, and relational mappings.
- **Referential Integrity:** A relational database constraint rule ensuring that relationships between tables remain consistent. It dictates that a foreign key value in a child table must either match an existing primary key value in the referenced parent table or be explicitly `NULL`.

---

## 2. The Hierarchy of Database Keys

Keys are attribute sets used to uniquely identify records and enforce data integrity constraints:

```text
+───────────────────────────────────────────────────────────────────────────+
|                           SUPER KEY (Any unique combo)                    |
|   +───────────────────────────────────────────────────────────────────+   |
|   |                       CANDIDATE KEY (Minimal Super Key)           |   |
|   |   +───────────────────────────────────+                           |   |
|   |   |            PRIMARY KEY            |    ALTERNATE KEY          |   |
|   |   |   (Chosen Candidate Key)          |    (Unchosen Candidate    |   |
|   |   |   • Unique, NOT NULL              |     Keys)                 |   |
|   |   +───────────────────────────────────+                           |   |
|   +───────────────────────────────────────────────────────────────────+   |
+───────────────────────────────────────────────────────────────────────────+
```

### Key Types Defined with Concrete Examples

| Key Type | Formal Definition | Constraints & Rules | Example Scenario |
| :--- | :--- | :--- | :--- |
| **Super Key** | Any set of one or more attributes that uniquely identifies a row in a table. | May contain redundant, non-minimal attributes. | `{StudentID, Email, FullName}` or `{StudentID, PhoneNumber}`. |
| **Candidate Key** | A **minimal** Super Key containing no redundant attributes. | Must be unique; can be more than one candidate key in a table. | `StudentID` is a candidate key; `Email` is also a candidate key. |
| **Primary Key (PK)** | The single Candidate Key officially selected by the database architect to uniquely identify table rows. | **Strictly UNIQUE and NOT NULL**. Only one Primary Key permitted per table. | `StudentID` chosen as table Primary Key. |
| **Alternate Key** | Candidate keys that were **not** chosen as the primary key. | Unique constraint applied in table schema. | `Email` acts as Alternate Key. |
| **Composite Key** | A Primary Key that consists of **two or more attributes combined** to ensure uniqueness. | Used when no single column alone guarantees uniqueness. | In an `Enrollment` table: `{StudentID, CourseID}` together form a composite PK. |
| **Foreign Key (FK)** | A field (or collection of fields) in a child table that points directly to the Primary Key of a parent table. | Enforces **Referential Integrity**. Can contain duplicates and can be `NULL` (unless declared `NOT NULL`). | `CourseID` in `Enrollment` table referencing `CourseID` in `Courses` table. |

---

## 3. Referential Integrity & Cascading Actions

When a referenced row in a parent table is updated or deleted, the foreign key constraint enforces relational integrity:

```sql
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY,
    CustomerID INT,
    OrderDate DATE,
    CONSTRAINT fk_customer
        FOREIGN KEY (CustomerID) 
        REFERENCES Customers(CustomerID)
        ON DELETE CASCADE       -- If Customer deleted, auto-delete their orders
        ON UPDATE CASCADE       -- If CustomerID changes, propagate to orders
);
```

### Common Referential Action Rules
1. **`ON DELETE RESTRICT / NO ACTION` (Default):** Rejects and aborts the deletion if child records reference that parent ID.
2. **`ON DELETE CASCADE`:** Automatically deletes all corresponding child rows when the parent row is deleted.
3. **`ON DELETE SET NULL`:** Deletes the parent row and sets the foreign key column in all child records to `NULL`.

---

## 4. Key Differences: DBMS vs. RDBMS vs. NoSQL

| Dimension | DBMS | RDBMS | NoSQL |
| :--- | :--- | :--- | :--- |
| **Data Structure** | Flat files, hierarchical or network trees. | 2D Relational Tables (Rows & Columns). | Documents (JSON), Key-Value, Graphs, Wide-column. |
| **Relationships** | Not supported at engine level. | Core foundation via Foreign Keys. | Implicitly embedded or denormalized. |
| **Normalization** | None. | Heavily normalized (3NF/BCNF) to prevent redundancy. | Denormalized for high read performance. |
| **Integrity Constraints**| Application logic must handle integrity. | Strictly enforced by database engine. | Soft integrity handled at application layer. |
| **Scalability** | Single server. | **Vertical scaling** (Scale Up: bigger CPU/RAM). | **Horizontal scaling** (Scale Out: cluster sharding). |
| **Representative Systems**| MS Access, SQLite (basic use), XML. | PostgreSQL, MySQL, Oracle DB, SQL Server. | MongoDB, Redis, Cassandra, Neo4j. |

---

## 5. Cybersecurity Relevance & Threat Vectors

### 1. Orphan Records & Inconsistent Security States
- If referential integrity is disabled (`FOREIGN_KEY_CHECKS = 0`), deleting a user record may leave "orphan" permission rows or resource access tokens in dependent tables. An attacker could register a new user that inherits the recycled identifier, gaining unauthorized access.

### 2. Information Disclosure via Database Metadata Schemas
- Attackers probing database systems use standardized schema introspection tables:
  ```sql
  -- Information Gathering query used in SQL Injection reconnaissance
  SELECT table_name, column_name FROM information_schema.columns WHERE table_schema = 'public';
  ```
- **Hardening:** Strictly limit access permissions on `information_schema` and database system catalogs so non-administrative database users cannot inspect table designs.

### 3. Composite Key Bypasses in Multi-Tenant Databases
- In multi-tenant SaaS applications, tables often use composite primary keys: `{TenantID, RecordID}`. If backend developers write queries like `SELECT * FROM invoices WHERE RecordID = 50` forgetting `AND TenantID = ?`, an attacker easily reads another tenant's private records (IDOR at database level).

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"An RDBMS organizes data into mathematically structured tables enforcing relationships and referential integrity through constraints. A table's rows are uniquely identified by a Primary Key, selected from Candidate Keys (minimal super keys). Foreign Keys establish referential integrity between child and parent tables, preventing orphan records by defining rules like CASCADE or RESTRICT. In cybersecurity, enforcing relational integrity at the schema level prevents inconsistent authorization states and broken object level access, while database schema permissions restrict attackers from enumerating tables during injection attacks."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing Candidate Key and Primary Key.  
  *Correction:* A table can have multiple Candidate Keys (e.g., both `SSN` and `EmployeeID`), but only **one** candidate key is officially anointed as the **Primary Key**. The unselected candidate keys become Alternate Keys.
- **Trap:** Believing Foreign Keys cannot contain NULLs or duplicates.  
  *Correction:* Foreign keys can contain duplicate values (one-to-many relationship) and can contain `NULL` values unless explicitly declared `NOT NULL`.
