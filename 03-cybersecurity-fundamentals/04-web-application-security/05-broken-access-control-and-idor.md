# Broken Access Control, IDOR & Privilege Escalation

> **Domain:** Cybersecurity Fundamentals & Application Security (AppSec)  
> **Sub-Domain:** Authorization Architecture & Access Control  
> **Interview Importance:** Critical / Ranked #1 on the OWASP Top 10  

---

## 1. Topic & Definitions

- **Access Control (Authorization):** The security mechanism that enforces policies determining whether an authenticated user is permitted to perform a requested action or access a requested object.
- **Broken Access Control (OWASP A01):** The failure of an application to properly enforce access limitations, allowing unauthorized users to view, edit, or delete data belonging to other accounts, or access administrative functions.
- **Insecure Direct Object Reference (IDOR):** A specific type of broken access control vulnerability occurring when an application exposes a direct reference to an internal database object (e.g. integer primary key, filename) in an API endpoint, URL parameter, or request body, without verifying whether the user has permission to access that specific object.

---

## 2. Horizontal vs. Vertical Privilege Escalation

```mermaid
graph TD
    BAC[Broken Access Control] --> Horizontal[1. Horizontal Privilege Escalation<br/>User A -> User B of SAME role level<br/>(e.g. Employee accessing peer's payroll)]
    BAC --> Vertical[2. Vertical Privilege Escalation<br/>Standard User -> Administrator<br/>(e.g. User promoting self to admin)]
    BAC --> BOLA[3. BOLA in APIs<br/>Broken Object Level Authorization<br/>(Mass API object scraping)]
```

### 1. Horizontal Privilege Escalation (Peer-to-Peer Access)
- **Scenario:** Alice and Bob are standard customers of an online banking app.
- Alice views her invoice: `GET /api/v1/invoices?id=10052`.
- Alice changes the URL to: `GET /api/v1/invoices?id=10053` (Bob's invoice).
- **The Bug:** If the backend executes `SELECT * FROM invoices WHERE id = 10053` without checking `AND customer_id = current_session_user`, Alice successfully reads Bob's financial invoice.

### 2. Vertical Privilege Escalation (Privilege Elevation)
- **Scenario:** A standard user accesses administrative functionality or elevates their role:
  - *Direct URL Browsing:* Navigating to `https://site.com/admin/delete_user` directly without admin rights.
  - *Parameter Tampering / Mass Assignment:* Submitting `{"role": "admin"}` in a profile update API request where the backend blindly assigns JSON properties to the user entity.

---

## 3. The Myth of UUIDs: Why GUIDs Do NOT Stop IDOR

```text
The Flawed Thought Process:
"I replaced sequential IDs (id=101) with random UUIDs (id=d9f8a7b6-c5e4-4a3b-8c2d-1e2f3a4b5c6d).
Now an attacker cannot guess the next ID. Our app is safe from IDOR!"
```

### Why UUIDs are Security through Obscurity:
1. **IDs are Constantly Leaked:** UUIDs leak in public comments, shared links, chat messages, referral tokens, and browser history.
2. **The Root Flaw is Authorization, NOT Guessability:**
   If Alice discovers Bob's UUID (e.g. from a shared project), and the server still permits Alice to read `GET /api/project/d9f8a7b6...`, **the application is still 100% vulnerable to IDOR**.
3. **The Rule:** An unguessable ID makes scanning harder, but **it is NOT an access control mechanism**.

---

## 4. Root Cause & Secure Architecture Pattern

```text
❌ VULNERABLE CONTROLLER LOGIC:
app.get("/api/order/:orderId", (req, res) => {
    // FATAL FLAW: Looks up order solely by ID from URL! Zero ownership check!
    const order = db.query("SELECT * FROM orders WHERE id = ?", [req.params.orderId]);
    return res.json(order);
});

✅ SECURE CONTROLLER LOGIC:
app.get("/api/order/:orderId", authenticateUser, (req, res) => {
    // SECURE: Enforces that order must belong to the authenticated session user!
    const currentUserId = req.session.userId;
    const order = db.query(
        "SELECT * FROM orders WHERE id = ? AND user_id = ?",
        [req.params.orderId, currentUserId]
    );

    if (!order) {
        // Return 404 Not Found to prevent ID enumeration!
        return res.status(404).json({ error: "Order not found" });
    }
    return res.json(order);
});
```

---

## 5. Architectural Defenses Against Broken Access Control

```text
+─────────────────────────────────────────────────────────────────────────────+
|                     ACCESS CONTROL DEFENSE PRINCIPLES                       |
+─────────────────────────────────────────────────────────────────────────────+
| 1. PRINCIPLE OF COMPLETE MEDIATION:                                         |
|    • Every single API endpoint, controller, and microservice MUST check     |
|      authorization on EVERY request. Never rely on client-side state.       |
|─────────────────────────────────────────────────────────────────────────────|
| 2. SERVER-SIDE RECORD OWNERSHIP BINDING:                                    |
|    • Bind queries directly to authenticated session claims:                 |
|      WHERE object_id = ? AND tenant_id = session.tenant_id                  |
|─────────────────────────────────────────────────────────────────────────────|
| 3. ADOPT ABAC (Attribute-Based Access Control):                             |
|    • Evaluate context beyond simple roles (e.g., Is user in department?     |
|      Is access during business hours? Does user own the record?).           |
|─────────────────────────────────────────────────────────────────────────────|
| 4. PREVENT ID ENUMERATION (Return 404 instead of 403):                      |
|    • If an unauthorized user probes an object ID, return HTTP 404           |
|      (Not Found) instead of 403 (Forbidden), preventing the attacker from   |
|      confirming whether the object ID actually exists.                      |
+─────────────────────────────────────────────────────────────────────────────+
```

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Broken Access Control currently ranks as the #1 threat on the OWASP Top 10. It manifests primarily as Insecure Direct Object References (IDOR), where endpoints accept direct object IDs from user input without verifying that the requesting user owns that resource. This drives horizontal escalation between peer accounts and vertical escalation where standard users execute administrative functions. A critical architectural principle is that replacing sequential IDs with UUIDs is merely security through obscurity; true defense requires strict server-side authorization checks on every request (Complete Mediation) by querying records scoped to the authenticated session's user ID. Applications should enforce centralized ABAC policies and return HTTP 404 responses for unauthorized object lookups to eliminate object enumeration."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Believing UUIDs fix IDOR.  
  *Correction:* State clearly that UUIDs only solve *guessability*, NOT authorization. If access control logic is missing, knowing or finding the UUID still results in unauthorized data compromise.
- **Trap:** Forgetting API security (BOLA).  
  *Correction:* Highlight that in modern Single Page Applications and mobile architectures, IDOR is known as **Broken Object Level Authorization (BOLA)**, and remains the #1 vulnerability on the OWASP API Security Top 10.
