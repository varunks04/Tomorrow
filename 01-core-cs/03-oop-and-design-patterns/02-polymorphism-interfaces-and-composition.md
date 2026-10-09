# Polymorphism, Interfaces, Abstract Classes & Composition

> **Domain:** Core Computer Science  
> **Sub-Domain:** Software Engineering & Architecture  
> **Interview Importance:** Very High / Senior Design & Architecture Interviews  

---

## 1. Topic & Definitions

- **Polymorphism:** The ability of different types to be treated through a uniform interface, with behavior dynamically customized based on the underlying concrete object type.
- **Dynamic Method Dispatch:** The runtime mechanism that determines which implementation of an overridden method to execute when invoked through a base class reference or interface pointer.
- **Interface:** A pure contract specifying **WHAT** methods an implementing class must provide, without containing state (instance fields) or implementation details.
- **Abstract Class:** A partially implemented blueprint that cannot be instantiated on its own; can contain both abstract method declarations (to be implemented by subclasses) and concrete methods with shared state.
- **Composition:** A software design technique where a complex object is constructed by assembling one or more distinct objects as internal fields (**"HAS-A" relationship**), rather than inheriting behavior (**"IS-A" relationship**).

---

## 2. Compile-Time vs. Runtime Polymorphism

```text
+───────────────────────────────────+   +───────────────────────────────────+
|     COMPILE-TIME POLYMORPHISM     |   |       RUNTIME POLYMORPHISM        |
|     (Static / Early Binding)      |   |       (Dynamic / Late Binding)    |
+───────────────────────────────────+   +───────────────────────────────────+
| • Method Overloading              |   | • Method Overriding               |
| • Resolved at compilation time    |   | • Resolved at runtime via vtable  |
| • Same method name, different     |   | • Identical method signature      |
|   parameter types or counts       |   |   between parent and child class  |
| • Faster (No runtime pointer hop) |   | • Flexible, supports plugin arch  |
+───────────────────────────────────+   +───────────────────────────────────+
```

### The Virtual Method Table (vtable) Internal Mechanism

When a class declares a `virtual` method:
1. The compiler generates a **vtable** for that class—an array of function pointers pointing to the class's method implementations.
2. Every instantiated object of that class contains an invisible pointer (**`vptr`**) at the very start of its memory layout pointing to its class's vtable.
3. When `baseRef->render()` is called:
   - The CPU reads `vptr` from the object memory (`[object + 0]`).
   - Looks up the method index in the vtable.
   - Jumps to the resolved function address.

```text
Object in RAM:                        vtable in Memory:
+-------------------+                 +-----------------------------------------+
| vptr (8 bytes)    | ──────────────► | [Index 0]: &DerivedClass::authenticate()|
|-------------------|                 | [Index 1]: &DerivedClass::render()      |
| instance_variable |                 +-----------------------------------------+
+-------------------+
```

---

## 3. Key Differences: Abstract Classes vs. Interfaces

| Dimension | Interface | Abstract Class |
| :--- | :--- | :--- |
| **Relationship** | Defines capability / contract (**"CAN-DO"**). | Defines identity / base blueprint (**"IS-A"**). |
| **Multiple Inheritance** | A class can implement **multiple interfaces**. | A class can inherit from **only one abstract class** (in Java/C#). |
| **State / Variables** | Cannot maintain instance state; only static constants. | Can have full instance fields, constructors, and private state. |
| **Method Implementation**| Historically zero implementation (modern languages allow default methods). | Can provide fully implemented concrete methods alongside abstract ones. |
| **Speed** | Marginally slower interface table (itable) lookup. | Fast vtable offset lookup. |
| **Design Intent** | Decoupling completely unrelated classes (e.g. `Serializable`, `Loggable`). | Sharing common core code and state among tightly related subclasses. |

---

## 4. Composition vs. Inheritance ("Favor Composition Over Inheritance")

A legendary software design principle. Why should engineers avoid deep inheritance trees?

### The Fragile Base Class Problem
In deep inheritance hierarchies:
- Subclasses depend on internal implementation details of parent classes.
- Any change to a base class cascades unpredictably into subclasses, often breaking invariants or security checks.
- Inheritance exposes the child to all base class vulnerabilities.

```text
❌ Rigid Inheritance ("IS-A"):
User ──► AuthenticatedUser ──► AdminUser ──► CloudAdminUser (Deep, brittle coupling)

✅ Flexible Composition ("HAS-A"):
class User {
    private AuthenticationHandler authHandler; // Pluggable component
    private RolePermissions permissions;        // Pluggable component
    private AuditLogger logger;                 // Pluggable component
}
```

---

## 5. Access Modifiers & Static Members

### Access Modifier Scope Matrix

| Modifier | Current Class | Same Package / Assembly | Subclass (Derived) | World (External) |
| :--- | :---: | :---: | :---: | :---: |
| **`private`** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **`protected`** | ✅ Yes | ✅ Yes (Java) | ✅ Yes | ❌ No |
| **`package / internal`** | ✅ Yes | ✅ Yes | ❌ No | ❌ No |
| **`public`** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |

### Static vs. Instance Members
- **Instance Member:** Associated with a specific object; allocated on the Heap during instantiation.
- **Static Member:** Belongs to the **Class itself**, allocated once in the global/metaspace segment for the entire lifetime of the process.
  - ⚠️ **Concurrency Security Hazard:** Static mutable variables are shared across **all concurrent threads and requests**. In web servers handling concurrent user requests, storing user session state or security contexts in static fields causes cross-account session leaks (User A sees User B's dashboard).

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Polymorphism is realized either at compile time through method overloading, or at runtime through method overriding backed by virtual method tables (vtables) that dispatch calls dynamically. Interfaces define pure behavioral contracts allowing multiple inheritance of types, while Abstract Classes provide shared base implementations and state. The golden architectural rule is to favor composition over inheritance: assembling objects using 'HAS-A' relationships decouples components and avoids the fragile base class problem. From a security perspective, over-relying on mutable static members creates catastrophic thread-safety vulnerabilities that cause cross-tenant session leaks."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing multiple inheritance of *type* with multiple inheritance of *implementation*.  
  *Correction:* Java and C# support multiple inheritance of type (implementing many interfaces) while strictly forbidding multiple inheritance of state/classes to eliminate the Diamond Problem.
- **Trap:** Forgetting the security danger of `protected` access in Java.  
  *Correction:* In Java, `protected` members are accessible not only by subclasses, but also by any class located in the **same package**, which can allow untrusted code sharing the namespace to access sensitive fields.
